# Android Error Log Analysis – All Tickets

Source: `data-1793366561685.csv` (23,277 real error rows; 24 Sep – 27 Oct 2026; 8 devices; stores 9771 and 7132). 1,262,347 synthetic `PaymentManager` rows (bulk insert on 23 Sep) are excluded.

## Index

| ID | Category | Title | Rows |
|---|---|---|---:|
| N1 | Network | Failed to send metric – devices cannot connect to metrics.io.homedepot.com | 15,323 |
| N2 | Network | SSL job failed, read timed out and other connection failures | 1,068 |
| A1 | Auth | GoLauncher lifecycle-phase API returns 401 (SPIFFE auth disabled) | 811 (+435 lifecycle null) |
| A2 | Auth | QA/Dev devices call the Production server and get 403 Forbidden | 274 |
| A3 | Auth | SPIFFE token unavailable | 104 (+~118 related) |
| B1 | Backend | QA store-info server returns 404 for stores 9771 and 7132 | 227 |
| C1 | Config/File | "Nothing saved yet" cases logged as ERROR | ~1,350 |
| C2 | Config/File | Support Service config path is missing a "/" | 163 |
| P1 | Parsing | Support Service cannot read the time zone from the time server | 193 |
| P2 | Parsing | GoLauncher cannot decrypt its Airwatch credentials on one device | 26 |
| D1 | Device/App | "Unable to resolve package signature" (and init repeating) | 131 |
| D2 | Device/App | Apps removed as "untrusted" – whitelist differs from built-in | 84 |
| D3 | Device/App | Low memory logged as an error | 18 |
| M1 | Misleveled info | "Metrics disabled" trace messages logged as ERROR | 2,485 |
| O1 | Other | Config load fails, lock held too long, init does nothing | ~72 |
| O2 | Other | Small items (manifest job, misleading messages, normal events) | ~40 |

Not ticketed: Zebra certificate permission denied (26 rows, devices 25156524202317 and 25156524203544) and the `st%STORE%` URL (23 rows, device 25160524206657).

---

# N1: Failed to Send Metric – Devices Cannot Connect to metrics.io.homedepot.com

## 1. What Is Happening?

Devices keep logging the following error:

```
Failed to send metric: failed to connect to metrics.io.homedepot.com ... after 5000ms
```

## 2. Details From the Error Log CSV

Period: September 24 – October 27, 2026

- Total rows: 15,323
- Percentage of all real errors: 65.8%
- Percentage of network errors: 92.8%
- Tag: `SocketMonitor`
- 15,298 rows (99.8%) show `SocketMonitor.kt` line 211.

### Connection failure breakdown

| Failure type | Approximate rows |
|---|---|
| Timeout with no error code | 14,059 |
| ENETUNREACH – Network unreachable | 1,133 |
| Connection refused | 120 |

### Servers tried

- IPv4: `165.130.2.3` – 8,440 rows
- IPv6 addresses – 6,171 rows

### Applications reporting the error

| Application | Rows |
|---|---|
| Support Service | 4,450 |
| Launcher | 4,241 |
| Support Tool | 3,915 |
| Log Processor | 2,480 |
| Go Launcher | 237 |

### Most affected devices

| DSN | Rows |
|---|---|
| `25147524201072` | 6,146 |
| `24198524213116` | 4,598 |
| `24198524213086` | 1,331 |

### Stores

| Store | Rows |
|---|---|
| 9771 | 9,096 |
| 7132 | 5,440 |

Errors occur in bursts, reaching up to 552 errors in one minute on a single device.

## 3. Why Is It Happening?

### Root Cause 1: Devices Are Running an Old Log Processor Library

The affected devices are running an older version of `CAD_HDPLog_Processor`.

- Metric sending was switched off in the code on September 22.
- Commit: `04e7ccd`
- Fixed versions: `ARTIFACTORY-700.0.1`, `ARTIFACTORY-700.0.2`

In the old code, `SocketMonitor.kt` line 211 opens the network connection. In the fixed code, line 211 is a different line because metric sending stops earlier at line 205.

Since 99.8% of errors point to the old line 211, the failing devices do not appear to have the fix. Errors continue until October 27.

### Root Cause 2: Old Code Sends Metrics to the Production Server

In `SocketMonitor.kt`, line 73:

```
val wavefrontHost = PROD_WAVEFRONT_HOST
```

The non-production host is never used. As a result:

- All devices attempt to connect to the production metrics server.
- QA devices cannot reach the production server.
- Connections eventually time out.

### Root Cause 3: Immediate Retries

The old code retries three times immediately.

- No delay between retries.
- Every attempt is logged as an error.
- One failed metric can generate three error rows.

### Root Cause 4: Same Code Exists in Five Applications

The same old library is included in five applications, so all five applications report the error.

## 4. What Needs to Be Done?

1. Identify the app versions installed on the failing devices (DSNs listed above).
2. Update the devices to a build containing the fix (`ARTIFACTORY-700.0.1` or later).
3. If metric sending is enabled again in the future:
   - Use the non-production host for test devices.
   - Add a delay between retries.
   - Log a failure only once.

## 5. Done When

After updating the devices, no new `Failed to send metric` errors appear in the logs.

## 6. Not Confirmed

- The exact app version installed on each device is not available in the logs.
- Which environment (QA or Production) each store belongs to.

---

# N2: SSL Job Failed, Read Timed Out and Other Connection Failures to Identity Service and Launcher Service

## 1. Summary

Total error rows: 1,068 (plus 113 DNS rows, see Error 5)
Period: September 24 – October 27, 2026
Percentage of real errors: 4.6%

Seven different errors were identified. Most failures appear to be network or server problems affecting a few devices.

The application code increases the number of logs through:

- Immediate retries.
- Long connection and read timeouts.
- The same failure being logged by multiple classes.

## 2. Error Breakdown

### Error 1: identity.service.homedepot.com Not Reachable (508 rows)

What the log says:

```
Request failed on attempt 1/2/3:
Failed to connect to identity.service.homedepot.com/165.130.65.x:443
```

Followed by `Failed to fetch new token` / `Token fetch failed`.

Details:

- 485 rows are from one device: DSN `24198524213086`, store `9771`.

| Date | Rows |
|---|---|
| October 7 | 190 |
| October 9 | 170 |
| October 13 | 40 |
| October 1 | 30 |
| October 14 | 25 |
| October 2 | 15 |
| October 15 | 15 |

The device failed to reach eight different identity service IP addresses (`165.130.65.2`, `.18`, `.34`, `.50`, `.66`, `.82`, `.98`, `.114`). When all these IPs fail, the problem is likely the device's network rather than one specific server.

