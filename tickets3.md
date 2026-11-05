# Android Error Log Analysis – Presentation Brief

Source: error-log export, 24 Sep – 27 Oct 2026. 23,277 real error rows, 8 devices, stores 9771 and 7132. (1.26 million synthetic PaymentManager rows from a bulk insert on 23 Sep are excluded.)

Apps involved: Support Service, Launcher, GoLauncher, Support Tool, and the Log Processor library (embedded in every app).

---

## 1. The big picture (one slide)

| Category | Rows | Share | What it really is |
|---|---:|---:|---|
| Network | 16,504 | 71% | 15,323 are metric sends to production from old builds. The rest are real connectivity or server failures on a few devices. |
| Misleveled info | 2,485 | 11% | "Metrics disabled" trace lines logged as ERROR. Not errors. |
| Config / File | 1,599 | 7% | About 1,350 are normal "nothing saved yet" cases logged as ERROR. 163 are a real path bug. |
| Auth | 1,332 | 6% | Calls sent without the SPIFFE token, or QA devices calling the production host. |
| Backend | 682 | 3% | Same lifecycle failure seen from the other side, plus store-info 404s on QA. |
| Device/App, Parsing, Other | 675 | 3% | Time zone parsing, signature checks, untrusted-app removal, config-load chain. |

Takeaway: about 17% of rows (about 3,900) are normal events logged as ERROR. About 66% is one issue (old builds sending metrics to production).

## 2. Five root causes behind almost everything (architecture view)

1. **Non-production devices call production.** Metrics, getStoreDetails, getFile and platformConfiguration all default to production hosts. A missing or wrong bootstrap.json also defaults to production.
2. **Some calls go out without the SPIFFE token.** GoLauncher (`useSpiffeAuth()` returns false) and Support Service (`getLifeCyclePhaseFromIsp` passes `requiresSpiffeAuth=false`) get HTTP 401.
3. **Wrong log levels.** Normal fallbacks, disabled features and system warnings are logged at ERROR.
4. **Retry and logging amplification.** Immediate retries, 30–90 second timeouts and the same failure logged by 2–3 classes turn one outage into many rows.
5. **Old builds in the field, and no app version in the log.** We cannot tell which build a device runs, so fixes cannot be confirmed.

---

## 3. Issue cards

Each card: **Error / Why / Application / Root cause / Action.**

### NETWORK

#### N1 – Failed to send metric (15,323 rows, 66% of all errors)
- **Error:** "Failed to send metric: failed to connect to metrics.io.homedepot.com … after 5000ms" (`SocketMonitor`, line 211).
- **Why:** The device cannot reach the production metrics server. Timeouts (14,059), network unreachable (1,133), refused (120). Three immediate retries, each logged.
- **Application:** Support Service 4,450, Launcher 4,241, Support Tool 3,915, Log Processor 2,480, GoLauncher 237. All embed the same Log Processor library.
- **Root cause:** Devices run an old Log Processor build. Old code always uses the production metrics host (`SocketMonitor.kt` line 73), even on QA devices. Sending was switched off on 22 Sep (commit 04e7ccd), but 99.8% of rows still point to the old line 211, so the fix is not on those devices.
- **Action:** Update devices to ARTIFACTORY-700.0.1 or later. When metrics return: use the non-production host for test devices, add a delay between retries, log a failure once.
- **Not confirmed:** The exact app version on each device (not in the log).

#### N2 – Connection failures to identity and launcher services (1,068 rows + 113 DNS)
- **Error:** Seven kinds: identity.service not reachable (508), SSLValidateJob failed (133), read timed out (107), cadlauncherservice not reachable (74–205), time-from-ISP timeout (61), upstream 503 (56), no internet (54).
- **Why:** Mostly device or network problems. 485 identity rows are one device (24198524213086). 107 read-timeout rows are one device (24198524213116). Several devices sit on 192.168.x.x or 10.0.0.4 networks that cannot reach the launcher service. The QA launcher service returned 503 on 1, 2 and 10 Oct.
- **Application:** Support Service (SpiffeTokenRefreshManager, TimeAction), Launcher (SSLValidateJob, PlatformConfigFacade, InterceptorNetwork).
- **Root cause:** Network or server unavailability. Our code makes it louder: `RetryInterceptor` retries 3 times with no delay, timeouts are 30–90 seconds, and one failure is logged by 3 classes (one token failure = 5 rows).
- **Action:** Ask device owners why the network was down. Add delay between retries. Reduce the 90 second timeout. Log each failure once at WARNING. Investigate "Base URL required" (18 rows).

