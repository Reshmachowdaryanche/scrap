# Android Error Log Analysis – Network & Service Issues

# TICKET 1: Failed to Send Metric – Devices Cannot Connect to metrics.io.homedepot.com

## 1. What Is Happening?

Devices keep logging the following error:

```
Failed to send metric: failed to connect to metrics.io.homedepot.com ... after 5000ms
```

## 2. Details From the Error Log CSV

Source: `data-1793366561685.csv`\
Period: September 24 – October 27, 2026

- Total rows: 15,323
- Percentage of all real errors: 65.8%
- Percentage of network errors: 92.8%
- Tag: `SocketMonitor`
- 15,298 rows (99.8%) show `SocketMonitor.kt` line 211.

### Connection failure breakdown

| Failure type                      | Approximate rows |
| --------------------------------- | ---------------- |
| Timeout with no error code        | 14,059           |
| ENETUNREACH – Network unreachable | 1,133            |
| Connection refused                | 120              |

### Servers tried

- IPv4: `165.130.2.3` – 8,440 rows
- IPv6 addresses – 6,171 rows

### Applications reporting the error

| Application     | Rows  |
| --------------- | ----- |
| Support Service | 4,450 |
| Launcher        | 4,241 |
| Support Tool    | 3,915 |
| Log Processor   | 2,480 |
| Go Launcher     | 237   |

### Most affected devices

| DSN              | Rows  |
| ---------------- | ----- |
| `25147524201072` | 6,146 |
| `24198524213116` | 4,598 |
| `24198524213086` | 1,331 |

### Stores

| Store | Rows  |
| ----- | ----- |
| 9771  | 9,096 |
| 7132  | 5,440 |

Errors occur in bursts, reaching up to 552 errors in one minute on a single device.

## 3. Why Is It Happening?

### Root Cause 1: Devices Are Running an Old Log Processor Library

The affected devices are running an older version of `CAD_HDPLog_Processor`.

- Metric sending was switched off in the code on September 22.
- Commit: `04e7ccd`
- Fixed versions:
  - `ARTIFACTORY-700.0.1`
  - `ARTIFACTORY-700.0.2`

In the old code:

- `SocketMonitor.kt` line 211 opens the network connection.
- In the fixed code, line 211 is a different line because metric sending stops earlier at line 205.

Since 99.8% of errors point to the old line 211, the failing devices do not appear to have the fix.

Errors continue until October 27.

### Root Cause 2: Old Code Sends Metrics to the Production Server

In `SocketMonitor.kt`, line 73:

```
val wavefrontHost = PROD_WAVEFRONT_HOST
```

The non-production host is never used.

As a result:

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

# TICKET 2: SSL Job Failed, Read Timed Out and Other Connection Failures to Identity Service and Launcher Service

## 1. Summary

Total error rows: 1,068\
Period: September 24 – October 27, 2026\
Percentage of real errors: 4.6%

Seven different errors were identified.

Most failures appear to be network or server problems affecting a few devices.

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

Followed by:

```
Failed to fetch new token
Token fetch failed
```

Details:

- 485 rows are from one device.
- DSN: `24198524213086`
- Store: `9771`

Most errors occurred on:

| Date       | Rows |
| ---------- | ---- |
| October 7  | 190  |
| October 9  | 170  |
| October 13 | 40   |
| October 1  | 30   |
| October 14 | 25   |
| October 2  | 15   |
| October 15 | 15   |

The device failed to reach eight different identity service IP addresses:

```
165.130.65.2
165.130.65.18
165.130.65.34
165.130.65.50
165.130.65.66
165.130.65.82
165.130.65.98
165.130.65.114
```

When all these IPs fail, the problem is likely the device's network rather than one specific server.

Why it happens in the code:

- `SpiffeTokenRefreshManager` retries every five minutes after a failure.
- October 7 had 190 rows, approximately 38 actual failures.
- This matches a retry loop running for approximately three hours while the network was down.
- `RetryInterceptor.kt` retries three times without delay.
- Timeout is 30 seconds (`SpiffeTokenModule.kt`).
- One outage can hold a call for up to 90 seconds.
- Three classes log the same failure.

One failed token fetch generates five log lines:

1. Three retries from `RetryInterceptor`.
2. One from `SpiffeTokenUseCase`.
3. One from `SpiffeTokenRepository`.

Therefore, 485 rows represent approximately 97 actual failures.

Cause: Network problem on one device. The application code increases the logging noise.

### Error 2: SSLValidateJob Has Failed (133 rows)

Application: Launcher

What the log says:

```
launcherChainedFlow SSLValidateJob has failed with message ...
```

Logged by `JobFlowHandlers`.

Error breakdown:

| Error                                     | Rows |
| ----------------------------------------- | ---- |
| SocketTimeoutException                    | 55   |
| ForbiddenException – HTTP 403             | 41   |
| IllegalStateException – Base URL required | 18   |
| ConnectException                          | 5    |
| NetworkInternetException                  | 5    |
| ServerException                           | 5    |

Additional details:

The 55 timeout errors indicate:

```
Failed to connect to cadlauncherservice.storeapps.homedepot.ca
35.196.74.108:443
after 90000ms
ETIMEDOUT
```

Stores:

| Store | Rows |
| ----- | ---- |
| 9771  | 97   |
| 7132  | 37   |

Device addresses found in timeout rows:

| Device IP     | Rows |
| ------------- | ---- |
| 192.168.1.15  | 25   |
| 192.168.3.139 | 13   |
| 192.168.1.4   | 5    |
| 10.0.0.4      | 5    |

These look like home or office-style networks rather than store networks.

Why it happens in the code:

