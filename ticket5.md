# Android Error Log Analysis – Presentation Brief

Source: error-log export, 24 Sep – 27 Oct 2026 (8 devices, 2 stores). Apps involved: Support Service, Launcher, GoLauncher, Support Tool, and the Log Processor library that is embedded in every app.

---

## 1. The big picture

| Category | Share of errors | What it really is |
|---|---:|---|
| Network | 71% | Mostly old builds sending metrics to the production server. The rest are real connectivity and server failures. |
| Misleveled info | 11% | "Metrics disabled" trace lines logged as ERROR. |
| Config / File | 7% | Mostly normal "nothing saved yet" cases logged as ERROR, plus one real path bug. |
| Auth | 6% | Calls sent without the SPIFFE token, or QA devices calling production. |
| Backend | 3% | The same lifecycle failure seen from the other side, plus QA store-info 404s. |
| Device/App, Parsing, Other | 3% | Time zone parsing, signature checks, untrusted-app removal, config-load chain. |

About 1 in 6 errors is a normal event logged as ERROR. About two thirds are one issue: old builds sending metrics to production.

**Synthetic data:** the export also holds a very large block of test rows (`PaymentManager`) that is not real. It is excluded from everything above and covered in section 4.

## 2. Five root causes behind almost everything

1. **Non-production devices call production.** Metrics, store details, file downloads and platform configuration default to production hosts. A missing bootstrap.json also defaults to production.
2. **Some calls go out without the SPIFFE token.** GoLauncher and the Support Service get HTTP 401.
3. **Wrong log levels.** Normal fallbacks, disabled features and system warnings are logged at ERROR.
4. **Retry and logging amplification.** Immediate retries, long timeouts and the same failure logged by several classes turn one outage into many rows.
5. **Old builds in the field, and no app version in the log.** We cannot tell which build a device runs, so we cannot confirm fixes.

---

## 3. Issue cards

Each card: **Error / Why / Application / Root cause / Action.**

### NETWORK

#### N1 – Failed to send metric
- **Error:** "Failed to send metric: failed to connect to metrics.io.homedepot.com".
- **Why:** devices cannot reach the production metrics server and time out. The old code retries three times at once and logs every attempt.
- **Application:** all five (Support Service, Launcher, Support Tool, Log Processor, GoLauncher), because they share the Log Processor library.
- **Root cause:** devices run an old Log Processor build. The old code always uses the production metrics host, even on QA devices. Metric sending was switched off on 22 Sep (commit 04e7ccd), but the devices do not have that fix.
- **Action:** update devices to ARTIFACTORY-700.0.1 or later. When metrics come back, use the right host per environment, add a delay between retries and log a failure once.

#### N2 – Connection failures to the identity and launcher services
- **Error:** seven related failures: identity service not reachable, SSLValidateJob failed, read timed out, launcher service not reachable, time-from-ISP timeout, upstream 503 and "no internet".
- **Why:** mostly device or network problems. A few devices lost their network, some sit on home-style networks that cannot reach the launcher service, and the QA launcher service was down on 1, 2 and 10 Oct.
- **Application:** Support Service (token refresh, time lookup) and Launcher (SSL job, platform config, network interceptor).
- **Root cause:** network or server unavailability. Our code makes it louder: retries have no delay, timeouts are 30–90 seconds, and one failure is logged by about three classes.
- **Action:** ask device owners why the network was down, add a delay between retries, shorten the 90 second timeout, log each failure once at WARNING, and look into the "Base URL required" error.

### AUTH

#### A1 – GoLauncher lifecycle call returns 401
- **Error:** HTTP 401 on the lifecycle-phase call, then "parsed object is null".
- **Why:** the request has no SPIFFE token, so the backend rejects it.
- **Application:** GoLauncher.
- **Root cause:** `useSpiffeAuth()` returns false in the committed code (commit c960cff, in all 7.0.00x and 700.0.x builds). The endpoint now requires a token.
- **Action:** commit `useSpiffeAuth()` = true (already changed locally, not committed), fix the outdated comment, test on a device and release.

#### A2 – QA devices call production and get 403
- **Error:** 403 Forbidden on getFile, getStoreDetails and events.
- **Why:** test devices send requests to production, and production refuses them.
- **Application:** Launcher.
- **Root cause:** the getStoreDetails URL is hardcoded to production. The sample bootstrap.json has the production host. A missing bootstrap.json falls back to production, and getFile follows the same fallback when getStoreDetails fails.
- **Action:** add a QA getStoreDetails endpoint chosen by environment, fix the sample bootstrap.json, and never fall back silently to production on a QA device.
- **Not confirmed:** some getFile 403s on one device have no earlier getStoreDetails 403. The cadlauncherservice team needs to check.

