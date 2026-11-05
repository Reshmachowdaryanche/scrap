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
- **Error (as logged):**
  ```
  SocketMonitor | Failed to send metric: failed to connect to metrics.io.homedepot.com/165.130.2.3 (port 0) from /10.0.2.16 (port 59044) after 5000ms: isConnected failed: ENETUNREACH (Network is unreachable)
  java.net.ConnectException: failed to connect to metrics.io.homedepot.com/165.130.2.3 (port 0) ... at libcore.io.IoBridge.isConnected(IoBridge.java:341) ...

  Other variant (timeout):
  SocketMonitor | Failed to send metric: failed to connect to metrics.io.homedepot.com/2600:140f:2e00::b856:70b0 (port 0) after 5000ms
  java.net.SocketTimeoutException: failed to connect to metrics.io.homedepot.com/... at libcore.io.IoBridge.connectErrno(IoBridge.java:190) ...
  ```
- **Why:** devices cannot reach the production metrics server and time out. The old code retries three times at once and logs every attempt.
- **Application:** all five (Support Service, Launcher, Support Tool, Log Processor, GoLauncher), because they share the Log Processor library.
- **Root cause:** devices run an old Log Processor build. The old code always uses the production metrics host, even on QA devices. Metric sending was switched off on 22 Sep (commit 04e7ccd), but the devices do not have that fix.
- **Action:** update devices to ARTIFACTORY-700.0.1 or later. When metrics come back, use the right host per environment, add a delay between retries and log a failure once.

#### N2 – Connection failures to the identity and launcher services
- **Error (as logged):**
  ```
  1. Identity service not reachable
  RetryInterceptor | Request failed on attempt 1: Failed to connect to identity.service.homedepot.com/165.130.65.82:443
  SpiffeTokenRepository | Token fetch failed: StandaloneCoroutine was cancelled kotlinx.coroutines.JobCancellationException: StandaloneCoroutine was cancelled

  2. SSLValidateJob failed
  JobFlowHandlers | launcherChainedFlow SSLValidateJob has failed with message java.net.SocketTimeoutException: failed to connect to cadlauncherservice.storeapps.homedepot.ca/35.196.74.108 (port 443) from /192.168.1.15 (port 51370) after 90000ms: isConnected failed: ETIMEDOUT (Connection timed out).message

  3. Read timed out
  RetryInterceptor | Request failed on attempt 3: Read timed out

  4. Launcher service not reachable
  PlatformConfigFacade | Unable to retrieve configuration from [https://cadlauncherservice.storeapps.homedepot.ca/v1/stores/9771/getStoreDetails]. java.net.SocketTimeoutException: failed to connect to cadlauncherservice.storeapps.homedepot.ca/35.196.74.108 (port 443) from /192.168.1.15 (port 51418) after 90000ms: isConnected failed: ETIMEDOUT (Connection timed out) at libcore.io.IoBridge.isConnected(IoBridge.java:283) ...

  5. Time-from-ISP timeout
  TimeAction | Failed to get time from ISP - error: com.android.volley.TimeoutError at com.android.volley.toolbox.NetworkUtility.shouldRetryException(NetworkUtility.java:162) ...

  6. Upstream 503
  ApiErrorHandler | message: upstream connect error or disconnect/reset before headers. reset reason: connection termination, error code: 503 endpoint: https://cadlauncherservice-qa.np-storeapps.homedepot.ca/v1/getFile?storeNum9771&fileNamePlatformConfig_210.json

  7. No internet
  InterceptorNetwork | No internet connection
  ```
- **Why:** mostly device or network problems. A few devices lost their network, some sit on home-style networks that cannot reach the launcher service, and the QA launcher service was down on 1, 2 and 10 Oct.
- **Application:** Support Service (token refresh, time lookup) and Launcher (SSL job, platform config, network interceptor).
- **Root cause:** network or server unavailability. Our code makes it louder: retries have no delay, timeouts are 30–90 seconds, and one failure is logged by about three classes.
- **Action:** ask device owners why the network was down, add a delay between retries, shorten the 90 second timeout, log each failure once at WARNING, and look into the "Base URL required" error.

### AUTH

#### A1 – GoLauncher lifecycle call returns 401
- **Error (as logged):**
  ```
  RestDelegate | Web Service call error java.lang.Exception: Response code greater than 400: 401 :: at com.homedepot.sa.xp.golauncher.core.RestDelegate.execute(RestDelegate.java:101) at ...LifecyclePhaseServiceClient.retrieveFromServer(LifecyclePhaseServiceClient.java:141) ...
  LifecyclePhaseServiceClient | Error retrieving lifecycle phase from server - parsed object is null.
  ```