Why it happens in the code:

- `SpiffeTokenRefreshManager` retries every five minutes after a failure.
- October 7 had 190 rows, approximately 38 actual failures. This matches a retry loop running for approximately three hours while the network was down.
- `RetryInterceptor.kt` retries three times without delay.
- Timeout is 30 seconds (`SpiffeTokenModule.kt`). One outage can hold a call for up to 90 seconds.
- Three classes log the same failure. One failed token fetch generates five log lines: three retries from `RetryInterceptor`, one from `SpiffeTokenUseCase`, one from `SpiffeTokenRepository`.
- Therefore, 485 rows represent approximately 97 actual failures.

Cause: Network problem on one device. The application code increases the logging noise.

### Error 2: SSLValidateJob Has Failed (133 rows)

Application: Launcher

```
launcherChainedFlow SSLValidateJob has failed with message ...
```

Logged by `JobFlowHandlers`.

| Error | Rows |
|---|---|
| SocketTimeoutException | 55 |
| ForbiddenException – HTTP 403 | 41 |
| IllegalStateException – Base URL required | 18 |
| ConnectException | 5 |
| NetworkInternetException | 5 |
| ServerException | 5 |

The 55 timeout errors: `Failed to connect to cadlauncherservice.storeapps.homedepot.ca 35.196.74.108:443 after 90000ms ETIMEDOUT`.

| Store | Rows |
|---|---|
| 9771 | 97 |
| 7132 | 37 |

| Device IP (timeout rows) | Rows |
|---|---|
| 192.168.1.15 | 25 |
| 192.168.3.139 | 13 |
| 192.168.1.4 | 5 |
| 10.0.0.4 | 5 |

These look like home or office-style networks rather than store networks.

Why it happens in the code:

- `SSLValidateJob.kt` performs a normal GET request to the store-info URL. It is not an actual certificate validation check.
- If the API call throws an exception, `JobFlowHandlers.kt` logs `SSLValidateJob has failed` (lines 77–78 and 138–139).
- The 90-second timeout comes from the launcher's HTTP client (`NetworkConfig.kt`).
- `Base URL required` means the base URL was empty when the request executed, possibly because configuration had not loaded yet. Not investigated.

Cause: Server unreachable from the affected networks. Some requests are refused with HTTP 403 (see A2). The 90-second timeout makes each failure slow.

### Error 3: Read Timed Out (107 rows)

```
Request failed on attempt 1/2/3: Read timed out
```

Followed by `Failed to fetch new token` / `Token fetch failed`.

- All 107 rows come from one device: DSN `24198524213116`, store `7132`.
- October 3: 95 rows. October 5: 12 rows.
- 105 rows are identity token calls. 2 rows are the launcher service `getStoreDetails` API (QA, store 7132).
- This represents approximately 21 failed token fetches.

Meaning: The connection was established, but the server did not respond within the 30-second read timeout. The server may have been slow or overloaded.

Why it happens in the code: `RetryInterceptor` retries three times immediately, and the same failure is logged by multiple classes.

Cause: Slow server response over two days. The application code increases the logging noise.

### Error 4: cadlauncherservice Not Reachable (74 rows by the main message; 205 rows if every row mentioning the host is counted)

```
Unable to retrieve configuration from [https://cadlauncherservice.../getStoreDetails]
SocketTimeoutException: failed to connect ... after 90000ms / 45000ms / 30000ms
```

Another message: `Error getting store info ... ConnectException`.

| Host | Rows |
|---|---|
| `cadlauncherservice-qa.np-storeapps.homedepot.ca` | 133 |
| `cadlauncherservice.storeapps.homedepot.ca` | 72 |

| Timeout | Rows |
|---|---|
| 90,000 ms | 81 |
| 45,000 ms | 33 |
| 30,000 ms | 26 |

| Device IP | Rows |
|---|---|
| 192.168.1.15 | 52 |
| 192.168.3.139 | 25 |
| 192.168.1.4 | 23 |
| 10.0.0.4 | 19 |

- Stores: 9771 (189 rows), 7132 (16 rows).
- Errors appeared on 20 different days.
- `PlatformConfigFacade`: 83 rows. `ApiErrorHandler`: 56 rows. `SupportServiceConfigProvider`: 35 rows.

Why it happens in the code: `PlatformConfigFacade.kt` catches exceptions and logs `Unable to retrieve configuration from [url]` (lines 264 and 558). The same unreachable server in Error 2 is being logged by other classes.

Cause: Server unreachable from the affected networks. Same underlying issue as Error 2.

### Error 5: Time From ISP Timed Out (61 rows)

```
Failed to get time from ISP - error: com.android.volley.TimeoutError
```

- 59 rows `TimeoutError`, 2 rows `NoConnectionError`. Tag: `TimeAction`.
- Store 9771: 46 rows. Store 7132: 15 rows. 18 different days.
- DSN `25147524201072`: 24, `25160524206657`: 13, `24198524213086`: 13.

Why it happens in the code (`TimeAction.kt`):

1. `getTime()` first retrieves a SPIFFE token.
2. Calls `callGetTimeApi()` using Volley.
3. Requests `/v1/stores/{store}/getTime` (QA or Production endpoint depending on configuration).
4. No custom timeout or retry configuration was found, so Volley defaults are used.

On error the app falls back to NTP (`getTimeTimezoneFromNTP()`). If NTP also fails, DNS errors are generated for `ntp.homedepot.com` (113 DNS rows in total, 62 of them related to this flow).

Cause: The launcher service did not respond in time. A single outage can generate three errors: SPIFFE token failure, ISP time API failure, NTP DNS failure.

### Error 6: Upstream Connect Error (56 rows)

```
upstream connect error or disconnect/reset before headers. reset reason: remote connection failure
Connection refused, error code: 503
```

Endpoint: `https://cadlauncherservice-qa.np-storeapps.homedepot.ca/v1/getFile?storeNum9771&fileName=PlatformConfig_xxx.json`

- All errors are store 9771. October 10: 37 rows. October 1: 15. October 2: 4.
- 54 rows `remote connection failure`, 2 rows `connection termination`. All HTTP 503.

Meaning: This message is generated by the server-side proxy. The QA launcher service refused the connection.

Cause: QA launcher service was unavailable on October 1, 2 and 10. Nothing needs to be fixed in the application for this server-side failure.

### Error 7: No Internet Connection (54 rows)

Tag: `InterceptorNetwork`

- Store 9771: 53 rows. Store 7132: 1 row.
- 40 rows belong to DSN `24198524213086`, the same device as Error 1.
- October 9: 18, October 1: 12, October 14: 8, October 13: 7, October 6: 2, other dates: 7.

