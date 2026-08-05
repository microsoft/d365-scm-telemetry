## Warehoouse Mobile App

Having a superior Wi-Fi network results in shipping on time. You can use the WMA telemetry to monitor and improve the Wi-Fi coverage in your warehouse.

The telemetry includes the following data points:
* backendProcessingTime: The time (in milliseconds) the backend spent processing the request. The time is measured by the IIS, so it includes literally everything happening at the backend.
* roundTripLatencyDurationInMilliseconds : The total network latency, measured as the total time the device awaited a response minus the backend processing.
* renderingDurationInMilliseconds: The time for the device to render the page. This depends on the complexity of the page to render (how many controls), and the GPU on the device.
* wifiSignalStrength: The current measured strength of the Wi-Fi signal.
* lastKnownWMSLocation: The last known location of the device, typically determined by scan of a location or license plate during a pick or put operation. This can be used with the wifiSignalStrength to create a coverage map of Wi-Fi strength across the locations.

![image](https://github.com/microsoft/d365-scm-telemetry/assets/28089437/1a396dad-31cc-4cc5-842f-ca9d91f0caec)

### How long the work takes (Warehouse Management mobile app version 4.1.5.0)

The data points above measure how long the *backend* takes to answer. From version 4.1.5.0 the app also reports how long the *work* takes, so a slow step can be attributed to the server or to a page that is simply hard to complete.

Added to `Warehouse.MobileApp.ClientRoundTrip` (WHS000006):
* timeSpentLastPage: The time (in milliseconds) the worker spent on the page that this request replaced. Only reported on the round trip that actually changed the page, so it measures dwell per page rather than per request, and is empty for the first page of a session. A page counts as new when its structure changes (its layout and its set of fields and buttons), so retyping or correcting a value does not restart the measurement.
* idleTimeLastPage: The part of that same interval during which the app was in the background, or the device was locked or asleep. Kept apart from timeSpentLastPage because a page left open on a locked device during a break otherwise looks identical to a page a worker struggles with. Only timeSpentLastPage indicates that a page is hard to fill in; the two together make up the wall-clock gap.
* serverSessionId: The identifier of the server session the worker signed in to, stamped on every mobile app event. Supply Chain Management records the same identifier, so you can join app events to the corresponding server-side activity and follow one worker's session across both. Use it to keep detailed diagnostics in the server-side telemetry rather than increasing the volume each device sends.

#### Warehouse.MobileApp.OperationFlowTrack (WHS000032)

One row per unit of warehouse work, such as a full put-away or a picking run, rather than per page or per request. A row is emitted when the server confirms the work, and also when the worker gives up on it, so the data is not limited to the flows that succeeded. Because each row covers a complete flow, you can compare the same operation across sites, shifts and devices, and see what a configuration change did to the time the operation takes end to end.

Outcome:
* outcome: `completed` when the server confirmed the work, otherwise `abandoned`.
* status: How the flow ended: `success`, `cancelled` (the worker backed out), `signedOut` (the worker signed out mid-operation) or `sessionLost`.
* durationInMilliseconds: Wall-clock time of the whole operation.
* firstPagePattern / lastPagePattern: The page patterns the flow started and ended on, which identify which operation this was.
* wasInDetourSession: Whether the flow ran inside a nested sub-operation.

Requests. The sums are reported next to the averages they produce because an average cannot be re-aggregated: the mean of per-operation means is not the mean over operations, so any downstream grouping needs the totals.
* requestCount: Requests the app sent during the flow.
* requestFailureCount: Transport failures such as time-outs and 5xx responses. Distinct from errorMessageCount, which counts errors the server deliberately returned.
* totalRequestDurationInMilliseconds / averageRequestDurationInMilliseconds: Time spent waiting on those requests.

Pages:
* pageCount: Pages the worker moved through.
* totalActiveFillDurationInMilliseconds / averageActiveFillDurationInMilliseconds: Time the worker spent actually filling in pages. Excludes waiting on the server, idle time and time on app-local screens, so it isolates the part of the flow the worker controls.

Where the rest of the time went:
* totalIdleDurationInMilliseconds: Time the app was in the background or the device was locked. Recorded but never counted as fill time.
* totalChromeDurationInMilliseconds: Time on app-local screens such as settings, about, diagnostics and connection, which are not part of the operation.
* totalWaitDurationInMilliseconds: Time waiting on in-flight requests.

Worker actions:
* buttonClickCount: Buttons the worker pressed.
* submitCount / cancelCount / detourCount: Presses split by what the button did, so progress can be told apart from backing out and from entering a sub-operation.

Dialog boxes the worker had to open to enter or edit a value. These are split by kind because opening the calculator six times and opening the edit dialog once describe very different sessions:
* modalOpenCount: Total dialog boxes opened. Only the outermost one is timed, so nesting is not counted twice.
* totalModalOpenDurationInMilliseconds: Time those dialog boxes were open.
* modalCalculatorCount / modalTextEditCount / modalControlEditCount / modalOtherCount: The same total, split by kind.

Server messages during the flow:
* errorMessageCount: Error messages the server returned. An error does not end the operation; the worker stays on the same page and tries again.
* warningMessageCount: Warning messages the server returned.

How the worker entered data:
* scanIntentCount / scanKeyboardCount / scanBigBlockCount / scanCharByCharCount: Values entered, split by how they reached the app.
* intentProGloveCount / intentZebraCount: Scans attributed to a specific dedicated scanner, such as a ring scanner.
* cameraOpenCount / cameraOpenDurationInMilliseconds: How often the camera scanner was opened and how long it was open. The camera is the only input source with a real open and close boundary, so the others are reported as counts rather than durations. Compare these to see how much of a flow is spent scanning with the camera rather than with a dedicated scanner.