- **Why:** the request has no SPIFFE token, so the backend rejects it.
- **Application:** GoLauncher.
- **Root cause:** `useSpiffeAuth()` returns false in the committed code (commit c960cff, in all 7.0.00x and 700.0.x builds). The endpoint now requires a token.
- **Action:** commit `useSpiffeAuth()` = true (already changed locally, not committed), fix the outdated comment, test on a device and release.

#### A2 – QA devices call production and get 403
- **Error (as logged):**
  ```
  ApiErrorHandler | message: , error code: 403 endpoint: https://cadlauncherservice.storeapps.homedepot.ca/v1/stores/9771/getStoreDetails
  ```
- **Why:** test devices send requests to production, and production refuses them.
- **Application:** Launcher.
- **Root cause:** the getStoreDetails URL is hardcoded to production. The sample bootstrap.json has the production host. A missing bootstrap.json falls back to production, and getFile follows the same fallback when getStoreDetails fails.
- **Action:** add a QA getStoreDetails endpoint chosen by environment, fix the sample bootstrap.json, and never fall back silently to production on a QA device.
- **Not confirmed:** some getFile 403s on one device have no earlier getStoreDetails 403. The cadlauncherservice team needs to check.

#### A3 – SPIFFE token unavailable
- **Error (as logged):**
  ```
  SpiffeAuthInterceptor | SPIFFE token unavailable for request: https://cadlauncherservice.storeapps.homedepot.ca/v1/stores/9771/getTime java.lang.IllegalStateException: SPIFFE token unavailable after waiting. at ...SpiffeTokenUseCaseImpl.invoke-d1pmJ48(SpiffeTokenUseCaseImpl.kt:33) ...
  SpiffeTokenRefreshMgr | Token refresh failed, retrying in 5 min
  ```
- **Why:** the device cannot reach the identity service, so there is no token, and the Launcher stops before sending any request.
- **Application:** Support Service issues the token. Launcher uses it.
- **Root cause:** a side effect of the network problem in N2, not a separate bug. Refresh retries only every 5 minutes, and every request in between logs an error.
- **Action:** fix the device network, log "token unavailable" once per outage, retry sooner at first (30 s, 1 min, 2 min, then 5 min), and skip non-critical calls while there is no token.

### BACKEND

#### B1 – QA store-info returns 404 for stores 9771 and 7132
- **Error (as logged):**
  ```
  DeviceSessionProvider | java.io.FileNotFoundException: https://thd-store-info-service-qa.apps-np.homedepot.com/stores/9771 at com.android.okhttp.internal.huc.HttpURLConnectionImpl.getInputStream(HttpURLConnectionImpl.java:255) ...
  (Same message for stores/7132. Java raises FileNotFoundException when the server answers 404.)
  ```
- **Why:** the QA store-info server most likely has no data for stores 9771 and 7132. The app asks again every time because it saves the store only on success.
- **Application:** Support Service.
- **Root cause:** missing QA data (probably), plus a call with no timeout, no retry delay and a stack trace logged at ERROR. The app falls back to `en_US`.
- **Action:** ask the store-info team to add stores 9771 and 7132 to QA (or give valid QA test stores), remember failures and retry later, log 404 as one short warning, and add a timeout.

### CONFIG / FILE

#### C1 – "Nothing saved yet" logged as ERROR
- **Error (as logged):**
  ```
  InitConfigProvider | External storage not mounted and persistent cache not found, retrieving go init config from local resources.
  InitConfigProvider | Attempted to load configuration from Shared Preferences at key: init.configuration but not found.
  LauncherUtils | Unable to retrieve cached platform configuration from internal storage. java.io.FileNotFoundException: /data/user/0/com.homedepot.sa.xp.firstphone/files/PlatformConfig_111.json: open failed: ENOENT (No such file or directory) at libcore.io.IoBridge.open(IoBridge.java:492) ...
  ```
- **Why:** first start, install, update or cleared data. No saved config exists, so the app uses the built-in one. This works as designed.
- **Application:** Support Service, Launcher, GoLauncher.
- **Root cause:** wrong log level and a misleading message (the code never checks whether storage is mounted).
- **Action:** change to INFO, reword to "No saved config found, using built-in config", log once, and keep ERROR for real read or parse failures.

#### C2 – Config path is missing a "/"
- **Error (as logged):**
  ```
  SupportServiceConfigProvider | Unable to retrieve configuration at [/storage/emulated/0supportserviceconfig.json]. java.io.FileNotFoundException: /storage/emulated/0supportserviceconfig.json: open failed: ENOENT (No such file or directory) ...
  SupportServiceConfigProvider | Exception while attempting to write config to file at path [/storage/emulated/0supportserviceconfig.json].
  ```
