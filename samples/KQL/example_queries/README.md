## Warehoouse Mobile App

Having a superior Wi-Fi network results in shipping on time. You can use the WMA telemetry to monitor and improve the Wi-Fi coverage in your warehouse.

The telemetry includes the following data points:
* roundTripLatencyDurationInMilliseconds : The total time the device is awaiting a response, including network latency and backend processing.
* backendProcessingTime: The time (in milliseconds) the backend spent processing the request. The time is measured by the IIS, so it includes literally everything happening at the backend.
* renderingDurationInMilliseconds: The time for the device to render the page. This depends on the complexity of the page to render (how many controls), and the GPU on the device.
* wifiSignalStrength: The current measured strength of the Wi-Fi signal.
* lastKnownWMSLocation: The last known location of the device, typically determined by scan of a location or license plate during a pick or put operation. This can be used with the wifiSignalStrength to create a coverage map of Wi-Fi strength across the locations.

![image](https://github.com/microsoft/d365-scm-telemetry/assets/28089437/3158400c-cf26-4d7b-837c-bc4a5f7cfe2c)
