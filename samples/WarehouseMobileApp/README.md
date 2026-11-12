# Warehouse Management mobile app telemetry

The Warehouse Management mobile app can send telemetry to your own Azure Application Insights resource. This guide explains what you get there and shows, with examples, how to answer common questions about your warehouse devices, network, and workers' operations.

> [!IMPORTANT]
> Most of the events and fields in this guide are available only from **Warehouse Management mobile app version 4.2.3.0**. This includes the `dependencies` and `exceptions` data, `ServerInvokeFailure`, `OperationFlowTrack`, `MediaCapture`, the failure details (`errorClass`, `statusCode`, `errorActivityId`), and the retry and run-tracking fields (`activityId`, `appRunId`, `eventSequence`). Devices on earlier versions send only a basic set of events, such as `ClientRoundTrip`, `FailedServerInvoke`, and `ServiceInvokeException`, so queries in this guide return partial or no results for them. Update your devices to version 4.2.3.0 or later to get the full telemetry. Use [Which app versions are in use?](#which-app-versions-are-in-use) to check your fleet.

All queries are written in Kusto Query Language (KQL). New to KQL? Start with [Querying telemetry with KQL](../KQL).

## Contents

- [Quick start](#quick-start)
- [Where to find the data](#where-to-find-the-data)
- [What a row looks like](#what-a-row-looks-like)
- [Common questions, answered with queries](#common-questions-answered-with-queries)
  - [Is the app fast for my workers?](#is-the-app-fast-for-my-workers)
  - [Is it the server or the network?](#is-it-the-server-or-the-network)
  - [Where is my Wi-Fi weak?](#where-is-my-wi-fi-weak)
  - [Which requests are failing, and why?](#which-requests-are-failing-and-why)
  - [Did a retry save the worker?](#did-a-retry-save-the-worker)
  - [How long do warehouse operations take?](#how-long-do-warehouse-operations-take)
  - [How do workers scan?](#how-do-workers-scan)
  - [Which app versions are in use?](#which-app-versions-are-in-use)
  - [What happened on one device?](#what-happened-on-one-device)
- [Set up alerts](#set-up-alerts)
- [Understanding failures](#understanding-failures)
- [Field reference](#field-reference)
- [Event reference](#event-reference)
- [Tips and caveats](#tips-and-caveats)

## Quick start

1. Create an Application Insights resource in the Azure portal, and copy its **connection string**.
2. In Dynamics 365 Supply Chain Management, enter the connection string in the Warehouse Management mobile app telemetry settings.
3. Sign in on a device with the mobile app and do some work, for example open a menu item and complete a pick.
4. In the Application Insights resource, open **Logs** and run:

```kusto
customEvents
| where timestamp > ago(1h)
| where name startswith "Warehouse.MobileApp."
| summarize events = count() by name
| order by events desc
```

If you see rows, telemetry is flowing. The device receives the connection string when the worker signs in, so a worker who was already signed in must sign in again before data appears.

> [!TIP]
> Use the connection string, not the instrumentation key. The instrumentation key only exists for older configurations.

### What is never sent

- Scanned barcode values. Scan events carry only the type, the source, and the length of a scan.
- Sign-in diagnostics that contain identifiable information.

## Where to find the data

| Table | Workspace-based table | What it holds |
| --- | --- | --- |
| `customEvents` | `AppEvents` | All app events. Names start with `Warehouse.MobileApp.` |
| `dependencies` | `AppDependencies` | One row per request from a device to Supply Chain Management. Powers the **Performance** and **Application map** views. |
| `exceptions` | `AppExceptions` | App errors with stack traces. Powers the **Failures** view. |

The queries in this guide use the classic table names, which work in the **Logs** view of the Application Insights resource. If you query a Log Analytics workspace directly, use the workspace table names, `Properties` instead of `customDimensions`, and `Measurements` instead of `customMeasurements`.

### Text and numbers

Application Insights stores text in `customDimensions` and numbers in `customMeasurements`. Most mobile app fields, including most durations, are text. Convert them before you calculate:

```kusto
| extend networkMs = todouble(customDimensions.roundTripLatencyDurationInMilliseconds)
```

A few fields are numbers, for example `numberOfRetries`, `requestBytes`, `responseBytes`, `count`, and `suppressedCount`. To be safe, read them from both columns:

```kusto
| extend retries = coalesce(todouble(customMeasurements.numberOfRetries), todouble(customDimensions.numberOfRetries))
```

## What a row looks like

### A successful request

Every time a device gets a page from the server, a `ClientRoundTrip` event is written:

```json
{
  "timestamp": "2025-03-04T10:15:02.114Z",
  "name": "Warehouse.MobileApp.ClientRoundTrip",
  "user_Id": "8f1c3a77e2",
  "client_Model": "TC52",
  "application_Version": "3.4.12",
  "customDimensions": {
    "eventId": "WHS000006",
    "deviceId": "8f1c3a77e2",
    "deviceName": "Forklift-07",
    "devicePlatform": "android",
    "companyName": "USMF",
    "sessionId": "4c0b51d2-…",
    "appRunId": "b71e…",
    "eventSequence": "42",
    "operationName": "getNextForm",
    "activityId": "e0d4…",
    "roundTripLatencyDurationInMilliseconds": "137",
    "backendProcessingTime": "275",
    "renderingDurationInMilliseconds": "38",
    "lastKnownWarehouse": "24",
    "lastKnownWMSLocation": "BULK-001",
    "lastKnownDataAreaId": "usmf",
    "wifiSignalStrength": "Good",
    "batteryLevel": "0.71"
  }
}
```

How to read it: the worker on **Forklift-07** waited about **412 ms** for the next page. The server spent **275 ms** (`backendProcessingTime`) and the network **137 ms** (`roundTripLatencyDurationInMilliseconds`). Add the two to get the time the worker waited. The device was last seen at location **BULK-001** in warehouse **24**, with **Good** Wi-Fi.

### A finished warehouse operation

When a worker finishes or leaves an operation, such as a pick, an `OperationFlowTrack` event summarizes it:

```json
{
  "name": "Warehouse.MobileApp.OperationFlowTrack",
  "customDimensions": {
    "eventId": "WHS000032",
    "formName": "WHSWorkExecute",
    "outcome": "completed",
    "status": "success",
    "operationDurationMs": "48210",
    "fillDurationTotalMs": "31050",
    "requestDurationTotalMs": "9870",
    "idleDurationMs": "0",
    "requestCount": "7",
    "pageCount": "6",
    "submitCount": "5",
    "errorMessageCount": "1",
    "scanIntentCount": "4",
    "scanKeyboardCount": "1"
  }
}
```

How to read it: the operation took **48 seconds**. The worker spent **31 seconds** filling in pages and **10 seconds** waiting for the server, saw **1 error message**, and scanned **5 times**.

### A failed request

When a request fails, a `ServerInvokeFailure` event explains why. This one was throttled by the server (HTTP 429):

```json
{
  "name": "Warehouse.MobileApp.ServerInvokeFailure",
  "customDimensions": {
    "operationName": "getNextForm",
    "activityId": "e0d4…",
    "statusCode": "429",
    "errorClass": "http",
    "errorMessage": "Too Many Requests: …",
    "errorActivityId": "",
    "deviceName": "Forklift-07",
    "wifiSignalStrength": "Moderate"
  },
  "customMeasurements": { "requestBytes": 1834 }
}
```

The same attempt also appears in `dependencies`, with the same ID:

```json
{
  "itemType": "dependency",
  "name": "getNextForm",
  "id": "e0d4…",
  "target": "Dynamics 365 SCM",
  "type": "HTTP",
  "resultCode": "429",
  "success": false,
  "customDimensions": { "errorClass": "http" }
}
```

A Wi-Fi drop looks similar, but with `resultCode` `0`, an empty `statusCode`, and `errorClass` `connection`.

## Common questions, answered with queries

### Is the app fast for my workers?

Median and slowest (95th percentile) response time per warehouse:

```kusto
customEvents
| where timestamp > ago(7d)
| where name == "Warehouse.MobileApp.ClientRoundTrip"
| extend warehouse = tostring(customDimensions.lastKnownWarehouse),
         latency = todouble(customDimensions.roundTripLatencyDurationInMilliseconds)
                 + todouble(customDimensions.backendProcessingTime)
| summarize requests = count(),
            p50 = percentile(latency, 50),
            p95 = percentile(latency, 95)
          by warehouse
| order by p95 desc
```

Example result:

| warehouse | requests | p50 | p95 |
| --- | --- | --- | --- |
| 24 | 18 204 | 380 | 2 950 |
| 11 | 9 812 | 310 | 940 |

Warehouse 24 has a long tail: 5% of its requests take about 3 seconds. The next query shows why.

### Is it the server or the network?

`backendProcessingTime` is the time the server spent. `roundTripLatencyDurationInMilliseconds` is the network time: how long the device waited minus the server time.

```kusto
customEvents
| where timestamp > ago(1d)
| where name == "Warehouse.MobileApp.ClientRoundTrip"
| extend network = todouble(customDimensions.roundTripLatencyDurationInMilliseconds),
         server = todouble(customDimensions.backendProcessingTime)
| where network + server > 3000
| summarize slowRequests = count(),
            medianServerMs = percentile(server, 50),
            medianNetworkMs = percentile(network, 50)
          by warehouse = tostring(customDimensions.lastKnownWarehouse),
             operationName = tostring(customDimensions.operationName)
| order by slowRequests desc
```

- High `medianNetworkMs` and low `medianServerMs`: look at Wi-Fi coverage and access points.
- High `medianServerMs`: look at server load, batch jobs, and the process behind the page.

### Where is my Wi-Fi weak?

Devices that report **Poor** signal on more than 20% of their requests:

```kusto
customEvents
| where timestamp > ago(1d)
| where name == "Warehouse.MobileApp.ClientRoundTrip"
| summarize requests = count(),
            poor = countif(tostring(customDimensions.wifiSignalStrength) == "Poor")
          by deviceName = tostring(customDimensions.deviceName),
             warehouse = tostring(customDimensions.lastKnownWarehouse)
| extend poorPct = round(100.0 * poor / requests, 1)
| where poorPct > 20
| order by poorPct desc
```

Build a coverage map of your warehouse by location:

```kusto
customEvents
| where timestamp > ago(7d)
| where name == "Warehouse.MobileApp.ClientRoundTrip"
| extend signal = tostring(customDimensions.wifiSignalStrength)
| where signal in ("Poor", "Moderate", "Good", "Great")
| summarize requests = count(),
            poorPct = round(100.0 * countif(signal == "Poor") / count(), 1)
          by warehouse = tostring(customDimensions.lastKnownWarehouse),
             location = tostring(customDimensions.lastKnownWMSLocation)
| where requests >= 20
| order by poorPct desc
```

`lastKnownWMSLocation` is the last location the worker scanned, so it shows roughly where the device was.

Count how often devices lose their connection:

```kusto
customEvents
| where timestamp > ago(7d)
| where name == "Warehouse.MobileApp.NetworkConnectionStateChange"
| where tostring(customDimensions.currentState) != tostring(customDimensions.previousState)
| summarize changes = count() by deviceName = tostring(customDimensions.deviceName), bin(timestamp, 1d)
| order by changes desc
```

### Which requests are failing, and why?

Failures by type over the last day, as a chart:

```kusto
dependencies
| where timestamp > ago(1d)
| where target == "Dynamics 365 SCM" and success == false
| summarize failures = count() by bin(timestamp, 15m), errorClass = tostring(customDimensions.errorClass)
| render timechart
```

The most common failures, with the message the server returned:

```kusto
customEvents
| where timestamp > ago(7d)
| where name == "Warehouse.MobileApp.ServerInvokeFailure"
| where tostring(customDimensions.errorClass) != "abort"
| summarize failures = count(), devices = dcount(tostring(customDimensions.deviceId))
          by statusCode = tostring(customDimensions.statusCode),
             errorClass = tostring(customDimensions.errorClass),
             operationName = tostring(customDimensions.operationName),
             errorMessage = tostring(customDimensions.errorMessage)
| order by failures desc
```

Network problems by device. DNS and TLS failures usually point to network or device configuration, not the server:

```kusto
customEvents
| where timestamp > ago(7d)
| where name == "Warehouse.MobileApp.ServerInvokeFailure"
| extend errorClass = tostring(customDimensions.errorClass)
| where errorClass in ("dns", "tls", "connection", "timeout")
| summarize failures = count()
          by errorClass,
             deviceName = tostring(customDimensions.deviceName),
             wifi = tostring(customDimensions.wifiSignalStrength)
| order by failures desc
```

Need help from your Dynamics 365 administrator or Microsoft support? Give them the server activity ID of the failure:

```kusto
customEvents
| where timestamp > ago(3d)
| where name == "Warehouse.MobileApp.ServerInvokeFailure"
| where isnotempty(customDimensions.errorActivityId)
| project timestamp,
          deviceName = tostring(customDimensions.deviceName),
          operationName = tostring(customDimensions.operationName),
          statusCode = tostring(customDimensions.statusCode),
          errorMessage = tostring(customDimensions.errorMessage),
          serverActivityId = tostring(customDimensions.errorActivityId)
| order by timestamp desc
```

### Did a retry save the worker?

The app retries some failures automatically (see [Retries](#retries)). Each attempt is a separate `dependencies` row, and all attempts of one request share the same `id`. This query separates requests that recovered from requests the worker actually saw fail:

```kusto
dependencies
| where timestamp > ago(1d)
| where target == "Dynamics 365 SCM"
| summarize attempts = count(), succeeded = countif(success) by id, name
| summarize requests = count(),
            failedForGood = countif(succeeded == 0),
            recoveredByRetry = countif(succeeded > 0 and attempts > 1)
          by name
| extend failedPct = round(100.0 * failedForGood / requests, 2)
| order by failedForGood desc
```

### How long do warehouse operations take?

Throughput, abandonment, and where the time goes, per mobile device menu item form:

```kusto
customEvents
| where timestamp > ago(7d)
| where name == "Warehouse.MobileApp.OperationFlowTrack"
| extend form = tostring(customDimensions.formName),
         outcome = tostring(customDimensions.outcome),
         durationSec = todouble(customDimensions.operationDurationMs) / 1000,
         fillSec = todouble(customDimensions.fillDurationTotalMs) / 1000,
         waitSec = todouble(customDimensions.requestDurationTotalMs) / 1000
| summarize operations = count(),
            abandonedPct = round(100.0 * countif(outcome == "abandoned") / count(), 1),
            medianSec = percentile(durationSec, 50),
            medianFillSec = percentile(fillSec, 50),
            medianWaitSec = percentile(waitSec, 50)
          by form
| order by operations desc
```

- `medianFillSec` is time the worker spends on the pages (reading, scanning, typing).
- `medianWaitSec` is time waiting for the server. If it is a large share of `medianSec`, see [Is it the server or the network?](#is-it-the-server-or-the-network)
- A high `abandonedPct` can point to a confusing or blocked process.

### How do workers scan?

```kusto
customEvents
| where timestamp > ago(7d)
| where name == "Warehouse.MobileApp.OperationFlowTrack"
| summarize intent = sum(toint(customDimensions.scanIntentCount)),
            keyboard = sum(toint(customDimensions.scanKeyboardCount)),
            bigBlock = sum(toint(customDimensions.scanBigBlockCount)),
            charByChar = sum(toint(customDimensions.scanCharByCharCount))
          by deviceModel = tostring(customDimensions.deviceModel)
```

A high `charByChar` count usually means the scanner is set up as a keyboard that types one character at a time, which is slow. Configure the scanner to send intents instead.

### Which app versions are in use?

```kusto
customEvents
| where timestamp > ago(7d)
| where name startswith "Warehouse.MobileApp."
| summarize devices = dcount(user_Id), lastSeen = max(timestamp)
          by application_Version, platform = tostring(customDimensions.devicePlatform)
| order by devices desc
```

Find devices that still need an update to get the full telemetry described in this guide:

```kusto
customEvents
| where timestamp > ago(7d)
| where name startswith "Warehouse.MobileApp."
| summarize arg_max(timestamp, application_Version)
          by deviceName = tostring(customDimensions.deviceName), user_Id
| where parse_version(application_Version) < parse_version("4.2.3.0")
| project deviceName, user_Id, version = application_Version, lastSeen = timestamp
| order by lastSeen desc
```

### What happened on one device?

Step 1, find the device's recent app runs:

```kusto
customEvents
| where timestamp > ago(1d)
| where customDimensions.deviceName == "Forklift-07"
| summarize start = min(timestamp), end = max(timestamp), events = count()
          by appRunId = tostring(customDimensions.appRunId)
| order by start desc
```

Step 2, replay one run in order:

```kusto
let run = "<appRunId from step 1>";
customEvents
| where customDimensions.appRunId == run
| extend seq = toint(customDimensions.eventSequence),
         eventTime = iff(customDimensions.offline == "true",
                         todatetime(customDimensions.originalTimestamp), timestamp)
| project eventTime, seq, name,
          route = tostring(customDimensions.route),
          formName = tostring(customDimensions.formName),
          customDimensions
| order by seq asc
```

Events recorded while the device was offline arrive later. For those, `offline` is `"true"` and `originalTimestamp` holds the real event time.

## Set up alerts

Use these queries as **Log search alert** rules in Azure Monitor. A typical setup evaluates every 5 minutes and fires when the query returns rows. See [Alerting on telemetry](../Alerts) for how to create a rule.

More than 10% of requests failing in the last 15 minutes:

```kusto
dependencies
| where timestamp > ago(15m)
| where target == "Dynamics 365 SCM"
| summarize attempts = count(), failed = countif(success == false)
| where attempts >= 20
| extend failureRate = 100.0 * failed / attempts
| where failureRate > 10
```

Server errors (5xx) on several devices at once:

```kusto
dependencies
| where timestamp > ago(15m)
| where target == "Dynamics 365 SCM" and toint(resultCode) between (500 .. 599)
| summarize failures = count(), devices = dcount(user_Id)
| where devices >= 3
```

More than 5 critical app errors in the last 15 minutes:

```kusto
exceptions
| where timestamp > ago(15m) and severityLevel == 4
| summarize critical = count(), devices = dcount(user_Id)
| where critical > 5
```

Unusual spikes of failed requests compared with the past week:

```kusto
dependencies
| where timestamp > ago(7d)
| where target == "Dynamics 365 SCM" and success == false
| make-series failures = count() default = 0 on timestamp step 15m
| extend (anomalies, score, baseline) = series_decompose_anomalies(failures, 2.5)
| mv-expand timestamp to typeof(datetime), failures to typeof(long),
            anomalies to typeof(int), baseline to typeof(double)
| where anomalies > 0
| project timestamp, failures, expected = round(baseline, 1)
```

## Understanding failures

### Failure types

`errorClass` appears on failed `dependencies` rows and on `ServerInvokeFailure` events.

| `errorClass` | What happened | Typical cause | What to check |
| --- | --- | --- | --- |
| `http` | The server answered with an error status | Server error, throttling, expired sign-in | `statusCode`, `errorMessage`, `errorActivityId` |
| `timeout` | No answer in time | Slow server, weak Wi-Fi, large page | `wifiSignalStrength`, server load, the **Server request timeout** app setting |
| `connection` | The connection failed or dropped | Wi-Fi drop, device offline, access point roaming | `NetworkConnectionStateChange` events near the failure |
| `dns` | The server name could not be resolved | DNS or network configuration | Which access points and devices are affected |
| `tls` | The secure connection failed | Certificate, proxy, or wrong device clock | Proxy and certificate settings, device date and time |
| `abort` | The request was cancelled on the device | The operating system stopped the request | Not a server failure. Exclude it from failure counts. |
| `unknown` | Anything else | — | `errorMessage` |

### Retries

| Situation | Retried? |
| --- | --- |
| 408, 429, and 5xx responses | Yes |
| 400, 401, 403, 404, and other 4xx responses | No |
| Request timeout | Yes, for requests that only read data |
| Connection, DNS, and TLS errors | No |
| Requests that change data, such as `getNextForm` | Only on 429 (Too Many Requests) |

`getNextForm` is tried at most 3 times (the first attempt and 2 retries).

### Which events a failure writes

| Event | Written for | Good for |
| --- | --- | --- |
| `dependencies` row with `success == false` | Every failed attempt, except cancelled ones | Failure rates and the **Failures** view |
| `ServerInvokeFailure` | Every failed attempt | Diagnosis: status, failure type, message, server activity ID |
| `FailedServerInvoke` (`WHS000030`) | Every failed attempt | The error message shown to the worker, in the device language |
| `ServiceInvokeException` (`WHS000031`) | Every failed attempt that can be retried | Counting retryable failures per device |

`ServiceInvokeException` means the failure *could* be retried, not that a retry happened. The last attempt also writes one when no retries are left. `numberOfRetries` is the number of the attempt that failed, where `1` is the first attempt.

> [!TIP]
> `FailedServerInvoke` reports timeouts as 408 even when no HTTP response arrived. For the real HTTP status, use `dependencies.resultCode` or `ServerInvokeFailure.statusCode`. Both are `0` or empty when no response arrived.

## Field reference

### On every row

| Field | Example | Meaning |
| --- | --- | --- |
| `user_Id` (column) | `8f1c3a77e2` | Device ID |
| `client_Model` (column) | `TC52` | Device model |
| `application_Version` (column) | `3.4.12` | App version |
| `deviceId` | `8f1c3a77e2` | Device ID |
| `deviceName` | `Forklift-07` | Device name set by your operator. Empty when not set. |
| `devicePlatform` | `android` | Operating system |
| `deviceModel`, `deviceBrand` | `TC52`, `Zebra` | Hardware |
| `companyName` | `USMF` | Legal entity |
| `environmentId` | `c2a1…` | Microsoft Entra tenant ID of the signed-in worker |
| `sessionId` | `4c0b51d2-…` | Server session. Matches the session ID on the server. |
| `appRunId` | `b71e…` | One app run on a device. A new value starts after sign-out. |
| `eventSequence` | `42` | Order of the event within the app run |
| `route` | `/nextForm` | Current app screen |
| `formName` | `WHSWorkExecute` | Current server form |
| `offline`, `originalTimestamp` | `true`, `2025-03-04T10:15:00Z` | Present only on events recorded while offline |

### On request events

| Field | Meaning |
| --- | --- |
| `operationName` | Server operation, for example `getNextForm` |
| `activityId` | Request ID created on the device. Shared by all attempts, and equal to `dependencies.id`. |
| `roundTripLatencyDurationInMilliseconds` | Network time: how long the device waited, minus the server time |
| `backendProcessingTime` | Time the server spent, in milliseconds |
| `renderingDurationInMilliseconds` | Time to draw the page on the device |
| `wifiSignalStrength` | `Poor`, `Moderate`, `Good`, `Great`, or `Unknown` (not on Wi-Fi) |
| `networkAccess`, `networkConnectionProfiles` | Connectivity at the time of the request |
| `lastKnownWarehouse`, `lastKnownWMSLocation`, `lastKnownDataAreaId` | Last warehouse, location, and company the worker used |
| `batteryLevel`, `batteryState`, `powerSource`, `isEnergySaverTurnedOn` | Power state |

### Following a story across rows

| To follow | Use |
| --- | --- |
| One request and its retries | `activityId` in `customEvents`, `id` in `dependencies` |
| One app run on a device | `appRunId`, ordered by `toint(eventSequence)` |
| One server session | `sessionId` |
| A failure in server logs | `errorActivityId` on `ServerInvokeFailure` |
| A device over time | `user_Id` or `deviceId`, with `deviceName` for display |

## Event reference

All event names start with `Warehouse.MobileApp.`

| Event | When it is written |
| --- | --- |
| `ClientRoundTrip` (`WHS000006`) | A request to the server succeeded |
| `ServerInvokeFailure` | A request attempt failed. Includes the failure details. |
| `FailedServerInvoke` (`WHS000030`) | A request attempt failed. Includes the message shown to the worker. |
| `ServiceInvokeException` (`WHS000031`) | A request attempt failed with an error that can be retried |
| `OperationFlowTrack` (`WHS000032`) | A warehouse operation, such as a pick, ended |
| `MediaCapture` (`WHS000033`) | A photo or video capture session ended |
| `LoginSubmission`, `Logout` | A worker signed in or out |
| `ConnectionChanged`, `ConnectionSwitch`, `ConnectionSwitchResult` | Connection settings were changed or switched |
| `NetworkConnectionStateChange` | The device's connectivity changed |
| `NetworkConnectionPeriodicStatus` | Every 60 seconds while the connection is poor |
| `Scanner`, `CameraBarcodeScan` | A barcode was scanned (never the scanned value) |
| `Exception`, `CriticalLogs`, `warning` | An app error or warning |
| `TelemetryQueueOverflow` | The device was offline so long that some telemetry was dropped |

## Tips and caveats

- Most fields are text. Use `todouble()` or `toint()` before you calculate.
- Durations ending in `Ms` or `InMilliseconds` are in milliseconds.
- `dependencies` counts attempts, not requests. A request that was retried twice has three rows with the same `id`.
- Cancelled requests (`errorClass == "abort"`) are not server failures. They have no `dependencies` row; exclude them from `ServerInvokeFailure` counts.
- Repeated identical errors are grouped, at most one row per minute per device. `count` is how many occurrences the row stands for. To count real occurrences, sum `coalesce(count, 1)`.
- `FailedServerInvoke` messages are translated to the device language. Group by status code for language-independent results.
- `installationId` is always `not supported`. Use `deviceId` to identify a device.
- Most events and fields in this guide require app version 4.2.3.0 or later. When you compare periods or devices, filter by `application_Version`, because devices on earlier versions do not send them.

# Disclaimer
Microsoft Corporation (“Microsoft”) grants you a nonexclusive, perpetual, royalty-free right to use and modify the software code provided by us for the purposes of illustration  ("Sample Code") and to reproduce and distribute the object code form of the Sample Code, provided that you agree: (i) to not use our name, logo, or trademarks to market your software product in which the Sample Code is embedded; (ii) to include a valid copyright notice on your software product in which the Sample Code is embedded; and (iii) to indemnify, hold harmless, and defend us and our suppliers from and against any claims or lawsuits, whether in an action of contract, tort or otherwise, including attorneys’ fees, that arise or result from the use or distribution of the Sample Code or the use or other dealings in the Sample Code. Unless applicable law gives you more rights, Microsoft reserves all other rights not expressly granted herein, whether by implication, estoppel or otherwise. 

THE SAMPLE CODE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY, FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL MICROSOFT OR ITS LICENSORS BE LIABLE FOR ANY DIRECT, INDIRECT, INCIDENTAL, SPECIAL, EXEMPLARY, OR CONSEQUENTIAL DAMAGES (INCLUDING, BUT NOT LIMITED TO, PROCUREMENT OF SUBSTITUTE GOODS OR SERVICES; LOSS OF USE, DATA, OR PROFITS; OR BUSINESS INTERRUPTION) HOWEVER CAUSED AND ON ANY THEORY OF LIABILITY, WHETHER IN CONTRACT, STRICT LIABILITY, OR TORT (INCLUDING NEGLIGENCE OR OTHERWISE) ARISING IN ANY WAY OUT OF THE USE OF THE SAMPLE CODE, EVEN IF ADVISED OF THE POSSIBILITY OF SUCH DAMAGE.