- **Why:** the path is wrong, so the file can never be read or saved.
- **Application:** Support Service.
- **Root cause:** `GenericPlatform.configurationSubfolder` returns an empty string. Zebra, Samsung and the default provider return "/Bootstrap/Config/".
- **Action:** build paths with `File(parent, name)`, give GenericPlatform a real subfolder, and log the device model to find which devices are detected as Generic.

### PARSING

#### P1 – Time zone cannot be read
- **Error (as logged):**
  ```
  TimeAction | Exception parsing timeFromISP object: For input string: ""-4""
  ```
- **Why:** the time server returns the offset as text ("-4"), but the code parses it as a number.
- **Application:** Support Service.
- **Root cause:** parsing assumes a number. The lookup table only knows positive minutes, so "-4" (hours?) may still fail after the first fix.
- **Action:** call the time server once and read the real answer, parse text or number, convert hours to minutes if needed, and log one error instead of two.

#### P2 – GoLauncher cannot decrypt its Airwatch credentials
- **Error (as logged):**
  ```
  ConfigurationManager | Exception while decrypting airwatch QA credentials. java.lang.IllegalArgumentException: Concatenated string is null. Unable to parse. at com.homedepot.sa.xp.golauncher.crypto.Credentials.<init>(Credentials.java:26) at ...CryptoHelper.decrypt(CryptoHelper.java:37) at ...ConfigurationManager.applyCredentialConfig(ConfigurationManager.java:418) ...
  (Same message for PR credentials, line 444.)
  ```
- **Why:** decryption fails, the real error is swallowed, and a null value is passed on.
- **Application:** GoLauncher.
- **Root cause:** `decryptBlowfish` catches every exception and returns null, so only the misleading downstream message is logged. Likely a wrong or old build on that one device (not confirmed).
- **Action:** log the real exception (never the values), log the build type and QA/PR, check the build on that device, and fix the null risk in `ConfigurationManager`.
- **Security note:** the encryption key is hardcoded in source and the cipher is plain Blowfish. Move the key out and use a safer mode.

### DEVICE / APP

#### D1 – "Unable to resolve package signature"
- **Error (as logged):**
  ```
  ZebraPlatform | Unable to resolve package signature for com.homedepot.sa.xp.printerservice
  (Same message for com.homedepot.sa.xp.fp_filemanager and com.homedepot.sa.xp.firstphone.)
  ```
- **Why:** the app is probably not installed, or its signature cannot be read. The log cannot tell which.
- **Application:** Support Service.
- **Root cause:** these packages come from a cloud or SD-card config (the built-in list is empty), and the code logs an error for any package it cannot check. On one device, init also seems to repeat many times a minute (cause unknown, possible link to O1).
- **Action:** skip packages that are not installed (INFO), log the reason, confirm which config holds these packages, and find why init repeats.

#### D2 – Apps removed as "untrusted"
- **Error (as logged):**
  ```
  PackageInstallReceiver | Untrusted package installed - com.paloaltonetworks.globalprotect - Removing...
  ```
- **Why:** the package is not on the whitelist the device is using.
- **Application:** Support Service.
- **Root cause:** devices use a different whitelist (cloud or SD card) from the built-in one, because apps on the built-in list were still removed. `mytoolbelt` was renamed on 11 Aug and old installs are removed. GlobalProtect, clouddpc and zebra.wfc.central were never on the list.
- **Action:** confirm the active whitelist, decide whether GlobalProtect, zebra.wfc.central and clouddpc belong on it, update the MDM app list to the new mytoolbelt name, and log at WARNING with the whitelist source.

#### D3 – Low memory logged as ERROR
- **Error (as logged):**
  ```
  LauncherApplication | System is running low on memory
  ```
- **Why:** Android tells the app memory is low. It is a system warning, not an app failure.
- **Application:** Launcher.
- **Root cause:** `onLowMemory()` logs at ERROR.
- **Action:** log at WARNING with device model, free memory and uptime.

### MISLEVELED INFO

#### M1 – "Metrics disabled" traces logged as ERROR
- **Error (as logged):**
  ```
  MetricsLoggingManager | Sending metric: scanner.enabled with value: 1 from: LauncherViewModel.startScanner
  MetricsLoggingManager | Metric Point Tags: {countryCanada, buildType151, lifeCycleqa}
  ```