### AUTH

#### A1 – GoLauncher lifecycle-phase API returns 401 (811 rows + 435 "lifecycle null")
- **Error:** HTTP 401 on the platformConfiguration / lifecycle-phase call, then "parsed object is null".
- **Why:** The request has no SPIFFE token, so the backend rejects it.
- **Application:** GoLauncher (`AirWatchApiService`). Devices 25156524202317, 24198524213116. Peaks 6 and 9 Oct.
- **Root cause:** `useSpiffeAuth()` returns false in committed code (commit c960cff, 7 Aug 2026, present in tags 7.0.001–7.0.005 and 700.0.1/700.0.2). The endpoint now requires a token.
- **Action:** Commit `useSpiffeAuth()` = true (already changed locally, not committed). Fix the outdated comment. Test on a device and release.
- **Not confirmed:** 5% of lifecycle-null rows come from devices without a 401 (cause not checked).

#### A2 – QA devices call production and get 403 (274 rows)
- **Error:** 403 Forbidden on getFile (120), getStoreDetails (94), events (37). 93% of the URLs are the production host.
- **Why:** Test devices send requests to production, and production refuses them.
- **Application:** Launcher (CAD_Launcher, CAD_Launcher02) mainly. Devices 25147524201072, 24198524213116, 25147524201090, 25160524206657, 25156524202317.
- **Root cause:** The getStoreDetails URL is hardcoded to production (`BootstrapConfiguration.java` line 21). The sample bootstrap.json has the production host and `isProd: true`. If bootstrap.json is missing, defaults are production. When getStoreDetails fails, getFile falls back to the production template.
- **Action:** Add a QA getStoreDetails endpoint chosen by environment. Fix the sample bootstrap.json. Never fall back silently to production on a QA device.
- **Not confirmed:** 73 getFile 403s with no earlier getStoreDetails 403 (mostly device 24198524213116) need the cadlauncherservice team. Real bootstrap.json contents and token environment unknown.

#### A3 – SPIFFE token unavailable (104 rows + about 118 related)
- **Error:** "SPIFFE token unavailable for request" (43), "Token refresh failed, retrying in 5 min" (61).
- **Why:** The device cannot reach identity.service.homedepot.com, so there is no token. The Launcher then throws before sending any request.
- **Application:** Support Service issues the token and shares it by content provider. Launcher (`SpiffeAuthInterceptor`) consumes it. 51 of 61 failures are device 24198524213086.
- **Root cause:** A side effect of the network problem in N2, not a separate bug. Refresh retries only every 5 minutes, and every request in between logs an error.
- **Action:** Fix device network. Log "token unavailable" once per outage. Retry sooner at first (30 s, 1 min, 2 min, then 5 min). Skip non-critical calls while there is no token.

### BACKEND

#### B1 – QA store-info returns 404 for stores 9771 and 7132 (227 rows)
- **Error:** Store-info call fails with 404, full stack trace logged every time.
- **Why:** The QA store-info server most likely has no data for these two stores. The store number is saved only on success, so the app asks again each time.
- **Application:** Support Service (`DeviceSessionProvider.kt`). Devices 25147524201072, 24198524213086.
- **Root cause:** Missing QA data (probably) plus a call with no timeout and no retry delay that logs the whole stack trace at ERROR. The app falls back to `en_US`.
- **Action:** Ask the store-info team to add the stores to QA. Remember failures and retry later. Log 404 as one short warning. Add a 10 s timeout.
- **Not confirmed:** Whether the stores exist in QA, and the bootstrap.json contents on those devices.

### CONFIG / FILE

#### C1 – "Nothing saved yet" logged as ERROR (about 1,350 rows)
- **Error:** "External storage not mounted and persistent cache not found…" (525), "…Shared Preferences … not found" (263), "Unable to retrieve cached platform configuration … FileNotFoundException" (484), others (about 80).
- **Why:** First start, install, update or cleared data. No saved config exists, so the app uses the built-in one. This works as designed.
- **Application:** Support Service (`InitConfigProvider`), Launcher (`LauncherUtils`), GoLauncher and others.
- **Root cause:** Wrong log level, and a misleading message (the code never checks if storage is mounted).
- **Action:** Change to INFO. Reword to "No saved init config found, using built-in config." Log once. Keep ERROR for real read or parse failures.