- `SSLValidateJob.kt` performs a normal GET request to the store-info URL.
- It is not an actual certificate validation check.
- If the API call throws an exception, `JobFlowHandlers.kt` logs `SSLValidateJob has failed`.
- Relevant lines: 77–78 and 138–139.
- The 90-second timeout comes from the launcher's HTTP client.
- `NetworkConfig.kt` configures connect, read and write timeouts to 90 seconds.

The `Base URL required` error means the base URL was empty when the request executed, possibly because configuration had not loaded yet. This has not been investigated.

Cause:

- Server unreachable from the affected networks.
- Some requests are refused with HTTP 403.
- The 90-second timeout makes each failure slow.

### Error 3: Read Timed Out (107 rows)

What the log says:

```
Request failed on attempt 1/2/3: Read timed out
```

Followed by:

```
Failed to fetch new token
Token fetch failed
```

Details:

- All 107 rows come from one device.
- DSN: `24198524213116`
- Store: `7132`

Dates:

| Date      | Rows |
| --------- | ---- |
| October 3 | 95   |
| October 5 | 12   |

Of the 107 rows:

- 105 are related to identity token calls.
- 2 are related to the launcher service `getStoreDetails` API (QA, store 7132).

This represents approximately 21 failed token fetches.

Meaning:

The connection was established, but the server did not respond within the 30-second read timeout.

The server may have been slow or overloaded.

Why it happens in the code:

- `RetryInterceptor` retries three times immediately.
- The same slow request is called again without waiting.
- The same failure is logged by multiple classes.

Cause: Slow server response over two days. The application code increases the logging noise.

### Error 4: cadlauncherservice Not Reachable (74 rows in the first count, 205 rows including all matching host references)

What the log says:

```
Unable to retrieve configuration from [https://cadlauncherservice.../getStoreDetails]
SocketTimeoutException: failed to connect ... after 90000ms / 45000ms / 30000ms
```

Another message:

```
Error getting store info ... ConnectException
```

Details:

Hosts:

| Host                                              | Rows |
| ------------------------------------------------- | ---- |
| `cadlauncherservice-qa.np-storeapps.homedepot.ca` | 133  |
| `cadlauncherservice.storeapps.homedepot.ca`       | 72   |

Timeout breakdown:

| Timeout   | Rows |
| --------- | ---- |
| 90,000 ms | 81   |
| 45,000 ms | 33   |
| 30,000 ms | 26   |

Device addresses:

| Device IP     | Rows |
| ------------- | ---- |
| 192.168.1.15  | 52   |
| 192.168.3.139 | 25   |
| 192.168.1.4   | 23   |
| 10.0.0.4      | 19   |

Stores:

| Store | Rows |
| ----- | ---- |
| 9771  | 189  |
| 7132  | 16   |

Other observations:

- Errors appeared on 20 different days.
- `PlatformConfigFacade`: 83 rows.
- `ApiErrorHandler`: 56 rows.
- `SupportServiceConfigProvider`: 35 rows.

Why it happens in the code:

`PlatformConfigFacade.kt` catches exceptions and logs:

```
Unable to retrieve configuration from [url]
```

Relevant lines: 264 and 558.

The same unreachable server identified in Error 2 is being logged by other classes.

Cause: Server unreachable from the affected networks. This appears to be the same underlying issue as Error 2.

### Error 5: Time From ISP Timed Out (61 rows)

What the log says:

```
Failed to get time from ISP - error: com.android.volley.TimeoutError
```

- 59 rows: `TimeoutError`
- 2 rows: `NoConnectionError`

Details:

- Tag: `TimeAction`
- Store 9771: 46 rows.
- Store 7132: 15 rows.
- Errors appeared across 18 different days.

DSN breakdown:

| DSN              | Rows |
| ---------------- | ---- |
| `25147524201072` | 24   |
| `25160524206657` | 13   |
| `24198524213086` | 13   |

Why it happens in the code:

`TimeAction.kt` implements the following flow:

1. `getTime()` first retrieves a SPIFFE token.
2. Calls `callGetTimeApi()` using Volley.
3. Makes a request to `/v1/stores/{store}/getTime`.
4. Uses the QA or Production endpoint depending on configuration.

No custom timeout or retry configuration was found in the Support Service code. Therefore, Volley defaults are used.

When an error occurs, the application falls back to NTP:

```
getTimeTimezoneFromNTP()
```

If NTP also fails, additional DNS errors are generated for:

```
ntp.homedepot.com
```

There are 62 related DNS errors.

Cause: The launcher service did not respond in time. A single outage can generate three errors:

1. SPIFFE token failure.
2. ISP time API failure.
3. NTP DNS failure.

### Error 6: Upstream Connect Error (56 rows)

What the log says:

```
upstream connect error or disconnect/reset before headers.
reset reason: remote connection failure
```

Another message:

```
Connection refused, error code: 503
```

Endpoint:

```
https://cadlauncherservice-qa.np-storeapps.homedepot.ca/v1/getFile?storeNum9771&fileName=PlatformConfig_xxx.json
```

Details:

\- All errors belong to store 9771.

Date breakdown:

| Date       | Rows |
| ---------- | ---- |
| October 10 | 37   |
| October 1  | 15   |
| October 2  | 4    |

Other observations:

- 54 rows: `remote connection failure`.
- 2 rows: `connection termination`.
- All errors return HTTP 503.

Meaning:

This message is generated by the server-side proxy.

The QA launcher service refused the connection, and the application received an HTTP 503 response.

Why it happens in the code:

`ApiErrorHandler.kt` reports the server response and logs it as an error.

Cause: QA launcher service was unavailable on October 1, 2 and 10.

Nothing needs to be fixed in the application for this specific server-side failure.

### Error 7: No Internet Connection (54 rows)

What the log says:

```
No internet connection
```

Tag:

```
InterceptorNetwork
```

Details:

- Store 9771: 53 rows.
- Store 7132: 1 row.
- 40 rows belong to DSN `24198524213086`, the same device as Error 1.

Date breakdown:

| Date        | Rows |
| ----------- | ---- |
| October 9   | 18   |
| October 1   | 12   |
| October 14  | 8    |
| October 13  | 7    |
| October 6   | 2    |
| Other dates | 7    |

Why it happens in the code:

In the Launcher application:

`infrastructureModule.kt`, line 66, checks network connectivity before every request.

If the device is offline:

1. It logs an error.
2. Throws `NetworkInternetException`.

The same location also checks the Wi-Fi IP address, for example:

```
0.0.0.0
```

Cause: The device was offline.

The code works as designed, but it logs an expected network condition as an error.

## 3. Why Is It Happening? – Summary

Most failures are network or server-related rather than application bugs.

### Device-specific issues

| DSN              | Observed issue                                            |
| ---------------- | --------------------------------------------------------- |
| `24198524213086` | Identity service connection failures and offline messages |
| `24198524213116` | Read timeouts                                             |

### Network-related issues

- Devices using `192.168.x.x` and `10.0.0.4` networks cannot reach `cadlauncherservice`.
- QA launcher service refused connections (HTTP 503) on October 1, 2 and 10.

### Application code contributing to the problem

- Three immediate retries without delay.
- 30-second and 90-second timeouts.
- The same failure logged by three different classes.
- Expected conditions, such as offline network and HTTP 503 responses, logged at Error level.

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





# TICKET 4: QA/Dev Devices Are Calling the Production Server and Getting 403 Forbidden

## 1. What Is Happening?

Some QA/Dev devices are receiving `403 Forbidden` responses from the Launcher service.

The error logs contain 274 occurrences of 403 Forbidden errors.

Approximately 93% of these API calls are going to the Production server instead of the QA server.

### API failure breakdown

| API               | 403 Error Count |
| ----------------- | --------------- |
| `getFile`         | 120             |
| `getStoreDetails` | 94              |
| `events`          | 37              |

## 2. Why Is It Happening?

### Root Cause 1: No QA URL for getStoreDetails

The Launcher does not have a separate QA endpoint configured for `getStoreDetails`.

Instead, the Production URL is hardcoded directly in the code.

As a result, QA/Dev devices also send requests to the Production server.

### Root Cause 2: Sample bootstrap.json Contains Production URL

The sample `bootstrap.json` file also contains the Production server address.

This can cause test devices to use the Production endpoint instead of QA.

### Root Cause 3: Production Defaults Are Used When Bootstrap File Is Missing

When the bootstrap configuration file is missing on the device, the Launcher falls back to the default Production server addresses.

Consequently, QA/Dev devices may unintentionally communicate with Production.

### Root Cause 4: getFile Falls Back to Production

When the `getStoreDetails` request fails, the subsequent `getFile` request also falls back to the Production server.

Therefore, one initial configuration or API failure can lead to additional Production API failures.

### Root Cause 5: Production Server Rejects Test Devices

The Production server refuses requests from these test devices, resulting in HTTP 403 Forbidden responses.

## 3. What Is Not Confirmed Yet?

The following points require further investigation:

1\. getFile 403 errors without earlier getStoreDetails failures

- 73 `getFile` 403 errors do not have a preceding `getStoreDetails` 403.
- Most are associated with device `24198524213116`.
- The `cadlauncherservice` team needs to investigate its server logs to identify the reason.

2\. Actual bootstrap.json contents

- The configuration currently present on the affected devices has not been verified.
- We need to confirm whether those devices contain Production or QA URLs.

3\. Login token environment

- It is not yet confirmed whether the login token belongs to the QA or Production environment.
- This needs to be verified as part of the investigation.

## 4. What Needs to Be Done?

1\. Add a QA endpoint for getStoreDetails

- Introduce a separate QA URL instead of using the hardcoded Production URL.
- Ensure the correct endpoint is selected based on the environment.

2\. Fix the sample bootstrap.json

- Update the sample configuration to contain the appropriate QA endpoint for test environments.

3\. Prevent silent Production fallback

- If the bootstrap configuration file is missing on a QA/Dev device, do not silently fall back to Production defaults.
- Handle missing configuration explicitly.

4\. Investigate device 24198524213116

- Contact the `cadlauncherservice` team.
- Request server-side logs for the 73 `getFile` 403 failures.
- Identify why these requests are rejected without an earlier `getStoreDetails` 403.

5\. Verify the login token environment

- Confirm that QA/Dev devices use the correct environment-specific authentication token.

## 5. Done When

- QA/Dev devices never make API calls to the Production server.
- The correct QA endpoints are used for `getStoreDetails`, `getFile`, and other applicable APIs.
- Missing bootstrap configuration does not cause a silent fallback to Production.
- The 403 Forbidden errors related to incorrect environment routing stop.

## 6. Key Findings

| Area                     | Finding                                                     |
| ------------------------ | ----------------------------------------------------------- |
| Total errors             | 274                                                         |
| Production traffic       | 93% of failing calls                                        |
| Main issue               | QA/Dev devices calling Production endpoints                 |
| `getFile`                | 120 failures                                                |
| `getStoreDetails`        | 94 failures                                                 |
| `events`                 | 37 failures                                                 |
| Configuration issue      | Production URL hardcoded in Launcher                        |
| Bootstrap issue          | Sample `bootstrap.json` contains Production URL             |
| Fallback issue           | Missing bootstrap configuration uses Production defaults    |
| Additional investigation | 73 `getFile` failures without earlier `getStoreDetails` 403 |
| Affected device          | `24198524213116`                                            |