- **Why:** metrics are switched off, but the code writes two ERROR lines before it returns. Nothing is sent.
- **Application:** Launcher (most), Log Processor, Support Service, GoLauncher, Support Tool.
- **Root cause:** logging was left at ERROR when sending was disabled (commits 91cf367, 7209fd3, 04e7ccd).
- **Action:** move to DEBUG in all code bases and remove identifiers from these lines.
- **Privacy:** the tags include user IDs, device serial and store numbers (and MAC/IP in GoLauncher), and they end up in the uploaded error log. Needs a privacy review.

### OTHER

#### O1 – Config load fails, lock held too long, init does nothing
- **Error (as logged):**
  ```
  SupportServiceConfigProvider | Unexpected error while making HTTP call. Message: HTTP response of 401 from https://cadlauncherservice.storeapps.homedepot.ca/v1/stores/9771/platformConfiguration java.lang.Exception: HTTP response of 401 ... at ...SupportServiceConfigProvider.executeWebCall(...) ...
  SupportServiceConfigProvider | null loadConfig failed to pull supportServiceConfig.json from PCF java.lang.NullPointerException at ...SupportServiceConfigProvider.getLifeCyclePhaseFromIsp(SupportServiceConfigProvider.kt:350) at ...loadConfig...(SupportServiceConfigProvider.kt:63) ...
  SupportServiceApplication | Error acquiring lock to retrieve configuration.
  SilentFullInitActivity | No service configuration available. Not executing any actions.
  ```
- **Why:** one chain of problems. The app asks the server for its settings, the request fails, the error handling crashes, and start-up then asks for the settings at the wrong moment, gets nothing, and skips its work.
- **Application:** Support Service.
- **Root cause (in simple steps):**
  1. **The request is wrong.** The app asks the server for the store's settings without its login token (SPIFFE), and always asks the production server, even on QA devices. The server answers "401 not allowed", or the connection times out after 45 seconds.
  2. **The error handling crashes.** The code assumes the server always answers. When it gets nothing, it crashes. Then the code that logs the error crashes too, because it expects an error message and there is none. That is why the log line starts with "null". The real reason (the 401) is hidden.
  3. **The settings are locked while loading.** While the app loads its settings, it locks them. Any other part that asks for the settings waits only 5 seconds. A slow or unreachable server can keep the lock for up to 45 seconds. The waiting part gives up, logs "Error acquiring lock" and gets nothing.
  4. **Start-up skips its work.** With nothing returned, start-up logs "No service configuration available" and does not run its tasks (app whitelist, file permissions, time and others).
  5. **The lock code has its own bug.** After giving up, the waiting part still "unlocks", even though it never held the lock. Repeated timeouts then let several parts in at once, so the lock no longer protects the settings.
  - **Good to know:** the settings do load afterwards from the SD card or the built-in file. Start-up skipped its work only because it asked while the lock was held. A 401 answers quickly, so the long wait comes from timeouts, not from the 401.
- **Action:**
  1. Send the SPIFFE token on the platformConfiguration call and use the QA host for QA stores.
  2. Remove both `!!` operators. If the call fails, keep the default lifecycle and log one clear line with the HTTP code.
  3. Do not hold the lock during network calls. Load the local config first and use it right away, then refresh from the cloud in the background.
  4. In `getConfiguration`, release only when the lock was acquired. Wait longer, or return the last known config instead of null.
  5. Log "No service configuration available" once, as a warning, with the reason.

#### O2 – Small items (four separate findings)



**O2.1 – Job service missing from the manifest (real bug)**
- **Error (as logged):**
  ```
  SilentFullInitActivity | No such service ComponentInfo{com.homedepot.sa.xp.firstphone.supportservice/com.homedepot.sa.xp.firstphone.supportservice.core.jobs.DisableGOLauncherJob}
  ```
- **Why:** Android cannot start the job because the app never declared it.
- **Application:** Support Service.
- **Root cause (in simple steps):**
  1. During start-up, the app tries to schedule a background job called `DisableGOLauncherJob`. This happens only in some start-ups, which is why the log shows it only 3 times.
  2. For Android to run a job, the app must list it in its manifest file (`AndroidManifest.xml`). This job is not listed.
  3. So Android refuses to schedule it and the app logs "No such service". The job never runs.
  4. The job's task is to switch off the old GoLauncher app when the new Launcher is installed. That task does not happen through this job.
  5. Second problem: the job is set to repeat every 60 seconds, but Android allows 15 minutes at the shortest. Even after the manifest is fixed, it would not run every minute.
- **Action:** if job is required, add it to manifest . 
- **Not confirmed:** only this repo's manifests were checked, not other build versions.