#### A3 – SPIFFE token unavailable
- **Error:** "SPIFFE token unavailable for request" and "Token refresh failed, retrying in 5 min".
- **Why:** the device cannot reach the identity service, so there is no token, and the Launcher stops before sending any request.
- **Application:** Support Service issues the token. Launcher uses it.
- **Root cause:** a side effect of the network problem in N2, not a separate bug. Refresh retries only every 5 minutes, and every request in between logs an error.
- **Action:** fix the device network, log "token unavailable" once per outage, retry sooner at first (30 s, 1 min, 2 min, then 5 min), and skip non-critical calls while there is no token.

### BACKEND

#### B1 – QA store-info returns 404 for two stores
- **Error:** store-info call fails with 404 and logs a full stack trace each time.
- **Why:** the QA store-info server most likely has no data for these two stores. The app asks again every time because it saves the store only on success.
- **Application:** Support Service.
- **Root cause:** missing QA data (probably), plus a call with no timeout, no retry delay and a stack trace logged at ERROR. The app falls back to `en_US`.
- **Action:** ask the store-info team to add the stores to QA, remember failures and retry later, log 404 as one short warning, and add a timeout.

### CONFIG / FILE

#### C1 – "Nothing saved yet" logged as ERROR
- **Error:** messages like "External storage not mounted…", "Shared Preferences … not found" and "Unable to retrieve cached platform configuration".
- **Why:** first start, install, update or cleared data. No saved config exists, so the app uses the built-in one. This works as designed.
- **Application:** Support Service, Launcher, GoLauncher.
- **Root cause:** wrong log level and a misleading message (the code never checks whether storage is mounted).
- **Action:** change to INFO, reword to "No saved config found, using built-in config", log once, and keep ERROR for real read or parse failures.

#### C2 – Config path is missing a "/"
- **Error:** "Unable to retrieve / write configuration at /storage/emulated/0supportserviceconfig.json".
- **Why:** the path is wrong, so the file can never be read or saved.
- **Application:** Support Service.
- **Root cause:** `GenericPlatform.configurationSubfolder` returns an empty string. Zebra, Samsung and the default provider return "/Bootstrap/Config/".
- **Action:** build paths with `File(parent, name)`, give GenericPlatform a real subfolder, and log the device model to find which devices are detected as Generic.

### PARSING

#### P1 – Time zone cannot be read
- **Error:** "Exception parsing timeFromISP object: For input string: "-4"".
- **Why:** the time server returns the offset as text ("-4"), but the code parses it as a number.
- **Application:** Support Service.
- **Root cause:** parsing assumes a number. The lookup table only knows positive minutes, so "-4" (hours?) may still fail after the first fix.
- **Action:** call the time server once and read the real answer, parse text or number, convert hours to minutes if needed, and log one error instead of two.

#### P2 – GoLauncher cannot decrypt its Airwatch credentials
- **Error:** "Exception while decrypting airwatch QA/PR credentials … Concatenated string is null."
- **Why:** decryption fails, the real error is swallowed, and a null value is passed on.
- **Application:** GoLauncher.
- **Root cause:** `decryptBlowfish` catches every exception and returns null, so only the misleading downstream message is logged. Likely a wrong or old build on that one device (not confirmed).
- **Action:** log the real exception (never the values), log the build type and QA/PR, check the build on that device, and fix the null risk in `ConfigurationManager`.
- **Security note:** the encryption key is hardcoded in source and the cipher is plain Blowfish. Move the key out and use a safer mode.

### DEVICE / APP

#### D1 – "Unable to resolve package signature"
- **Error:** signature check fails for printerservice, fp_filemanager and firstphone.
- **Why:** the app is probably not installed, or its signature cannot be read. The log cannot tell which.
- **Application:** Support Service.
- **Root cause:** these packages come from a cloud or SD-card config (the built-in list is empty), and the code logs an error for any package it cannot check. On one device, init also seems to repeat many times a minute (cause unknown, possible link to O1).
- **Action:** skip packages that are not installed (INFO), log the reason, confirm which config holds these packages, and find why init repeats.

#### D2 – Apps removed as "untrusted"
- **Error:** "Untrusted package installed – <package> – Removing…", then the app is uninstalled.
- **Why:** the package is not on the whitelist the device is using.
- **Application:** Support Service.
- **Root cause:** devices use a different whitelist (cloud or SD card) from the built-in one, because apps on the built-in list were still removed. `mytoolbelt` was renamed on 11 Aug and old installs are removed. GlobalProtect, clouddpc and zebra.wfc.central were never on the list.
- **Action:** confirm the active whitelist, decide whether GlobalProtect, zebra.wfc.central and clouddpc belong on it, update the MDM app list to the new mytoolbelt name, and log at WARNING with the whitelist source.