Summary: QA/Dev devices are unintentionally communicating with Production because of hardcoded Production URLs, incorrect bootstrap configuration, and Production fallback behavior. The fix is to configure environment-specific endpoints, prevent silent Production fallback, and investigate the remaining 403 failures with the backend team.




# TICKET 3: GoLauncher Lifecycle-Phase API Returns 401 Because SPIFFE Authentication Is Disabled

## 1. What Is Happening?

GoLauncher is receiving HTTP 401 Unauthorized errors when calling the lifecycle-phase API.

The issue occurs because SPIFFE authentication is disabled for GoLauncher, while the platform configuration endpoint now requires a SPIFFE authentication token.

## 2. Evidence From the Error Logs

- Total rows: 811
- Percentage of Auth errors: 61%
- Affected devices:
  - `25156524202317`
  - `24198524213116`
- Most affected dates: October 6 and October 9.

These errors indicate that GoLauncher is making requests without the required SPIFFE authentication token.

## 3. Root Cause

The issue is related to the following method:

```
AirWatchApiService.useSpiffeAuth()
```

In the committed code, this method returns:

```
override fun useSpiffeAuth(): Boolean {
    return false
}
```

This change was introduced in:

- Commit: `c960cff`
- Released tags:
  - `7.0.001`
  - `7.0.002`
  - `7.0.003`
  - `7.0.004`

Previously, SPIFFE authentication was disabled for GoLauncher.

However, the `platformConfiguration` endpoint now requires a SPIFFE token.

As a result:

1. GoLauncher makes the lifecycle-phase API request.
2. SPIFFE authentication is disabled.
3. The request is sent without the required authentication token.
4. The backend returns HTTP 401 Unauthorized.
5. The lifecycle-phase retrieval fails.
6. GoLauncher subsequently logs `Lifecycle phase null`.

## 4. Current Implementation Status

An uncommitted code change already updates the authentication configuration to enable SPIFFE authentication.

The current change sets:

```
override fun useSpiffeAuth(): Boolean {
    return true
}
```

However, the comment above this method still says:

```
// Do NOT use SPIFFE
```

This comment is outdated and does not reflect the current implementation.

## 5. What Needs to Be Done?

1. Keep `useSpiffeAuth()` returning `true`.
2. Update the outdated comment to indicate that SPIFFE authentication is required.
3. Verify that the lifecycle-phase API request includes the SPIFFE authentication token.
4. Test the change on a device.
5. Verify that the backend returns a successful response instead of HTTP 401.
6. Release the updated GoLauncher build.

## 6. Expected Result

After enabling SPIFFE authentication:

- GoLauncher should attach the SPIFFE token to the lifecycle-phase API request.
- The backend should accept the authenticated request.
- The lifecycle-phase API should return the expected response.
- The `Lifecycle phase null` errors should stop.

## 7. Done When

- No new HTTP 401 errors occur for the GoLauncher lifecycle-phase API.
- No new `Lifecycle phase null` errors occur.
- The updated implementation is tested and released.

## 8. Key Files / References

| Item              | Details                                            |
| ----------------- | -------------------------------------------------- |
| Class / Interface | `AirWatchApiService`                               |
| Method            | `useSpiffeAuth()`                                  |
| Required value    | `true`                                             |
| Commit            | `c960cff`                                          |
| Affected releases | `7.0.001` – `7.0.004`                              |
| API               | `platformConfiguration` / lifecycle-phase endpoint |
| Affected devices  | `25156524202317`, `24198524213116`                 |

Summary: GoLauncher has SPIFFE authentication disabled in the committed implementation, but the lifecycle-phase API now requires it. Enabling `useSpiffeAuth()` and releasing the updated build should resolve the 401 errors and the related `Lifecycle phase null` errors.


# TICKET 5: SPIFFE Token Unavailable – Launcher Calls Fail Before Leaving the Device When the Login Token Cannot Be Fetched

## WHAT IS WRONG

The Launcher logs errors because it does not have a login (SPIFFE) token, so its requests never go out.

Approximately 104 rows in the logs (September 23 to October 27):

- `SPIFFE token unavailable for request ...` – 43 rows
- `Token refresh failed, retrying in 5 min` – 61 rows

Related rows:

\- `SPIFFE authentication failed - cannot proceed without token` from `PlatformConfigFacade` – approximately 118 additional rows.

Most affected device:

- 51 of the 61 refresh failures come from one device: DSN `24198524213086`.
- This is the same device that cannot reach `identity.service.homedepot.com` (see the network ticket about identity service / timeouts).

## WHY IT HAPPENS

1\. The token comes from the Support Service app. It requests a token from `identity.service.homedepot.com` and shares it with Launcher through a Content Provider (`SpiffeIdProvider`).

2\. If the device cannot reach the identity service due to network issues, DNS failures, or timeouts, the token request fails.

3\. The Launcher's `SpiffeAuthInterceptor.kt` (lines 21–26) has no token, so it logs an error and throws an exception. The actual API request is never sent.

4\. `SpiffeTokenUseCaseImpl.waitForEnvironment()` waits up to 12 × 5 seconds (approximately 60 seconds) for the environment configuration before giving up.

5\. `SpiffeTokenRefreshManager` (Support Service) retries only every 5 minutes after a failure. As a result, the device remains without a token for a long time, and every request in between fails again.

6\. Each failed request is logged as an error, so one network problem creates many error rows.

## SO WHAT IS THE ROOT CAUSE?

It is a side effect of the network problem affecting the identity service, not necessarily a separate SPIFFE authentication bug.

Fixing the network connectivity issue should remove most of these rows.

The code only makes the issue more visible: one network outage generates many repeated error logs.

