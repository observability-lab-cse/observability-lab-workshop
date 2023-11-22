# Advanced Observability Instrumentation

> Goal: Gain insights on how to add custom metrics and traces or other specific configurations.
<!-- 
1. How to add custom instrumentation
2. How to add custom traces for distributed tracing -->

## Custom metrics

Custom metrics, also called user-defined metrics or application-specific metrics, allow us to define and collect information about our system that the standard built-in metrics cannot track.
This will typically be metrics related to the business logic of our application, allowing us to measure the impact of events happening in the system on the user experience or the business.

### Adding custom metrics

The basic observability instrumentation with OTel collector gives us a good starting point to add custom metrics to our application in just a few steps.

Given our business logic, where devices periodically send temperature measurements, it would be useful to know how many device updates occurred in a specified time window. This will be our device update counter metric which we will define in the `devices-state-manager` application which handles all device state and temperature updates.
To add this metric to our application, let's follow the following steps:

1. First, we need to add a new .NET package to the `devices-state-manager` application: `System.Diagnostics.DiagnosticSource`.
2. In the application code, we need to create an instance of the [`Meter`](https://learn.microsoft.com/en-us/dotnet/api/system.diagnostics.metrics.meter?view=net-8.0) class, which will be responsible for creating and tracking metrics.

<details markdown="1">
<summary>Click here to see how to create the meter.</summary>

```csharp
class EventHubReceiverService: IHostedService
{
    private readonly Meter _meter;
    // other dependencies
    // ...

    public EventHubReceiverService(
        string? storageConnectionString,
        string? blobContainerName,
        string? eventHubsConnectionString,
        string? eventHubName,
        string? consumerGroup,
        string? baseUrl,
        ILogger<EventHubReceiverService> logger)
    {
        // Set up other dependencies
        // ...
        _meter = new Meter("DevicesStateManager");
    }
}
```

</details>

3. Now we can define our custom metric. It will track processed device state updates, so let's give it a meaningful name and description. We will use the [`Counter`](https://learn.microsoft.com/en-us/dotnet/api/system.diagnostics.metrics.counter-1?view=net-7.0) class which is useful for tracking the total number of requests, events, etc.

<details markdown="1">
<summary>Click here to see how to define the metric.</summary>

```csharp
    _deviceUpdateCounter = _meter.CreateCounter<int>("device-updates", description: "Number of successful device state updates");
```

</details>

4. Now that we have our metric defined, let's start tracking the device updates. Add code to incrent the counter every time a device temperature and state is updated successfully. You can optionally set tags when the counter is updated. Try to add a tag containing the device ID.

<details markdown="1">
<summary>Click here to see how to increment the counter.</summary>

```csharp
    _deviceUpdateCounter.Add(1, new KeyValuePair<string, object?>("deviceId", deviceId));
```

</details>

Well done! You defined the first custom metric for our application. Now, it would be useful to also be able to track failed device updates. Try to add another metric to track these events.

5. Finally, we need to register our Meter with the previously added OTel instrumentation, by setting an additional environment variable for the `devices-state-manager` container.
Set the `OTEL_DOTNET_AUTO_METRICS_ADDITIONAL_SOURCES` environment variable with the name of the Meter created in step 2. You can do this by adding the variable to the k8s deployment manifest of `devices-state-manager`.

<details markdown="1">
<summary>Click here to see the snippet of the k8s deployment manifest.</summary>

```yaml
- name: OTEL_DOTNET_AUTO_METRICS_ADDITIONAL_SOURCES
  value: "<meter-name>"
```

</details>

6. Redeploy the `devices-state-manager` and wait until there are a few device temperature updates.
7. Go to Application Insights and select the 'Metrics' section. You should be able to find the `device-updates` in the Metric drop-down list after selecting the `azure.applicationinsights` metric namespace. You can adjust the aggregation and time span.

<details markdown="1">
<summary>Click here to see the metric graph in Application Insights.</summary>

![Device updates](./custom-metrics1.png)

</details>