#### C2 – Config path is missing a "/" (163 rows)
- **Error:** "Unable to retrieve / write configuration at /storage/emulated/0supportserviceconfig.json".
- **Why:** The path is wrong, so the file can never be read or saved.
- **Application:** Support Service (`SupportServiceConfigProvider.kt` line 50). All rows have no device ID.
- **Root cause:** `GenericPlatform.configurationSubfolder` returns an empty string. Zebra, Samsung and the default provider return "/Bootstrap/Config/".
- **Action:** Build paths with `File(parent, name)`. Give GenericPlatform a real subfolder. Log the device model to find which devices are detected as Generic.

### PARSING

#### P1 – Time zone cannot be read (193 rows)
- **Error:** "Exception parsing timeFromISP object: For input string: ""-4""".
- **Why:** The time server returns `GMTOffset` as text ("-4"), but the code parses it as a number.
- **Application:** Support Service (`TimeAction.kt` line 218). Devices 25147524201072, 24198524213086, 25160524206657.
- **Root cause:** Parsing assumes a number. The lookup table (`olsonTimezoneFromOffset`) only knows positive minutes, so "-4" (hours?) may fail even after the first fix. The function always returns false and logs twice.
- **Action:** Call the time server once and read the real answer. Parse text or number, convert hours to minutes if needed. Return the real result, log once.
- **Not confirmed:** Whether device time zones are actually wrong (NTP may be covering).

#### P2 – GoLauncher cannot decrypt its Airwatch credentials (26 rows, one device)
- **Error:** "Exception while decrypting airwatch QA/PR credentials … Concatenated string is null."
- **Why:** Blowfish decryption fails, the real error is swallowed, and a null string is passed on.
- **Application:** GoLauncher (`ConfigurationManager`, `CryptoHelper`). Device 25160524206657.
- **Root cause:** `decryptBlowfish` catches every exception and returns null, so only the misleading downstream message is logged. Likely a wrong or old build for that device (not confirmed; no app version in the log).
- **Action:** Throw or log the real exception (never the values). Log build type and QA/PR. Check the build on that device. Fix the `_cachedBuildType` null/int risk (line 361).
- **Security note:** The encryption key is hardcoded and the cipher uses plain Blowfish. Plan to move the key out of source and use a safer mode.

### DEVICE / APP

#### D1 – "Unable to resolve package signature" (131 rows, only 20 bursts)
- **Error:** Signature check fails for printerservice (65), fp_filemanager (62), firstphone (2).
- **Why:** The app is probably not installed, or its signature cannot be read. The log cannot tell which.
- **Application:** Support Service (`ZebraPlatform.kt` lines 485–489, called from SilentFullInitActivity, SilentInitReceiver, PackageInstallReceiver). Device 24198524213086 has 86 rows, 34 in one minute on 14 Oct.
- **Root cause:** The built-in config has an empty `manageFilePermissionWhiteList`, so these packages come from a cloud or SD-card config. Why init repeats about 17 times a minute is unknown (possible link to O1, not proven).
- **Action:** Skip packages that are not installed (INFO). Log the reason. Confirm which config holds these packages. Find why init repeats.

#### D2 – Apps removed as "untrusted" (84 rows)
- **Error:** "Untrusted package installed - <package> - Removing…" then uninstall. Top: homedepotbrowser (22), GlobalProtect (10), zebra.wfc.central (8), hdbluetoothmanager (8), ca.homedepot.mytoolbelt (4), clouddpc (3).
- **Why:** The package is not on the whitelist the device is using.
- **Application:** Support Service (`PackageInstallReceiver.kt` lines 49–55). All 8 devices.
- **Root cause:** homedepotbrowser and hdbluetoothmanager are on the built-in whitelist since Feb 2026, yet were removed, so devices use a different whitelist (cloud or SD card). `ca.homedepot.mytoolbelt` was renamed to `com.homedepot.mytoolbelt` on 11 Aug (commit cd64c58); old installs are removed. GlobalProtect, clouddpc and zebra.wfc.central were never on the list.
- **Action:** Confirm the active whitelist. Decide if GlobalProtect, zebra.wfc.central and clouddpc belong on it (removing clouddpc may matter). Update the MDM app list to the new mytoolbelt name. Log at WARNING and say which whitelist was used.

#### D3 – Low memory logged as ERROR (18 rows)
- **Application:** Launcher (`LauncherApplication.java` lines 162–165, `onLowMemory`).
- **Root cause:** Android system warning logged at ERROR.
- **Action:** Log at WARNING with device model, free memory and uptime.

### MISLEVELED INFO