## NOT SURE YET

- Why device `24198524213086` cannot reach the identity service. Is it device Wi-Fi, store network, or firewall? The `192.168.x.x` network issue is the likely suspect.
- Whether the token was missing because the Support Service app was not running or the Content Provider was not ready on the other devices.

## WHAT TO DO

1\. Fix network access to `identity.service.homedepot.com` for device `24198524213086`. Check Wi-Fi, DNS, and firewall restrictions.

2\. Log `token unavailable` once per outage, not once per request.

3\. Retry token refresh sooner initially, using progressive retry intervals:

- First retry: 30 seconds.
- Second retry: 1 minute.
- Third retry: 2 minutes.
- Subsequent retries: 5 minutes.

4\. Make the Launcher queue or skip non-critical calls (such as metrics/events) while no token is available.

## DONE WHEN

- A single token outage creates one error log, not dozens.
- The device gets a SPIFFE token again soon after network connectivity returns.
- Launcher requests resume once the token becomes available.






# TITLE: Support Service keeps asking the QA store-info server for stores 9771 and 7132, gets "not found" (404), and logs a big error every time

## WHAT IS WRONG

The Support Service app asks the store-info server for details of a store. For stores 9771 and 7132 the QA server answers "not found" (404).

The app then asks again and again, and writes a full error each time.

227 error rows in the log (24 Sep to 23 Oct 2026):

- store 9771: 208 rows
- store 7132: 19 rows

Main devices:

- 25147524201072 (77 rows)
- 24198524213086 (57)
- no device ID (54)
- 25156524202317 (21)

Separate, 20 more rows from the launcher "Unable to retrieve configuration from getStoreDetails":

- network not available (7)
- server problem (4)
- 502 (1)

These are network/server errors, not 404, and belong to the network/403 tickets.

## WHY IT HAPPENS

1\. `DeviceSessionProvider.kt` (`getSession`) asks the store-info server for the store's country and state, to choose the app language.

2\. It picks the QA or production server from `/Bootstrap/bootstrap.json`.

- If the file says "not production" (or has no `isProd` value), it uses the QA server.
- If the file is missing, it uses Production.

3\. The QA server says 404 for these two stores. Most likely they are not in the QA data. (NOT VERIFIED yet.)

4\. The call is `URL(url).readText()` with no timeout. On 404 it throws, and the catch block logs the full stack trace as an error.

5\. After a failure, the store number is not saved, so the next call asks the server again and logs the same error again. That is why one store gives many rows.

6\. With no store info, the app uses the default language `en_US`. A Quebec store would need `fr_CA`. (From the code only; not seen in the log.)

## NOT SURE YET

- Do stores 9771 and 7132 exist in the QA store-info server? Ask the store-info team.
- What is inside `bootstrap.json` on devices 25147524201072 and 24198524213086?
- Why 54 rows have no device ID.

## WHAT TO DO

1. Ask the store-info team to add stores 9771 and 7132 to QA (or give valid QA test store numbers).
2. After a failure, remember it and retry later with a delay, instead of calling every time.
3. Log a 404 as one short warning, not a full stack trace at error level.
4. Add a timeout (for example 10 seconds) to the call.

## DONE WHEN

One failure gives one short log line, QA stores return data, and the app does not stay on `en_US` by mistake.


\===== TICKET 1 =====

# TITLE: Normal "nothing saved yet" cases are logged as ERROR (about 1,350 rows) - change log level and fix wrong message

## WHAT IS WRONG

When the apps start for the first time (or after an install, update or cleared data) they look for a saved config. It is not there, so they use the built-in config. This is normal and works fine, but the apps log it as ERROR.

About 1,350 of the 1,599 Config/File error rows (24 Sep to 30 Oct 2026) are this.

## WHAT IS IN THE LOG

1. "External storage not mounted and persistent cache not found, retrieving go init config from local resources." – 525 rows (Support Service, `InitConfigProvider`).
2. "Attempted to load configuration from Shared Preferences at key: init.configuration but not found." – 263 rows (`InitConfigProvider`).
3. "Unable to retrieve cached platform configuration from internal storage. java.io.FileNotFoundException ..." – 484 rows (Launcher, `LauncherUtils`).

   Files: `PlatformConfig_41`, `PlatformConfig_21`, `ShellConfiguration`, `NotificationConfig`, `DockingConfiguration` and similar.
4. Smaller "not found in Shared Preferences" rows – about 80 rows:
   - `StoreNumberFacade`: 26
   - `BootstrapConfigurationFacade`: 18
   - `GoConfigurationFacade`: 16
   - `LifecyclePhaseServiceClient`: 12

## WHY IT HAPPENS

1. `InitConfigProvider.kt`, `retrieveFromPrivateStorage` (about lines 118–145): when nothing is saved it logs 2 errors (the "Shared Preferences not found" line and the "External storage not mounted" line).
2. `InitConfigProvider.kt`, `retrieveFromLocalResources` (about lines 135–139): logs "External storage not mounted..." as ERROR EVERY time it runs, even when the built-in config loaded fine.
3. The message is WRONG. Nothing in the code checks whether storage is mounted. It really means "no saved copy, using the built-in one".
4. `LauncherUtils.retrieveFromPrivateStorage` (Launcher, about line 469): its own comment says a cached copy exists only if an earlier attempt with this Launcher version succeeded and data was not cleared. So a new install, an update or cleared data always logs this as an error. This is the normal first run.
5. 210 of the 484 `LauncherUtils` rows and 516 of the 1,599 rows overall have no device ID.

## NOT SURE YET

\- Whether any of these rows hide a real problem (for example, a saved file that should exist but is gone). The log does not tell. We only know the built-in fallback is the designed behaviour.