Why it happens in the code: `infrastructureModule.kt` line 66 checks network connectivity before every request. If the device is offline it logs an error and throws `NetworkInternetException`. The same place also checks the Wi-Fi IP address (for example `0.0.0.0`).

Cause: The device was offline. The code works as designed, but logs an expected network condition as an error.

## 3. Why Is It Happening? – Summary

Most failures are network or server-related rather than application bugs.

| DSN | Observed issue |
|---|---|
| `24198524213086` | Identity service connection failures and offline messages |
| `24198524213116` | Read timeouts |

- Devices using `192.168.x.x` and `10.0.0.4` networks cannot reach `cadlauncherservice`.
- QA launcher service refused connections (HTTP 503) on October 1, 2 and 10.

Application code contributing to the problem:

- Three immediate retries without delay.
- 30-second and 90-second timeouts.
- The same failure logged by three different classes.
- Expected conditions (offline, HTTP 503) logged at Error level.

## 4. What Needs to Be Done?

1. Ask the owners of DSNs `24198524213086` and `24198524213116` why the network was unavailable between October 1 and October 15.
2. Investigate why devices are using `192.168.x.x` or `10.0.0.4` networks.
3. Ask the QA team about `cadlauncherservice-qa` downtime on October 1, 2 and 10.
4. Add a delay between retries in `RetryInterceptor`.
5. Reduce the 90-second timeout in `NetworkConfig.kt`.
6. Log each failure only once, preferably at Warning level, when the application handles it.
7. Investigate the `Base URL required` error (18 rows).

## 5. Done When

One network problem produces one log line instead of multiple repeated error logs.

## 6. Not Confirmed

- Why the affected devices lost network connectivity. The logs do not provide the reason.
- Volley's actual timeout value in `TimeAction`. No explicit configuration was found.
- Why the `Base URL required` error occurs.

---

# A1: GoLauncher Lifecycle-Phase API Returns 401 Because SPIFFE Authentication Is Disabled

## 1. What Is Happening?

GoLauncher is receiving HTTP 401 Unauthorized errors when calling the lifecycle-phase API. The issue occurs because SPIFFE authentication is disabled for GoLauncher, while the platform configuration endpoint now requires a SPIFFE authentication token.

## 2. Evidence From the Error Logs

- Total rows: 811 (61% of Auth errors).
- Related rows: 435 "Error retrieving lifecycle phase from server - parsed object is null" (tag `LifecyclePhaseServiceClient`). They spike on the same days as the 401s (6 and 9 October), and 95% come from devices that also have these 401s. They are the same failure seen from the other side.
- Affected devices: `25156524202317`, `24198524213116`.
- Most affected dates: October 6 and October 9.

These errors indicate that GoLauncher is making requests without the required SPIFFE authentication token.

## 3. Root Cause

The issue is related to `AirWatchApiService.useSpiffeAuth()`. In the committed code (Java) it returns:

```java
protected boolean useSpiffeAuth() {
    return false;
}
```

This was introduced in commit `c960cff` (7 Aug 2026, "MTB-6463: package name reverted (#22)"), which is contained in tags `ARTIFACTORY-7.0.001` and later (`7.0.001`–`7.0.005`, and `700.0.1`/`700.0.2`; the `MDM-` tags too).

Previously, SPIFFE authentication was disabled for GoLauncher. However, the `platformConfiguration` endpoint now requires a SPIFFE token.

As a result:

1. GoLauncher makes the lifecycle-phase API request.
2. SPIFFE authentication is disabled.
3. The request is sent without the required authentication token.
4. The backend returns HTTP 401 Unauthorized.
5. `RestDelegate` returns null, so `LifecyclePhaseServiceClient.retrieveFromServer` logs `parsed object is null`.

## 4. Current Implementation Status

An uncommitted code change already sets `useSpiffeAuth()` to return `true`. The comment above this method still says "Do NOT use SPIFFE" and is outdated.

## 5. What Needs to Be Done?

1. Keep `useSpiffeAuth()` returning `true`.
2. Update the outdated comment.
3. Verify that the lifecycle-phase API request includes the SPIFFE authentication token.
4. Test the change on a device.
5. Verify that the backend returns a successful response instead of HTTP 401.
6. Release the updated GoLauncher build.

## 6. Done When

- No new HTTP 401 errors occur for the GoLauncher lifecycle-phase API.
- No new `Lifecycle phase null` errors occur.
- The updated implementation is tested and released.

## 7. Key Files / References

| Item | Details |
|---|---|
| Class | `AirWatchApiService` |
| Method | `useSpiffeAuth()` |
| Required value | `true` |
| Commit | `c960cff` |
| Affected releases | `7.0.001` and later |
| API | `platformConfiguration` / lifecycle-phase endpoint |
| Affected devices | `25156524202317`, `24198524213116` |

## 8. Not Confirmed

- 5% of the lifecycle-null rows come from devices without these 401s. Their cause was not checked (possibly network).

---

# A2: QA/Dev Devices Are Calling the Production Server and Getting 403 Forbidden

## 1. What Is Happening?

Some QA/Dev devices are receiving `403 Forbidden` responses from the Launcher service. The error logs contain 274 occurrences. Approximately 93% of these API calls go to the Production server instead of the QA server.

| API | 403 Error Count |
|---|---|
| `getFile` | 120 |
| `getStoreDetails` | 94 |
| `events` | 37 |

Devices affected (their own logs say QA or dev): `25147524201072` (118 production-host 403s), `24198524213116` (59), `25147524201090` (23), `25160524206657` (14), `25156524202317` (7).

## 2. Why Is It Happening?

### Root Cause 1: No QA URL for getStoreDetails

The Launcher has no separate QA endpoint for `getStoreDetails`. The Production URL is hardcoded (`BootstrapConfiguration.java` line 21; `ConfigurationManager.java` lines 545–547). QA/Dev devices also send requests to Production.

### Root Cause 2: Sample bootstrap.json Contains Production URL

The sample `bootstrap.json` (CAD_Launcher and CAD_Launcher02) also contains the Production server address and `"isProd": true`.

### Root Cause 3: Production Defaults Are Used When Bootstrap File Is Missing

When `/sdcard/Bootstrap/bootstrap.json` is missing, the Launcher falls back to the default Production addresses (and `isProd = true`).

### Root Cause 4: getFile Falls Back to Production

When `getStoreDetails` fails, the store's lifecycle is unknown and the `getFile` request falls back to the Production template (`ConfigurationManager.java` lines 519–541). One failure leads to another.