#### M1 – "Metrics disabled" traces logged as ERROR (2,485 rows)
- **Error:** "Sending metric: … from: …" (1,241) and "Metric Point Tags: {…}" (1,244). About 1,240 metric calls, nothing sent.
- **Why:** Metrics are switched off, but the code logs two ERROR lines before returning.
- **Application:** Launcher 2,343, Log Processor 108, Support Tool 22, GoLauncher 12 (plus the Support Service code path).
- **Root cause:** Logging was left at ERROR when sending was disabled (commits 91cf367 on 10 Sep, 7209fd3 and 04e7ccd on 22 Sep).
- **Action:** Move to DEBUG in all four code bases and the Support Tool. Remove identifiers from these lines.
- **Privacy:** 138 rows contain user IDs, 54 contain serial and store numbers, and GoLauncher builds tags from MAC and IP. These land in the uploaded error log. Needs a privacy review.

### OTHER

#### O1 – Config load fails, lock held too long, init does nothing (about 72 rows)
- **Error:** (1) "null loadConfig failed … NullPointerException at getLifeCyclePhaseFromIsp" (32), (2) "Error acquiring lock to retrieve configuration" (19), (3) "No service configuration available. Not executing any actions." (18).
- **Why:** One chain. The platformConfiguration call gets 401 or times out, the code crashes in its own error handling, the config lock stays held, and init finds no config and skips its work.
- **Application:** Support Service (`SupportServiceConfigProvider`, `SupportServiceApplication`, `SilentFullInitActivity`). Devices 25147524201042, 25156524203544, 25160524206657.
- **Root cause:** (a) The call is made without the SPIFFE token and always to the production host. (b) `rootObject!!` (line 347) and `(e.message)!!` (line 350) throw NullPointerExceptions. (c) The semaphore is held through up to two 45 second web calls, but `getConfiguration` waits only 5 seconds.
- **Action:** Send the token. Use the QA host for QA stores. Remove the `!!`. Load the local config first and update from the cloud in the background. Return the last known config instead of null.
- **Not confirmed:** How many times init actually skipped its work. Link to D1 is not proven.

#### O2 – Small items (about 40 rows)
- `DisableGOLauncherJob` is scheduled but not declared in AndroidManifest.xml, so Android says "No such service". **Real bug.** It is also set to a 60 second period, but Android's minimum is 15 minutes.
- "Permission to read serial number denied" (9 rows) is logged whenever the cursor is empty, so it can be wrong (`RetrieveOemInfo.kt` line 35).
- Normal events logged as ERROR: "Loaded GoConfiguration from local resources" (8), "device is considered out of store" (5), "WiFi has invalid IP 0.0.0.0" (7).
- "Encryption key cannot be null or empty" (5 rows): caller not traced.

---

## 4. Real code bugs found (for the architecture discussion)

| # | Bug | App | Ticket |
|---|---|---|---|
| 1 | `GenericPlatform` returns an empty config subfolder, so the path has no "/" | Support Service | C2 |
| 2 | `useSpiffeAuth()` returns false | GoLauncher | A1 |
| 3 | `platformConfiguration` call sent without a token, with `!!` crashes and a lock held too long | Support Service | O1 |
| 4 | `GMTOffset` text parsed as a number | Support Service | P1 |
| 5 | `DisableGOLauncherJob` missing from the manifest | Support Service | O2 |
| 6 | `CryptoHelper` swallows the decrypt error | GoLauncher | P2 |

## 5. Decisions and asks

1. **Release:** push builds with the Log Processor fix (ARTIFACTORY-700.0.1 or later) to all devices.
2. **Environment handling:** QA devices must never call production. Decide on one source of truth (bootstrap.json) and no silent fallback to production.
3. **Auth:** all backend calls from every app send the SPIFFE token. Add a check or test for this.
4. **Logging standard:** ERROR only for real failures. Normal fallbacks and disabled features go to INFO or DEBUG. Add app version, device model and environment to every log row.
5. **Resilience standard:** one retry policy with backoff, shorter timeouts, one log line per failure.
6. **Teams to contact:** cadlauncherservice (403s on device 24198524213116, 3–9 Oct), store-info (add stores 9771 and 7132 to QA), config owner (which whitelist devices use).
7. **Privacy and security:** review user IDs and serial numbers in the error log. Remove the hardcoded Blowfish key from source.

## 6. What is not confirmed

- App versions on devices (not in the log).
- Real bootstrap.json contents on devices and who delivers the file.
- Token environment (QA or production).
- Real time-server reply (hours or minutes).
- Whether stores 9771 and 7132 exist in QA store-info.
- Which whitelist devices use.
- Support Tool findings (its repo was not available locally).

Not ticketed: Zebra certificate permission denied (26 rows) and the `st%STORE%` URL (23 rows, device 25160524206657).