## WHAT TO DO

1. Change these "not found, using default" logs from ERROR to INFO (or DEBUG).
2. Remove or correct the "External storage not mounted and persistent cache not found" text. Say:
   ```
   No saved init config found, using built-in config.
   ```
3. Log it once per start, not twice (the same message is logged in two places).
4. Keep ERROR only for real failures, for example a saved file that exists but cannot be read or parsed.

## DONE WHEN

A normal first start produces no error rows for these cases, and real failures still show as errors.

\===== TICKET 2 =====

# TITLE: Support Service config file path is missing a "/" (/storage/emulated/0supportserviceconfig.json) - file can never be read or written

## WHAT IS WRONG

163 error rows (1–23 Oct 2026, all with no device ID):

- "Unable to retrieve configuration at [/storage/emulated/0supportserviceconfig.json] ... FileNotFoundException" – 79 rows.
- "Exception while attempting to write config to file at path [/storage/emulated/0supportserviceconfig.json]" – 84 rows.

The path has no "/" between "0" and "supportserviceconfig.json". The app can neither read nor write this file.

## WHY IT HAPPENS

1. `SupportServiceConfigProvider.kt` (line 50) builds the path from three parts:
   ```
   external storage + storageProvider.configurationSubfolder + "supportserviceconfig.json"
   ```
2. `configurationSubfolder` comes from the device platform class:
   - Zebra: `"/Bootstrap/Config/"`
   - Samsung: `"/Bootstrap/Config/"`
   - Honeywell: `Constants.HONEYWELL_CONFIG_FOLDER`
   - Default provider: `"/Bootstrap/Config/"`
   - GenericPlatform: `""` (EMPTY STRING)
3. With the empty string, the path becomes:
   ```
   /storage/emulated/0supportserviceconfig.json
   ```
   That path is not valid for the app, so read and write both fail.
4. The same empty value is used when saving the config (lines about 76 and 116) and in the other load code (line 208), so the config can never be saved on this device type.

## LIKELY CAUSE

The device is detected as a Generic platform (not Zebra, Samsung or Honeywell).

All 163 rows have no device ID, which fits an unrecognised device or early start before the ID is known.

NOT CONFIRMED: Which device or model this is.

## NOT SURE YET

- Which device/model is detected as Generic.
- Why the platform is not recognised there.
- What happens to the app after the failure (it falls back to the built-in config, but we have not checked what else depends on the saved file).

## WHAT TO DO

1. Build the path safely (use `File(parent, name)` instead of joining strings), so a missing "/" cannot happen.
2. Give `GenericPlatform` a real `configurationSubfolder` (for example `"/Bootstrap/Config/"`), the same as the other platforms.
3. Find which devices are detected as Generic and check why (add the device model to the log).
4. Once the path is right, log a missing file as INFO, not ERROR (see Ticket 1).

## DONE WHEN

The config path always has the correct "/", and the config file can be read and saved on every device type.

## TICKET 1

Title: Support Service cannot read the time zone from the time server (193 errors)

What is wrong

- 193 error rows (24 Sep to 27 Oct 2026), all the same: `Exception parsing timeFromISP object: For input string: ""-4""`
- The app asks the time server for the store's time zone, cannot read the answer, and does not use it.
- Main devices: `25147524201072` (53), `24198524213086` (43), `25160524206657` (42), no device ID (49).

Why it happens

1. The time server sends `GMTOffset` as text: `"-4"`.
2. `TimeAction.kt` line 218 expects a number and uses `Integer.parseInt(...toString())`. For text, the quote marks stay, so it fails.
3. The launcher already knows it is text (`TimeIspResponse.kt` uses a String). The Support Service code does not.
4. Second problem: the lookup table (`olsonTimezoneFromOffset`) only knows positive minutes, such as 300 (New York) or 360 (Chicago). The value `-4` looks like hours. So fixing the first problem may not be enough. This is my guess. We haven't seen the real server answer.
5. The function always returns false, and the error is logged twice.
6. Device time is probably still set by the NTP backup and the saved time zone. Not checked.

Not sure yet

- The real server answer: is `GMTOffset` in hours or minutes, and is there a proper zone name?
- Whether device time zones are wrong because of this.

What to do

1. Call the time server once for a QA store and look at the real answer.
2. Read `GMTOffset` whether it is text or a number. Convert hours to minutes if needed.
3. Return the real result (true or false) and log one error, not two.

Done when: the time zone is read correctly and these 193 errors stop.

## TICKET 2

Title: GoLauncher cannot decrypt its built-in Airwatch credentials on one device, and the log hides the real reason (26 errors)

What is wrong

- 26 error rows on one device, `25160524206657` (25 Sep: 4 rows, 28 Sep: 22 rows).
- "Exception while decrypting airwatch QA credentials … Concatenated string is null. Unable to parse." (13 rows)
- The same message for PR credentials (13 rows).
- QA and PR fail together every time.

What the code does

1. `ConfigurationManager.applyCredentialConfig` (lines \~406-458) picks the encrypted credential text from the build: `BuildConfig.CREDENTIALS_<buildType>_QA` / `_PR`.
   - Build types 111, 131, 141 and 151 come from `bootstrap.json`. Any other build type uses `CREDENTIALS_QA` / `CREDENTIALS_PR`.
   - The values are filled at build time from `secrets.properties` (`GoLauncher/build.gradle` lines 37-50).
2. It calls `CryptoHelper.decrypt`: Base64 decode, then Blowfish decrypt.
3. `decryptBlowfish` (lines 62-73) catches every error, prints it and returns null. Then `new Credentials(null)` throws "Concatenated string is null" (`Credentials.java` line 26). That is the only message we see. The real error is lost.
4. After a failure the cached credentials are set to null. `getAirwatchCredentials()` then returns null. I didn't trace every place that uses it.