### Root Cause 5: Production Server Rejects Test Devices

The Production server refuses requests from these test devices, giving HTTP 403.

## 3. What the Log Shows

- Test devices calling the production host: confirmed (93% of 403 URLs).
- Bootstrap file missing on some affected devices (for example `25160524206657`): confirmed.
- 47 of 120 `getFile` 403s followed a `getStoreDetails` 403 on the same device within 2 minutes: the fallback chain is partly confirmed.

## 4. What Is Not Confirmed Yet?

1. **73 `getFile` 403s have no earlier `getStoreDetails` 403.** Most are device `24198524213116`. The `cadlauncherservice` team needs to check its server logs.
2. **The actual bootstrap.json contents** on the affected devices, and who delivers the file.
3. **The login token environment.** It is not confirmed whether the token is for QA or Production (the Support Service defaults to Production if its config is missing).

## 5. What Needs to Be Done?

1. Add a QA endpoint for `getStoreDetails` and choose it by environment.
2. Fix the sample `bootstrap.json` so test devices get QA values.
3. Do not silently fall back to Production on a QA/Dev device when the bootstrap file is missing.
4. Ask the `cadlauncherservice` team why device `24198524213116` got `getFile` 403s on 3–9 Oct.
5. Verify the login token environment on QA/Dev devices.

## 6. Done When

- QA/Dev devices never call the Production server.
- The correct QA endpoints are used for `getStoreDetails`, `getFile` and other APIs.
- Missing bootstrap configuration does not cause a silent fallback to Production.
- These 403 errors stop.

---

# A3: SPIFFE Token Unavailable – Launcher Calls Fail Before Leaving the Device When the Login Token Cannot Be Fetched

## What Is Wrong

The Launcher logs errors because it does not have a login (SPIFFE) token, so its requests never go out.

About 104 rows (September 24 to October 27):

- `SPIFFE token unavailable for request ...` – 43 rows
- `Token refresh failed, retrying in 5 min` – 61 rows

Related rows: `SPIFFE authentication failed - cannot proceed without token` from `PlatformConfigFacade` – about 118 more rows.

51 of the 61 refresh failures come from one device: DSN `24198524213086`, the same device that cannot reach `identity.service.homedepot.com` (see N2, Error 1).

## Why It Happens

1. The token comes from the Support Service app. It requests a token from `identity.service.homedepot.com` and shares it with the Launcher through a Content Provider (`spiffeidprovider`).
2. If the device cannot reach the identity service (network, DNS or timeout), the token request fails.
3. The Launcher's `SpiffeAuthInterceptor.kt` (lines 21–26) has no token, so it logs an error and throws. The API request is never sent.
4. `SpiffeTokenUseCaseImpl.waitForEnvironment()` waits up to 12 × 5 seconds (about 60 seconds) for the environment configuration before giving up.
5. `SpiffeTokenRefreshManager` (Support Service) retries only every 5 minutes after a failure. The device stays without a token for a long time, and every request in between fails again.
6. Each failed request is logged as an error, so one network problem creates many error rows.

## Root Cause

It is a side effect of the network problem to the identity service, not a separate SPIFFE bug. Fixing the network issue should remove most of these rows. The code makes it louder: one outage generates many repeated error logs.

## Not Sure Yet

- Why device `24198524213086` cannot reach the identity service (device Wi-Fi, store network or firewall). The `192.168.x.x` network is the likely suspect.
- Whether the token was missing on the other devices because the Support Service app was not running or the Content Provider was not ready.

## What To Do

1. Fix network access to `identity.service.homedepot.com` for device `24198524213086` (Wi-Fi, DNS, firewall).
2. Log `token unavailable` once per outage, not once per request.
3. Retry token refresh sooner at first: 30 seconds, 1 minute, 2 minutes, then every 5 minutes. (This is a suggestion, not from the existing code.)
4. Make the Launcher queue or skip non-critical calls (such as metrics and events) while there is no token.

## Done When

- A single token outage creates one error log, not dozens.
- The device gets a SPIFFE token soon after the network returns.
- Launcher requests resume once the token is available.

---

# B1: Support Service Keeps Asking the QA Store-Info Server for Stores 9771 and 7132, Gets "Not Found" (404), and Logs a Big Error Every Time

## What Is Wrong

The Support Service asks the store-info server for details of a store. For stores 9771 and 7132 the QA server answers "not found" (404). The app then asks again and again, and writes a full error each time.

227 error rows (24 Sep to 23 Oct 2026):

- store 9771: 208 rows
- store 7132: 19 rows

Main devices: `25147524201072` (77 rows), `24198524213086` (57), no device ID (54), `25156524202317` (21).

Separate, 20 more rows from the launcher ("Unable to retrieve configuration from getStoreDetails"): network not available (7), server problem (4), 502 (1). These are network/server errors, not 404, and belong to N2 and A2.

## Why It Happens

1. `DeviceSessionProvider.kt` (`getSession`) asks the store-info server for the store's country and state, to choose the app language.
2. It picks QA or Production from `/Bootstrap/bootstrap.json`. If the file says "not production" (or has no `isProd` value; `BootstrapConfig` defaults `isProd` to false), it uses QA. If the file is missing, it uses Production.
3. The QA server says 404 for these two stores. Most likely they are not in the QA data. (NOT VERIFIED yet.)
4. The call is `URL(url).readText()` with no timeout. On 404 it throws, and the catch block logs the full stack trace as an error.
5. After a failure the store number is not saved, so the next call asks the server again and logs the same error again. That is why one store gives many rows.
6. With no store info the app uses the default language `en_US`. A Quebec store would need `fr_CA`. (From the code only; not seen in the log.)

## Not Sure Yet

- Do stores 9771 and 7132 exist in the QA store-info server? Ask the store-info team.
- What is inside `bootstrap.json` on devices `25147524201072` and `24198524213086`?
- Why 54 rows have no device ID.

## What To Do

1. Ask the store-info team to add stores 9771 and 7132 to QA (or give valid QA test store numbers).
2. After a failure, remember it and retry later with a delay, instead of calling every time.
3. Log a 404 as one short warning, not a full stack trace at error level.
4. Add a timeout (for example 10 seconds) to the call.

## Done When

One failure gives one short log line, QA stores return data, and the app does not stay on `en_US` by mistake.

---

# C1: Normal "Nothing Saved Yet" Cases Are Logged as ERROR (About 1,350 Rows) – Change Log Level and Fix Wrong Message

## What Is Wrong

When the apps start for the first time (or after an install, update or cleared data) they look for a saved config. It is not there, so they use the built-in config. This is normal and works fine, but the apps log it as ERROR.