#### D3 – Low memory logged as ERROR
- **Error:** "System is running low on memory".
- **Why:** Android tells the app memory is low. It is a system warning, not an app failure.
- **Application:** Launcher.
- **Root cause:** `onLowMemory()` logs at ERROR.
- **Action:** log at WARNING with device model, free memory and uptime.

### MISLEVELED INFO

#### M1 – "Metrics disabled" traces logged as ERROR
- **Error:** "Sending metric: …" and "Metric Point Tags: {…}".
- **Why:** metrics are switched off, but the code writes two ERROR lines before it returns. Nothing is sent.
- **Application:** Launcher (most), Log Processor, Support Service, GoLauncher, Support Tool.
- **Root cause:** logging was left at ERROR when sending was disabled (commits 91cf367, 7209fd3, 04e7ccd).
- **Action:** move to DEBUG in all code bases and remove identifiers from these lines.
- **Privacy:** the tags include user IDs, device serial and store numbers (and MAC/IP in GoLauncher), and they end up in the uploaded error log. Needs a privacy review.

### OTHER

#### O1 – Config load fails, lock held too long, init does nothing
- **Error:** "loadConfig failed … NullPointerException", "Error acquiring lock to retrieve configuration", "No service configuration available. Not executing any actions."
- **Why:** one chain. The platform config call fails, the code crashes in its own error handling, the config lock stays held, and init finds no config and skips its work.
- **Application:** Support Service.
- **Root cause:** (a) the call is made without the SPIFFE token and always to the production host. (b) `!!` operators throw NullPointerExceptions in the error path. (c) the lock is held through two slow web calls, but `getConfiguration` waits only 5 seconds.
- **Action:** send the token, use the QA host for QA stores, remove the `!!`, load the local config first and update from the cloud in the background, and return the last known config instead of null.

#### O2 – Small items
- `DisableGOLauncherJob` is scheduled but not declared in AndroidManifest.xml, so Android says "No such service". This is a real bug. Its 60 second period is also below Android's 15 minute minimum.
- "Permission to read serial number denied" is logged whenever the result is empty, so it can be wrong.
- Normal events logged as ERROR: "Loaded GoConfiguration from local resources", "device is considered out of store" and "WiFi has invalid IP 0.0.0.0".
- "Encryption key cannot be null or empty": the caller was not traced.

---

## 4. Synthetic error: PaymentManager (test data, not a real failure)

- **Error:** "Failed to connect to payment service" (tag `PaymentManager`), logged at ERROR. It makes up 98% of the export.
- **Why it is coming:** it is test data, not app behaviour. The rows were inserted in bulk into the error-log database in one short burst on 23 Sep, with fake device IDs, fake store numbers and made-up payloads.
- **Application:** none. No CAD repo contains `PaymentManager`.
- **Root cause:** a bulk test insert into the error-log database. This is my reading of the data. Nobody has confirmed it.
- **Impact:** any unfiltered dashboard or count is mostly fake errors and hides the real ones.
- **Action:** find out who inserted it, delete it after a backup, keep test data out of the production error-log table, and exclude it in all queries until then.

---

## 5. Real code bugs found

| # | Bug | App | Ticket |
|---|---|---|---|
| 1 | `GenericPlatform` returns an empty config subfolder | Support Service | C2 |
| 2 | `useSpiffeAuth()` returns false | GoLauncher | A1 |
| 3 | Platform config call has no token, `!!` crashes and a lock is held too long | Support Service | O1 |
| 4 | Time zone offset text parsed as a number | Support Service | P1 |
| 5 | `DisableGOLauncherJob` missing from the manifest | Support Service | O2 |
| 6 | `CryptoHelper` swallows the decrypt error | GoLauncher | P2 |

## 6. Decisions and asks

1. **Release:** push builds with the Log Processor fix (ARTIFACTORY-700.0.1 or later) to all devices.
2. **Environment handling:** QA devices must never call production. Use bootstrap.json as the single source of truth, with no silent fallback to production.
3. **Auth:** every backend call from every app sends the SPIFFE token. Add a check or test for it.
4. **Logging standard:** ERROR only for real failures. Add app version, device model and environment to every log row.
5. **Resilience standard:** one retry policy with backoff, shorter timeouts, one log line per failure.
6. **Test data:** clean up the synthetic `PaymentManager` rows and keep test data out of the production error-log table.
7. **Teams to contact:** cadlauncherservice (the unexplained 403s), store-info (add the two QA stores), config owner (which whitelist devices use).
8. **Privacy and security:** review user IDs and serial numbers in the error log. Remove the hardcoded Blowfish key from source.

## 7. What is not confirmed

- App versions on devices (not in the log).
- Real bootstrap.json contents on devices and who delivers the file.
- Token environment (QA or production).
- Real time-server reply (hours or minutes).
- Whether the two stores exist in QA store-info.
- Which whitelist devices use.
- Support Tool findings (its repo was not available locally).