What the message tells us

- Empty credential text would log a different message ("Configured AW credentials are null!"), so the build had non-empty text.
- Bad Base64 would throw Guava's own error, not this one.
- So Base64 worked and Blowfish failed. The text isn't valid data for this app's key. Both QA and PR fail in the same run, so the cause is common to both. This is my reading of the code.

Why this device only (not confirmed)

- The log has no app version, so we don't know which build it ran.
- My local `secrets.properties` has empty placeholders, so I couldn't check the real values.
- The same device also has other setup errors (old `st%STORE%` server address, missing bootstrap file). It may run an old or wrongly built app.

Other things found (not in the 26 rows)

- `ConfigurationManager.java` line 361: `_cachedBuildType` is an `int` set from `bootstrapConfig == null ? null : getBuildType()`. A missing bootstrap config or build type can cause a NullPointerException.
- The encryption key is written in the source code and the cipher uses plain "Blowfish" mode. This is a security weakness, unrelated to these rows.

What to do

1. In `CryptoHelper.decryptBlowfish`, don't return null. Throw or log the real exception.
2. Log which build type and which credential (QA or PR) failed, never the values. Log once, not twice.
3. Find the app build on device `25160524206657`, check which `CREDENTIALS_*` value it was built with, and rebuild if it's wrong.
4. Fix the null/int problem at line 361.
5. Later: move the key out of the source code and use a safer cipher mode.

Done when: a decrypt failure shows its real reason and build type, and device `25160524206657` gets working credentials.



## TICKET 1

TITLE: Support Service logs "Unable to resolve package signature" for printerservice and fp_filemanager, and init seems to repeat many times per minute on one device (131 rows, only 20 bursts)

WHAT IS WRONG

131 error rows (28 Sep to 23 Oct 2026):

- "Unable to resolve package signature for com.homedepot.sa.xp.printerservice" - 65 rows
- "... com.homedepot.sa.xp.fp_filemanager" - 62 rows
- "... com.homedepot.sa.xp.firstphone" - 2 rows

The 131 rows are only 20 bursts. 19 of the 20 bursts contain both packages together.

Device `24198524213086` has 86 of the rows. Its biggest burst: 34 rows in ONE minute (14 Oct, 17:40). That is about 17 runs of the same loop in one minute.

WHY IT HAPPENS (from the code)

1. At init the Support Service loops over `config.manageFilePermissionWhiteList` and calls `scopedFilePermissionEnable` for each package (`SilentFullInitActivity.kt` line 367, `SilentInitReceiver.kt` line 160; also `PackageInstallReceiver.kt` lines 64 and 75).
2. `ZebraPlatform.kt` lines 485-489: it asks for the app's signature (`FirstPhoneUtils.getCallerSignatureBase64Encoded`). If that returns null it logs an error and returns.
3. The signature call returns null when the app cannot be found or its signature cannot be read. The log cannot tell these two cases apart.
4. The built-in config (`defaultsupportserviceconfig.json`) has an EMPTY `manageFilePermissionWhiteList`. So these two packages come from another config the device loaded (cloud or SD card).

NOT SURE YET

- Most likely these apps are simply not installed on those devices, but this is only a guess. Nothing else in the log mentions these two packages.
- Why init runs about 17 times a minute on device `24198524213086` (restart loop? repeated trigger?). Not checked.
- Which config (cloud or SD card) holds the two packages.

WHAT TO DO

1. If the package is not installed, skip it and log at INFO. Log the reason (not installed vs signature unreadable) when it is an error.
2. Find out why init repeats on device `24198524213086` on 14-15 Oct (see the other logs from that time).
3. Check the config source: should printerservice and fp_filemanager be in `manageFilePermissionWhiteList` for these devices?
4. Log once per init, not once per loop pass.

DONE WHEN

A package that is not installed does not create an error, and init does not repeat many times a minute.

## TICKET 2

TITLE: Support Service removes apps as "untrusted" that may be legitimate - app whitelist on devices differs from the built-in whitelist (84 rows)

### WHAT IS WRONG

84 rows (24 Sep to 27 Oct 2026): "Untrusted package installed - `<package>` - Removing..." The app is then uninstalled.

Packages removed most often:

- `com.homedepot.sa.xp.homedepotbrowser` - 22
- `com.paloaltonetworks.globalprotect` (VPN) - 10
- `com.zebra.wfc.central` - 8
- `com.homedepot.sa.xp.hdbluetoothmanager` - 8
- `ca.homedepot.mytoolbelt` - 4
- `com.google.android.apps.work.clouddpc` (Android device policy app) - 3
- Others: Google apps (Maps, YouTube, Gmail, Photos, Calendar, Dialer ...) and some Zebra/Symbol apps.

It happens on all 8 devices (27 rows have no device ID). Peak: 23 Oct (23 rows).

### WHAT THE CODE DOES (verified)

\- `PackageInstallReceiver.kt` lines 49-55: if `isAppInstallWhiteListEnabled` is true and the package is not in `appInstallWhiteList`, it logs an error and uninstalls the app. This is a security feature. The error line is a record that it worked.

### WHAT WE VERIFIED

