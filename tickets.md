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