About 1,350 of the 1,599 Config/File error rows (24 Sep to 30 Oct 2026) are this.

## What Is in the Log

1. "External storage not mounted and persistent cache not found, retrieving go init config from local resources." – 525 rows (Support Service, `InitConfigProvider`).
2. "Attempted to load configuration from Shared Preferences at key: init.configuration but not found." – 263 rows (`InitConfigProvider`).
3. "Unable to retrieve cached platform configuration from internal storage. java.io.FileNotFoundException ..." – 484 rows (Launcher, `LauncherUtils`). Files: `PlatformConfig_41`, `PlatformConfig_21`, `ShellConfiguration`, `NotificationConfig`, `DockingConfiguration` and similar.
4. Smaller "not found in Shared Preferences" rows – about 80 rows: `StoreNumberFacade` (26), `BootstrapConfigurationFacade` (18), `GoConfigurationFacade` (16), `LifecyclePhaseServiceClient` (12).

## Why It Happens

1. `InitConfigProvider.kt`, `retrieveFromPrivateStorage` (about lines 118–145): when nothing is saved it logs 2 errors (the "Shared Preferences not found" line and the "External storage not mounted" line).
2. `InitConfigProvider.kt`, `retrieveFromLocalResources` (about lines 135–139): logs "External storage not mounted..." as ERROR every time it runs, even when the built-in config loaded fine.
3. The message is wrong. Nothing in the code checks whether storage is mounted. It means "no saved copy, using the built-in one".
4. `LauncherUtils.retrieveFromPrivateStorage` (Launcher, about line 469): its own comment says a cached copy exists only if an earlier attempt with this Launcher version succeeded and data was not cleared. So a new install, an update or cleared data always logs this as an error.
5. 210 of the 484 `LauncherUtils` rows and 516 of the 1,599 rows overall have no device ID.

## Not Sure Yet

- Whether any of these rows hide a real problem (for example, a saved file that should exist but is gone). The log does not tell. We only know the built-in fallback is the designed behaviour.

## What To Do

1. Change these "not found, using default" logs from ERROR to INFO (or DEBUG).
2. Remove or correct the "External storage not mounted and persistent cache not found" text. Say: `No saved init config found, using built-in config.`
3. Log it once per start, not twice (the same message is logged in two places).
4. Keep ERROR only for real failures, for example a saved file that exists but cannot be read or parsed.

## Done When

A normal first start produces no error rows for these cases, and real failures still show as errors.

---

# C2: Support Service Config File Path Is Missing a "/" (/storage/emulated/0supportserviceconfig.json) – File Can Never Be Read or Written

## What Is Wrong

163 error rows (1–23 Oct 2026, all with no device ID):

- "Unable to retrieve configuration at [/storage/emulated/0supportserviceconfig.json] ... FileNotFoundException" – 79 rows.
- "Exception while attempting to write config to file at path [/storage/emulated/0supportserviceconfig.json]" – 84 rows.

The path has no "/" between "0" and "supportserviceconfig.json". The app can neither read nor write this file.

## Why It Happens

1. `SupportServiceConfigProvider.kt` (line 50) builds the path from three parts: `external storage + storageProvider.configurationSubfolder + "supportserviceconfig.json"`.
2. `configurationSubfolder` comes from the device platform class:
   - Zebra: `"/Bootstrap/Config/"`
   - Samsung: `"/Bootstrap/Config/"`
   - Honeywell: `Constants.HONEYWELL_CONFIG_FOLDER`
   - Default provider: `"/Bootstrap/Config/"`
   - GenericPlatform: `""` (empty string)
3. With the empty string the path becomes `/storage/emulated/0supportserviceconfig.json`. That path is not valid for the app, so read and write both fail.
4. The same empty value is used when saving the config (lines about 76 and 116) and in the other load code (line 208), so the config can never be saved on this device type.

## Likely Cause

The device is detected as a Generic platform (not Zebra, Samsung or Honeywell). All 163 rows have no device ID, which fits an unrecognised device or an early start before the ID is known. NOT CONFIRMED: which device or model this is.

## Not Sure Yet

- Which device/model is detected as Generic, and why.
- What happens to the app after the failure (it falls back to the built-in config, but we have not checked what else depends on the saved file).

## What To Do

1. Build the path safely (`File(parent, name)` instead of joining strings).
2. Give `GenericPlatform` a real `configurationSubfolder` (for example `"/Bootstrap/Config/"`), the same as the other platforms.
3. Find which devices are detected as Generic and check why (add the device model to the log).
4. Once the path is right, log a missing file as INFO, not ERROR (see C1).

## Done When

The config path always has the correct "/", and the config file can be read and saved on every device type.

---

# P1: Support Service Cannot Read the Time Zone From the Time Server (193 Errors)

## What Is Wrong

- 193 error rows (24 Sep to 27 Oct 2026), all the same: `Exception parsing timeFromISP object: For input string: ""-4""`
- The app asks the time server for the store's time zone, cannot read the answer, and does not use it.
- Main devices: `25147524201072` (53), `24198524213086` (43), `25160524206657` (42), no device ID (49).

## Why It Happens

1. The time server sends `GMTOffset` as text: `"-4"`.
2. `TimeAction.kt` line 218 expects a number and uses `Integer.parseInt(...toString())`. For text, the quote marks stay, so it fails.
3. The launcher already knows it is text (`TimeIspResponse.kt` uses a String). The Support Service code does not.
4. Second problem: the lookup table (`olsonTimezoneFromOffset`) only knows positive minutes, such as 300 (New York) or 360 (Chicago). The value `-4` looks like hours. So fixing the first problem may not be enough. This is my guess. We haven't seen the real server answer.
5. The function always returns false, and the error is logged twice.
6. Device time is probably still set by the NTP backup and the saved time zone. Not checked.

## Not Sure Yet

- The real server answer: is `GMTOffset` in hours or minutes, and is there a proper zone name?
- Whether device time zones are wrong because of this.

## What To Do

1. Call the time server once for a QA store and look at the real answer.
2. Read `GMTOffset` whether it is text or a number. Convert hours to minutes if needed.
3. Return the real result (true or false) and log one error, not two.

## Done When

The time zone is read correctly and these 193 errors stop.

---

# P2: GoLauncher Cannot Decrypt Its Built-In Airwatch Credentials on One Device, and the Log Hides the Real Reason (26 Errors)

## What Is Wrong