1. `homedepotbrowser` and `hdbluetoothmanager` ARE on the built-in whitelist (`defaultsupportserviceconfig.json`, since the first commit on 17 Feb 2026), but were removed 30 times. So these devices were NOT using the built-in whitelist. The code loads config from cloud first, then the SD card, then the built-in file. We have not seen which one the devices used.
2. `ca.homedepot.mytoolbelt`: commit `cd64c58` (11 Aug 2026, #27) renamed `ca.homedepot` to `com.homedepot` in the whitelist. A device still installed the OLD package name, so it is removed. This one is explained by the rename.
3. `globalprotect`, `clouddpc`, `zebra.wfc.central`, `printerservice`, `fp_filemanager` were never on the built-in whitelist (not in its git history).
4. The 27 rows with no device ID include many standard Google apps (Maps, YouTube, Gmail ...). This looks like a newly set-up or reset device, where Android installs its default apps before the whitelist is ready. This is my inference, not confirmed.
5. Several removals have the same millisecond (for example 28 Sep 16:58:13: GlobalProtect and the browser). Several installs were handled together.

### NOT SURE YET

- Which whitelist the devices actually used and why it lacks `homedepotbrowser` and `hdbluetoothmanager` (stale SD-card file, cloud config, or something else).
- Whether the removal of GlobalProtect, Zebra Workstation Connect and Google's `clouddpc` was wanted. Removing the device policy app could matter.

### WHAT TO DO

1. Config owner: confirm which whitelist is active on devices and why it differs from the built-in one.
2. Decide if GlobalProtect, `com.zebra.wfc.central` and `clouddpc` should be on the whitelist; add them if yes.
3. Find the source still installing the old `ca.homedepot.mytoolbelt` package (MDM app list) and update it to `com.homedepot.mytoolbelt`.
4. Log at WARNING (or INFO with a clear "removed by policy") instead of ERROR, and log which whitelist was used (cloud / SD card / built-in).
5. Check if devices in first-time setup should skip removal until the config is loaded.

### DONE WHEN

Only truly unapproved apps are removed, the active whitelist is known, and legitimate apps are no longer removed.







## TICKET 3

TITLE: Launcher logs "System is running low on memory" as an error (18 rows)

### WHAT IS WRONG

18 error rows from the launcher (`com.homedepot.sa.xp.firstphone`), tag `LauncherApplication`: "System is running low on memory".

### WHY IT HAPPENS (from the code)

`LauncherApplication.java` lines 162-165: `onLowMemory()` is called by Android when memory is low. The code logs `LogProcessor.e(...)` every time it is called. It is a system warning, not an app failure.

### NOT SURE YET

\- Which devices and which apps used the memory. The 18 rows were not checked per device.

### WHAT TO DO

1. Log at WARNING instead of ERROR.
2. Add device model, free memory and the time since app start to the log, so real memory problems can be found.
3. Check which devices send this message often.

### DONE WHEN

Low memory is a warning with useful details.



## TICKET 4

TITLE: "Metrics disabled" trace messages are logged as ERROR in all apps (2,485 rows), and some contain user IDs and serial numbers

### WHAT IS WRONG

2,485 error rows (22 Sep to 27 Oct 2026) are not errors. They are trace messages about metrics that are switched off. Every metric call writes 2 rows:

1. "Sending metric: `<name>` with value: ... from: `<place>`" - 1,241 rows
2. "Metric Point Tags: `{...}`" - 1,244 rows

That is about 1,240 metric calls. Nothing is actually sent.

Rows by app:

- Launcher (`com.homedepot.sa.xp.firstphone`): 2,343
- Log processor library (`SocketMonitor`): 108
- Support tool (`com.homedepot.sa.xp.supporttool`): 22 (code not found locally; probably the same pattern)
- GoLauncher: 12

Top metric calls in the launcher:

- `LauncherViewModel.startScanner` (369)
- `LauncherViewModel.sendLaunchMetric` (359)
- `PlatformConfigFacade.logMetrics` (292)
- `LauncherViewModel.performPostLoginTasks` (69)
- `UserLogoutJob.call` (36)

Most rows are from devices `25147524201072` (1,079) and `24198524213086` (524).

### WHY IT HAPPENS (from the code)

The metrics code writes two `LogProcessor.e` (ERROR) lines, then stops before the network call:

- Launcher (CAD_Launcher and CAD_Launcher02): `MetricsLoggingManager.java` lines 110-111 log the two lines at ERROR, then `if(true) return;` (about line 115). Commit `91cf367` (10 Sep 2026) added this. Its comment says the call is off "to avoid spamming the logs" because the Wavefront endpoint is not live yet. The same commit adds the two ERROR lines, so every metric still writes 2 error rows.
- Support Service: `MetricsLoggingManager.kt` lines 72-74 (ERROR x2, then `return`), commit `7209fd3` (22 Sep 2026).
- Log processor library: `SocketMonitor.kt` lines 203-205 (ERROR x2, then `return`), commit `04e7ccd` (22 Sep 2026).
- GoLauncher: `GoRepository.kt` lines 509-513 (ERROR x2, then `return@let`).

All rows are dated 22 Sep or later, after sending was switched off.

### IDENTIFIERS IN THE LOG

- 138 tag rows contain a user ID (for example `userMTB003` on login/logout metrics).
- 54 rows contain the device serial number and store number (SocketMonitor tags).
- GoRepository builds its tags from the device MAC and IP address (lines about 505-506). I did not check whether the 12 GoLauncher rows show them.

These go into the error log that is uploaded. Someone should confirm this is allowed.

### NOT SURE YET

- The support tool code (not found locally).
- Whether the metrics will be turned on again, and when.

### RELATED

Same root as the Network ticket about metrics failing to reach the production Wavefront host. Sending is now off, but the logging of it remains.

### WHAT TO DO

1. Log these two lines at DEBUG (or remove them) in all four places, and in the support tool.
2. Do not log user IDs, serial numbers, MAC or IP in these lines.
3. When metrics are turned on again, use the correct Wavefront host for each environment and log only real failures at ERROR.

### DONE WHEN

Switched-off metrics write no ERROR rows, and no user IDs or serial numbers are in the error log.









  