---

**O2.2 – "Permission to read serial number denied" can be wrong**
- **Error (as logged):**
  ```
  RetrieveOemInfo | ERROR: Permission to read serial number denied.
  ```
- **Why:** the app could not read the device serial number, and the message blames permission without knowing that this is the reason.
- **Application:** Support Service (Zebra devices).
- **Root cause (in simple steps):**
  1. At start-up the app asks the Zebra system for the device serial number.
  2. Before asking, it requests permission from Zebra and waits up to 60 seconds. If the permission is not ready in time, it asks anyway.
  3. If the answer is empty, the app writes "Permission denied". But an empty answer has several possible reasons: permission missing, the Zebra service not available, or simply no data. The app cannot tell them apart, so the message may be wrong.
  4. When the read fails, the serial is not saved. Other apps that ask the Support Service for the serial then get "unknown".
  5. Small code issue: when the answer is empty, the app does not close it (a resource leak).
- **Most likely cause:** the permission step failed or was late on those devices. This is not proven.
- **Action:** write a separate message for each case (no permission, service not found, no data), close the answer properly, and log whether the permission step worked. Then check one affected device.

---

**O2.3 – Normal events logged as ERROR**
- **Error (as logged):**
  ```
  GoConfigurationFacade | Loaded GoConfiguration from local resources.
  DeviceInfoFacade | Unable to determine store number from network and no cached value found. Device is considered out of store.
  InterceptorNetwork | WiFi has invalid IP address: 0.0.0.0. BSSID: 00:00:00:00:00:00, Signal: -127 dBm - IP: 0.0.0.0, BSSID: 00:00:00:00:00:00, Signal: -127 dBm
  ```
- **Why:** each message describes a normal state, not a failure.
- **Application:** GoLauncher (first message) and Launcher (other two).
- **Root cause:** wrong log level.
  - "Loaded … from local resources" is the designed fallback to the built-in config (same pattern as C1).
  - "Out of store" means the device is not on a store network, which is expected away from the store.
  - "Invalid IP 0.0.0.0" means Wi-Fi is connected but has no address yet. This is a short transitional state, and it is also sent as an analytics event.
- **Action:** log the first two at INFO and the third at WARNING. Keep ERROR for real failures.

---

**O2.4 – "Encryption key cannot be null or empty" (caller now traced)**
- **Error (as logged):**
  ```
  ActionHandlerReceiver | Error: Encryption key cannot be null or empty
  ```
- **Why:** an Airwatch request to the Support Service failed because the app's encryption key was empty.
- **Application:** Support Service (`AirwatchAction`, run through `ActionHandlerReceiver`).
- **Root cause:**
  1. Until commit bc425c5 (29 Sep 2026, "MTB-6778: AW env creds cleanup"), `AirwatchAction` built its decryptor with `CryptoHelper(GuavaBase64(), BuildConfig.CRYPTO_KEY)`.
  2. The build field `CRYPTO_KEY` is filled from `secrets.properties`, and it defaults to an empty string when the property is missing.
  3. `CryptoHelper` refuses an empty key and throws. `ActionHandlerReceiver` catches every exception from an action and logs only `"Error: " + message`, so the log does not say which action or which key.
  4. The commit removed this code, so current builds no longer have the problem. The rows are from 1 and 5 Oct, so they come from devices still on an older build (before ARTIFACTORY-7.0.8). This fits the old-builds theme in section 2.
- **Action:** confirm the devices were on a build older than 7.0.8, update them, and make sure no build is made without the secrets file. In `ActionHandlerReceiver`, log the action name with the error.
- **Not confirmed:** that those builds were made with an empty key (it is the only way this error can occur in that code).

---

## 4. Synthetic error: PaymentManager (test data, not a real failure)



- **Error (as logged):**
  ```
  PaymentManager | Failed to connect to payment service
  (error field: java.net.SocketTimeoutException: Connect timed out, and six similar exceptions)
  ```
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
7. **Teams to contact:** cadlauncherservice (the unexplained 403s), store-info (add stores 9771 and 7132 to QA), config owner (which whitelist devices use).
8. **Privacy and security:** review user IDs and serial numbers in the error log. Remove the hardcoded Blowfish key from source.

## 7. What is not confirmed

- App versions on devices (not in the log).
- Real bootstrap.json contents on devices and who delivers the file.
- Token environment (QA or production).
- Real time-server reply (hours or minutes).
- Whether stores 9771 and 7132 exist in QA store-info.
- Which whitelist devices use.
- Support Tool findings (its repo was not available locally).
