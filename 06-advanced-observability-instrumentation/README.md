# Advanced Observability Instrumentation

> Goal: Gain insights on how to add custom metrics and traces or other specific configurations.
<!-- 
1. How to add custom instrumentation
2. How to add custom traces for distributed tracing -->

## Custom metrics

Custom metrics, also called user-defined metrics or application-specific metrics, allow us to define and collect information about our system that the standard built-in metrics cannot.
This will typically be metrics related to the business logic of our application, allowing us to measure the impact of events happening in the system on the user experience or the business.

### Adding custom metrics

The basic observability instrumentation with OTel collector gives us a good starting point to easily add custom metrics to our application.

Given our business logic where devices periodically send temperature measurements, it would be useful to know how many device updates occurred in a specified time window. 
To achieve this, let's follow the following steps:

1. We need to add a new .NET package to the `DevicesStateManager` application: `System.Diagnostics.DiagnosticSource`.
2. In the application code, we need to create an instance of the [`Meter`](https://learn.microsoft.com/en-us/dotnet/api/system.diagnostics.metrics.meter?view=net-8.0) class, which will be responsible for creating and tracking metrics.
3. Now we can define our custom metric. It will track successful device state updates, so let's give it a meaningful name and description:

```csharp
    _deviceUpdateCounter = _meter.CreateCounter<int>("device-updates", description: "Number of successful device state updates");
```

We are using the [`Counter`](https://learn.microsoft.com/en-us/dotnet/api/system.diagnostics.metrics.counter-1?view=net-7.0) class which is used for tracking the total number of requests, events, etc.
4. Now that we have our metric defined, let's start tracking the device updates. Every time a device temperature and state is updated successfully, we will want to increment our counter:

```csharp
    _deviceUpdateCounter.Add(1);
```

You can optionally set tags when the counter is updated. Try to add a tag containing the device ID.
5. Finally, we need to register our Meter with the previously added OTel instrumentation, by setting an additional environment variable for the `devices-state-manager` container.
Set the `OTEL_DOTNET_AUTO_METRICS_ADDITIONAL_SOURCES` environment variable with the name of the Meter created in step 2.
 You can do this by adding the following snipped to the k8s deployment manifest:

```yaml
- name: OTEL_DOTNET_AUTO_METRICS_ADDITIONAL_SOURCES
  value: "<meter-name>"
```

6. Redeploy the `devices-state-manager` and wait until there are a few device temperature updates.
7. Go to Application Insights and select 'Metrics'. You should be able to find the `device-updates` in the Metric drop-down list after selecting the `azure.applicationinsights` metric namespace.