- 26 error rows on one device, `25160524206657` (25 Sep: 4 rows, 28 Sep: 22 rows).
- "Exception while decrypting airwatch QA credentials … Concatenated string is null. Unable to parse." (13 rows)
- The same message for PR credentials (13 rows).
- QA and PR fail together every time.

## What the Code Does

1. `ConfigurationManager.applyCredentialConfig` (lines ~406–458) picks the encrypted credential text from the build: `BuildConfig.CREDENTIALS_<buildType>_QA` / `_PR`.
   - Build types 111, 131, 141 and 151 come from `bootstrap.json`. Any other build type uses `CREDENTIALS_QA` / `CREDENTIALS_PR`.
   - The values are filled at build time from `secrets.properties` (`GoLauncher/build.gradle` lines 37–50).
2. It calls `CryptoHelper.decrypt`: Base64 decode, then Blowfish decrypt.
3. `decryptBlowfish` (lines 62–73) catches every error, prints it and returns null. Then `new Credentials(null)` throws "Concatenated string is null" (`Credentials.java` line 26). That is the only message we see. The real error is lost.
4. After a failure the cached credentials are set to null. `getAirwatchCredentials()` then returns null. I didn't trace every place that uses it.

## What the Message Tells Us

- Empty credential text would log a different message ("Configured AW credentials are null!"), so the build had non-empty text.
- Bad Base64 would throw Guava's own error, not this one.
- So Base64 worked and Blowfish failed. The text isn't valid data for this app's key. Both QA and PR fail in the same run, so the cause is common to both. This is my reading of the code.

## Why This Device Only (Not Confirmed)

- The log has no app version, so we don't know which build it ran.
- The local `secrets.properties` has empty placeholders, so the real values could not be checked.
- The same device also has other setup errors (old `st%STORE%` server address, missing bootstrap file). It may run an old or wrongly built app.

## Other Things Found (Not in the 26 Rows)

- `ConfigurationManager.java` line 361: `_cachedBuildType` is an `int` set from `bootstrapConfig == null ? null : getBuildType()`. A missing bootstrap config or build type can cause a NullPointerException.
- The encryption key is written in the source code and the cipher uses plain "Blowfish" mode. This is a security weakness, unrelated to these rows.

## What To Do

1. In `CryptoHelper.decryptBlowfish`, don't return null. Throw or log the real exception.
2. Log which build type and which credential (QA or PR) failed, never the values. Log once, not twice.
3. Find the app build on device `25160524206657`, check which `CREDENTIALS_*` value it was built with, and rebuild if it's wrong.
4. Fix the null/int problem at line 361.
5. Later: move the key out of the source code and use a safer cipher mode.

## Done When

A decrypt failure shows its real reason and build type, and device `25160524206657` gets working credentials.

---

# D1: Support Service Logs "Unable to Resolve Package Signature" for printerservice and fp_filemanager, and Init Seems to Repeat Many Times per Minute on One Device (131 Rows, Only 20 Bursts)

## What Is Wrong

131 error rows (28 Sep to 23 Oct 2026):

- "Unable to resolve package signature for com.homedepot.sa.xp.printerservice" – 65 rows
- "... com.homedepot.sa.xp.fp_filemanager" – 62 rows
- "... com.homedepot.sa.xp.firstphone" – 2 rows

The 131 rows are only 20 bursts. 19 of the 20 bursts contain both packages together.

Device `24198524213086` has 86 of the rows. Its biggest burst: 34 rows in one minute (14 Oct, 17:40). That is about 17 runs of the same loop in one minute.

## Why It Happens (From the Code)

1. At init the Support Service loops over `config.manageFilePermissionWhiteList` and calls `scopedFilePermissionEnable` for each package (`SilentFullInitActivity.kt` line 367, `SilentInitReceiver.kt` line 160; also `PackageInstallReceiver.kt` lines 64 and 75).
2. `ZebraPlatform.kt` lines 485–489: it asks for the app's signature (`FirstPhoneUtils.getCallerSignatureBase64Encoded`). If that returns null it logs an error and returns.
3. The signature call returns null when the app cannot be found or its signature cannot be read. The log cannot tell these two cases apart.
4. The built-in config (`defaultsupportserviceconfig.json`) has an empty `manageFilePermissionWhiteList`. So these two packages come from another config the device loaded (cloud or SD card).

## Not Sure Yet

- Most likely these apps are simply not installed on those devices, but this is only a guess. Nothing else in the log mentions these two packages.
- Why init runs about 17 times a minute on device `24198524213086` (restart loop? repeated trigger?). Not checked. A possible link: O1 (config load failure makes init do nothing and retry). NOT proven.
- Which config (cloud or SD card) holds the two packages.

## What To Do

1. If the package is not installed, skip it and log at INFO. Log the reason (not installed vs signature unreadable) when it is an error.
2. Find out why init repeats on device `24198524213086` on 14–15 Oct (see the other logs from that time).
3. Check the config source: should printerservice and fp_filemanager be in `manageFilePermissionWhiteList` for these devices?
4. Log once per init, not once per loop pass.

## Done When

A package that is not installed does not create an error, and init does not repeat many times a minute.

---

# D2: Support Service Removes Apps as "Untrusted" That May Be Legitimate – App Whitelist on Devices Differs From the Built-In Whitelist (84 Rows)

## What Is Wrong

84 rows (24 Sep to 27 Oct 2026): "Untrusted package installed - `<package>` - Removing..." The app is then uninstalled.

Packages removed most often:

- `com.homedepot.sa.xp.homedepotbrowser` – 22
- `com.paloaltonetworks.globalprotect` (VPN) – 10
- `com.zebra.wfc.central` – 8
- `com.homedepot.sa.xp.hdbluetoothmanager` – 8
- `ca.homedepot.mytoolbelt` – 4
- `com.google.android.apps.work.clouddpc` (Android device policy app) – 3
- Others: Google apps (Maps, YouTube, Gmail, Photos, Calendar, Dialer ...) and some Zebra/Symbol apps.

It happens on all 8 devices (27 rows have no device ID). Peak: 23 Oct (23 rows).

## What the Code Does (Verified)

`PackageInstallReceiver.kt` lines 49–55: if `isAppInstallWhiteListEnabled` is true and the package is not in `appInstallWhiteList`, it logs an error and uninstalls the app. This is a security feature. The error line is a record that it worked.

## What We Verified

1. `homedepotbrowser` and `hdbluetoothmanager` are on the built-in whitelist (`defaultsupportserviceconfig.json`, since the first commit on 17 Feb 2026), but were removed 30 times. So these devices were not using the built-in whitelist. The code loads config from cloud first, then the SD card, then the built-in file. We have not seen which one the devices used.
2. `ca.homedepot.mytoolbelt`: commit `cd64c58` (11 Aug 2026, #27) renamed `ca.homedepot` to `com.homedepot` in the whitelist. A device still installed the old package name, so it is removed. Explained by the rename.
3. `globalprotect`, `clouddpc`, `zebra.wfc.central`, `printerservice`, `fp_filemanager` were never on the built-in whitelist (not in its git history).
4. The 27 rows with no device ID include many standard Google apps (Maps, YouTube, Gmail ...). This looks like a newly set-up or reset device, where Android installs its default apps before the whitelist is ready. This is my inference, not confirmed.
5. Several removals have the same millisecond (for example 28 Sep 16:58:13: GlobalProtect and the browser). Several installs were handled together.

## Not Sure Yet

- Which whitelist the devices actually used and why it lacks `homedepotbrowser` and `hdbluetoothmanager` (stale SD-card file, cloud config, or something else).
- Whether the removal of GlobalProtect, Zebra Workstation Connect and Google's `clouddpc` was wanted. Removing the device policy app could matter.

## What To Do

1. Config owner: confirm which whitelist is active on devices and why it differs from the built-in one.
2. Decide if GlobalProtect, `com.zebra.wfc.central` and `clouddpc` should be on the whitelist; add them if yes.
3. Find the source still installing the old `ca.homedepot.mytoolbelt` package (MDM app list) and update it to `com.homedepot.mytoolbelt`.
4. Log at WARNING (or INFO with a clear "removed by policy") instead of ERROR, and log which whitelist was used (cloud / SD card / built-in).
5. Check if devices in first-time setup should skip removal until the config is loaded.

## Done When

Only truly unapproved apps are removed, the active whitelist is known, and legitimate apps are no longer removed.

---

# D3: Launcher Logs "System Is Running Low on Memory" as an Error (18 Rows)

## What Is Wrong

18 error rows from the launcher (`com.homedepot.sa.xp.firstphone`), tag `LauncherApplication`: "System is running low on memory".

## Why It Happens (From the Code)

`LauncherApplication.java` lines 162–165: `onLowMemory()` is called by Android when memory is low. The code logs `LogProcessor.e(...)` every time it is called. It is a system warning, not an app failure.

## Not Sure Yet

Which devices and which apps used the memory. The 18 rows were not checked per device.

## What To Do

1. Log at WARNING instead of ERROR.
2. Add device model, free memory and the time since app start to the log, so real memory problems can be found.
3. Check which devices send this message often.

## Done When

Low memory is a warning with useful details, not an error.

---

# M1: "Metrics Disabled" Trace Messages Are Logged as ERROR in All Apps (2,485 Rows), and Some Contain User IDs and Serial Numbers

## What Is Wrong

2,485 error rows (22 Sep to 27 Oct 2026) are not errors. They are trace messages about metrics that are switched off. Every metric call writes 2 rows:

1. "Sending metric: `<name>` with value: ... from: `<place>`" – 1,241 rows
2. "Metric Point Tags: `{...}`" – 1,244 rows

That is about 1,240 metric calls. Nothing is actually sent.

Rows by app:

- Launcher (`com.homedepot.sa.xp.firstphone`): 2,343
- Log processor library (`SocketMonitor`): 108
- Support tool (`com.homedepot.sa.xp.supporttool`): 22 (code not found locally; probably the same pattern)
- GoLauncher: 12

Top metric calls in the launcher: `LauncherViewModel.startScanner` (369), `LauncherViewModel.sendLaunchMetric` (359), `PlatformConfigFacade.logMetrics` (292), `LauncherViewModel.performPostLoginTasks` (69), `UserLogoutJob.call` (36).

Most rows are from devices `25147524201072` (1,079) and `24198524213086` (524).

## Why It Happens (From the Code)

The metrics code writes two `LogProcessor.e` (ERROR) lines, then stops before the network call:

- Launcher (CAD_Launcher and CAD_Launcher02): `MetricsLoggingManager.java` lines 110–111 log the two lines at ERROR, then `if(true) return;` (about line 115). Commit `91cf367` (10 Sep 2026) added this. Its comment says the call is off "to avoid spamming the logs" because the Wavefront endpoint is not live yet. The same commit adds the two ERROR lines, so every metric still writes 2 error rows.
- Support Service: `MetricsLoggingManager.kt` lines 72–74 (ERROR x2, then `return`), commit `7209fd3` (22 Sep 2026).
- Log processor library: `SocketMonitor.kt` lines 203–205 (ERROR x2, then `return`), commit `04e7ccd` (22 Sep 2026).
- GoLauncher: `GoRepository.kt` lines 509–513 (ERROR x2, then `return@let`).

All rows are dated 22 Sep or later, after sending was switched off.

## Identifiers in the Log

- 138 tag rows contain a user ID (for example `userMTB003` on login/logout metrics).
- 54 rows contain the device serial number and store number (SocketMonitor tags).
- GoRepository builds its tags from the device MAC and IP address (lines about 505–506). Not checked whether the 12 GoLauncher rows show them.

These go into the error log that is uploaded. Someone should confirm this is allowed.

## Not Sure Yet

- The support tool code (not found locally).
- Whether the metrics will be turned on again, and when.

## Related

Same root as N1 (metrics failing to reach the production Wavefront host). Sending is now off, but the logging of it remains.

## What To Do

1. Log these two lines at DEBUG (or remove them) in all four places, and in the support tool.
2. Do not log user IDs, serial numbers, MAC or IP in these lines.
3. When metrics are turned on again, use the correct Wavefront host for each environment and log only real failures at ERROR.

## Done When

Switched-off metrics write no ERROR rows, and no user IDs or serial numbers are in the error log.

---

# O1: Support Service Config Load Fails (No Login Token, Crash in Error Handling), Holds the Config Lock Too Long, and Then Init Does Nothing (About 72 Rows)

## What Is Wrong

About 72 error rows (24 Sep to 20 Oct 2026) are one chain of failures in the Support Service:

1. "null loadConfig failed to pull supportServiceConfig.json from PCF java.lang.NullPointerException at ...getLifeCyclePhaseFromIsp" – 32 rows
2. "Error acquiring lock to retrieve configuration." – 19 rows
3. "No service configuration available. Not executing any actions." – 18 rows (plus 3 rows "No such service ... DisableGOLauncherJob" in the same init, see O2)

Devices: `25147524201042` (12 NPE rows), `25156524203544` (8), `25160524206657` (6), `25156524202317` (3), others (3).
Busiest days: 10 Oct (8), 13 Oct (8), 28 Sep (4), 20 Oct (4).

The three kinds of rows come from the same devices on the same days.

## Why It Happens (Verified in the Code and in the Log)

1. `SupportServiceConfigProvider.getLifeCyclePhaseFromIsp` (line 341) calls the production URL `.../v1/stores/{store}/platformConfiguration` (`UrlConstants PLATFORM_CONFIG_URL_PROD`). It calls `executeWebCall(url, "GET", false)`, so `requiresSpiffeAuth` is false and no login token is sent.
2. The server refuses it. The log shows "HTTP response of 401 from `https://cadlauncherservice.storeapps.homedepot.ca/v1/stores/9771/platformConfiguration`". Connection failures (45-second timeout) also appear. `executeWebCall` returns null.
3. Line 347: `rootObject!!.getJSONObject(...)` throws a NullPointerException (no message). The catch block (line 350) does `LogProcessor.e(TAG, (e.message)!!)`. The message is null, so this throws a second NullPointerException. `loadConfig`'s outer handler then logs "null loadConfig failed ..." – the "null" at the start is the missing message. This matches the 32 rows.
4. `SupportServiceApplication.reloadConfiguration` (lines about 168–215) takes the config semaphore and only releases it after `loadConfig` finishes. `loadConfig` makes up to two web calls (platformConfiguration and the PCF config), each with a 45-second timeout.
5. `SupportServiceApplication.getConfiguration` (line 275) waits only 5 seconds for that semaphore. When it times out, it logs "Error acquiring lock to retrieve configuration." and returns null (19 rows).
6. `SilentFullInitActivity` (line 390) then logs "No service configuration available. Not executing any actions." (18 rows). Init skips its work (app whitelist, file permissions, time and other start-up tasks).

## Related Findings

- It is the same missing-token 401 as A1, now in the Support Service.
- It always uses the production host, even on QA devices (see A2).
- It may explain why init seems to repeat many times a minute on device `24198524213086` (D1). This link is NOT proven.

## Not Sure Yet

- The size of the effect: how many times init really skipped its work. The log only shows 18 "No service configuration" rows.
- Whether the platformConfiguration endpoint needs the token only on production. The 401 rows are from the production host.
- Why the SD-card or built-in config is not used as soon as the web call fails (the code does fall back, but the lock is already held for a long time).

## What To Do

1. Send the SPIFFE token on the platformConfiguration call in `getLifeCyclePhaseFromIsp` (`requiresSpiffeAuth = true`).
2. Use the QA host for QA stores, not always `PLATFORM_CONFIG_URL_PROD`.
3. Remove `!!` in `getLifeCyclePhaseFromIsp` (line 347) and in the catch (line 350). If the call fails, keep the default lifecycle ("PR" today) and log one clear line with the HTTP code.
4. Do not hold the config lock while waiting for the network. Load the local (SD card or built-in) config first and set it, then update from the cloud in the background.
5. Make `getConfiguration` wait longer or return the last known config instead of null.
6. Log "No service configuration available" once, as a warning, with the reason.

## Done When

The config load works with a token, a failed web call never crashes, init always has a config, and these rows stop.

---

# O2: Small Items in the "Other" Error Group: Normal Events Logged as ERROR, Misleading Messages, and a Job Service Missing From the Manifest (About 40 Rows)

## What Is Wrong

About 40 error rows (25 Sep to 23 Oct 2026) are small, separate items. Most are normal events logged as ERROR. One is a real bug (item 1).

1. **"No such service ComponentInfo{com.homedepot.sa.xp.firstphone.supportservice/...core.jobs.DisableGOLauncherJob}" – 3 rows** (devices `25160524206657` ×2, `24198524213086` ×1)
   - `SilentFullInitActivity.kt` (lines about 235–246) schedules `DisableGOLauncherJob` as a periodic job.
   - `DisableGOLauncherJob.kt` exists (`core/jobs`), but it is not declared in `AndroidManifest.xml` (no JobService entry in the manifest; no XML in the repo mentions it). Android reports "No such service". So the job never runs on these builds. Real bug. (Only this branch's manifest was checked, not every build flavour.)
   - Also: periodic is set to 60 seconds (line 245), but Android allows 15 minutes at the shortest, so even when declared it would not run every minute.

2. **"ERROR: Permission to read serial number denied." – 9 rows** (device `24198524213116` ×8, `25156524203544` ×1)
   - `RetrieveOemInfo.kt` line 35: the message is logged when the cursor is null or has no rows (`cursor == null || cursor.count < 1`). So the message can be wrong: an empty answer is not a permission problem. The log cannot tell the two cases apart.

3. **"Loaded GoConfiguration from local resources." – 8 rows** (devices `25156524202317` ×4, `25160524206657` ×2, `25147524201072` ×1)
   - `GoConfigurationFacade.java` line 144. A normal fallback to the built-in config, logged as ERROR. Same pattern as C1.

4. **"Unable to determine store number from network and no cached value found. Device is considered out of store." – 5 rows** (4 with no device ID)
   - `DeviceInfoFacade.java` line 81 (CAD_Launcher and CAD_Launcher02). Happens when the device is not on a store network. A normal state, not an app error.

5. **"WiFi has invalid IP address: 0.0.0.0. BSSID: ..." – 7 rows** (all with no device ID)
   - `WiFiExtensions.kt` line 65 (`WiFiValidation`, `InterceptorNetwork`). The Wi-Fi is connected but has no IP yet. Logged as ERROR and also sent as an analytics event.

6. **"Error: Encryption key cannot be null or empty" – 5 rows** (no device ID, 1 and 5 Oct)
   - Thrown by `CryptoHelper.java` line 18 (GoCryptoHelper library). The log gives no hint who called it with an empty key. The caller was not traced.

7. **Other single rows (about 3):** "startForegroundService() not allowed ... HeartBeatSupportService" (support tool, 1 row), "External storage is not mounted, unable to load configuration" (1 row). Not traced; the support tool code is not available locally.

## Not Sure Yet

- Which builds are missing `DisableGOLauncherJob` in the manifest.
- Who calls `CryptoHelper` with an empty key (item 6).
- The support tool code (not available locally).

## What To Do

1. Declare `DisableGOLauncherJob` in `AndroidManifest.xml` with `android:permission="android.permission.BIND_JOB_SERVICE"`, and set the period to 15 minutes or more.
2. `RetrieveOemInfo`: log "no data returned" and "permission denied" as two different messages.
3. Change items 3, 4 and 5 from ERROR to INFO or WARNING.
4. Item 6: log who passed the empty key, and what feature needs it.

## Done When

The job service runs, messages say what really happened, and normal events no longer appear as errors.
