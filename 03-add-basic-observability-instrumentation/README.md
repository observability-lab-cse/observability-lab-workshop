# Adding Basic Observability Instrumentation

>🎯 **Goal:** Exploring different options to instrument applications with standard tooling and easily integrate them into Azure for data collection and observability.

> **📌 Starting point 📌**
>
> Check out this branch [/02-deploy-application](https://github.com/observability-lab-cse/observability-lab/tree/section/02-deploy-application), in case you have not done the previous section, and run `make` from the root folder.

Now, let's dive into the exciting part of this workshop.

The goal of this section is to instrument the application we developed earlier to gain more visibility into its availability and health.

There are a few steps we need to take to achieve this:

1. Provision necessary resources.
2. Set up a mechanism to send application level telemetry data to these created resources.

There are multiple ways to instrument your application, or even whole solution in more general terms.
Our application is running in AKS, which already provides workload level visibility regarding the health of the resources. However, what we're missing, as previously mentioned, are application-level insights.

When we look at the AKS workloads, pods, or other data tables, it doesn't provide insights into how these pods communicate with each other, or with other services, and whether their communication is successful.

To gain this lower level of visibility into our solution, we have various tools at our disposal. To demonstrate how easily you can instrument an existing application, we'll explore an auto-instrumentation approach using OpenTelemetry's automated instrumentation agents and the OpenTelemetry collector to send telemetry data upstream. 😉

> Note: The [section below](TODO) offers a comparison of different approaches for instrumenting applications, along with links to relevant resources and samples.

But first, let's provision our resources so that we have a destination to send the newly gathered data.

## Provision Resources

Can you guess the first resource we need for this section? If you said Application Insights, then ding, ding, ding... 100 points to you!

As expected, we require Application Insights. However, we'll also need a Log Analytics workspace to support our Application Insights instance.

> If you're curious about how Log Analytics and Application Insights work together, explore this section [Azure 101](tbd) for more information.

- First, let's create a Log Analytics workspace in our resource group. Feel free to use either the Azure portal or any other tools for this task.

- Afterwards, you can create an Application Insights resource, and make sure to specifically select the just-created Log Analytics workspace as the backing instance for the new Application Insights resource.

## OpenTelemetry

As mentioned, there are many ways to instrument applications. Some require writing more custom code, some less.

For this workshop, let's use an approach that requires no changes to the application code, as it will come in handy when working with preexisting applications.
To do so, we will make use of the OpenTelemetry Auto instrumentation for different programming languages and the [OpenTelemetry Collector](https://opentelemetry.io/docs/collector/).

### Configure Applications

For auto-instrumentation, our applications need a small "agent" running alongside
the service to be able to gather telemetry data from common logging, metric, and tracing libraries.

Let's have a look at the applications we would like to instrument.

- [devices-state-manager](TODO) is written in C#. Following the instructions on [OpenTelemetry Auto-instrumentation for C#](https://opentelemetry.io/docs/instrumentation/net/automatic/), you can auto-instrument the plain vanilla version of the Docker file in such a way that we can scrape the logs and send them to the OpenTelemetry collector.
- [devices-api](TODO) is a Java application. The same instructions for Java services can be found here [OpenTelemetry Auto-instrumentation for Java](https://github.com/open-telemetry/opentelemetry-java-instrumentation). Like before, try adding the auto-instrumentation Agent into the build so we can expose all the metrics.

In case you have issues, you can see how to do it when opening the section below or check out the next section's [branch]().

<details markdown="1">
<summary>Click here for the Dockerfile with the auto-instrumentation Agent for Java</summary>

```Docker
FROM eclipse-temurin:17

RUN mkdir /app
COPY . /app
WORKDIR /app

# Download opentelemetry-javaagent.jar
RUN curl -L -O https://github.com/open-telemetry/opentelemetry-java-instrumentation/releases/latest/download/opentelemetry-javaagent.jar

RUN ./gradlew build

ENTRYPOINT ["java", "-javaagent:opentelemetry-javaagent.jar", "-jar","build/libs/devices-api.jar"]
```

</details>

The same again for the C# service.

<details markdown="1">
<summary>Click here for the Dockerfile with the auto-instrumentation Agent for C#</summary>

```Docker
FROM mcr.microsoft.com/dotnet/sdk:6.0 AS build
WORKDIR /src
COPY ["DeviceManager.csproj", "."]
RUN dotnet restore "./DeviceManager.csproj"
COPY . .
WORKDIR "/src/."
RUN dotnet build "DeviceManager.csproj" -c Release -o /app/build

FROM build AS publish
RUN dotnet publish "DeviceManager.csproj" -c Release -o /app/publish /p:UseAppHost=false

FROM base AS final
WORKDIR /app
COPY --from=publish /app/publish .

ARG OTEL_VERSION=1.0.0-rc.2
ADD https://github.com/open-telemetry/opentelemetry-dotnet-instrumentation/releases/download/v${OTEL_VERSION}/otel-dotnet-auto-install.sh otel-dotnet-auto-install.sh
RUN apt-get update && apt-get install -y unzip && apt-get install -y curl && \
    OTEL_DOTNET_AUTO_HOME="/otel-dotnet-auto" sh otel-dotnet-auto-install.sh

ENTRYPOINT ["dotnet", "DeviceManager.dll"]
```

</details>

Now, let's build the images with the agent and add a new tag to it, so that we know which images are auto-instrumented. TODO

### Configure Deployment

For the agent to properly connect to the Collector, there are a few environment variables that need to be set. For now, let's only set the mandatory ones.

In the case of the Java agent, not much is needed.

<details markdown="1">
<summary>Click here for the new deployment yaml with the env vars for auto-instrumentation Agent for Java</summary>

There are two new variables `OTEL_EXPORTER_OTLP_ENDPOINT` and `OTEL_LOGS_EXPORTER`.

```yaml
kind: Deployment
apiVersion: apps/v1

metadata:
  name: devices-api

spec:
  replicas: 1
  selector:
    matchLabels:
      app: devices-api
  template:
    metadata:
      labels:
        app: devices-api
    spec:
      containers:
        - name: devices-api
          image: acr${project-name}.azurecr.io/devices-api:latest TODO: Tags
          imagePullPolicy: Always
          ports:
            - containerPort: 8080
          resources:
            requests:
              cpu: 50m
              memory: 128Mi
            limits:
              cpu: 150m
              memory: 512Mi
          volumeMounts:
            - name: secrets-store-inline
              mountPath: "/mnt/secrets-store"
              readOnly: true
          env:
            - name: OTEL_EXPORTER_OTLP_ENDPOINT
              value: "http://opentelemetrycollector:4317"
            - name: OTEL_LOGS_EXPORTER
              value: otlp
            - name: OTEL_SERVICE_NAME
              valueFrom:
                fieldRef:
                  apiVersion: v1
                  fieldPath: metadata.labels['app']
            - name: AZURE_COSMOS_DB_URI
              valueFrom:
                secretKeyRef:
                  name: devices-api-secrets
                  key: CosmosDBEndpoint
            - name: AZURE_COSMOS_DB_KEY
              valueFrom:
                secretKeyRef:
                  name: devices-api-secrets
                  key: CosmosDBKey
            - name: AZURE_COSMOS_DB_NAME
              valueFrom:
                secretKeyRef:
                  name: devices-api-secrets
                  key: CosmosDBName
          readinessProbe:
            httpGet:
              path: /health
              port: 8080
            periodSeconds: 20
            initialDelaySeconds: 20
            failureThreshold: 15
      volumes:
        - name: secrets-store-inline
          csi:
            driver: secrets-store.csi.k8s.io
            readOnly: true
            volumeAttributes:
              secretProviderClass: "kvprovider"
```

</details>

For the C# application there are a few more variables needed

<details markdown="1">
<summary>Click here for the new deployment yaml with the env vars for autoinstrumentation Agent for C#</summary>

There are two new variables `OTEL_EXPORTER_OTLP_ENDPOINT` and `OTEL_LOGS_EXPORTER`.

```yaml
kind: Deployment
apiVersion: apps/v1

metadata:
  name: device-manager

spec:
  replicas: 1
  selector:
    matchLabels:
      app: device-manager
  template:
    metadata:
      labels:
        app: device-manager
    spec:
      containers:
        - name: device-manager
          image: acr${project-name}.azurecr.io/device-manager:latest TODO: Tags
          imagePullPolicy: Always
          ports:
            - containerPort: 8090
          resources:
            requests:
              cpu: 50m
              memory: 128Mi
            limits:
              cpu: 150m
              memory: 512Mi
          env:
            - name: OTEL_EXPORTER_OTLP_ENDPOINT
              value: "http://opentelemetrycollector:4318"
            - name: CORECLR_ENABLE_PROFILING
              value: "1"
            - name: CORECLR_PROFILER
              value: "{918728DD-259F-4A6A-AC2B-B85E1B658318}"
            - name: CORECLR_PROFILER_PATH
              value: "/otel-dotnet-auto/linux-x64/OpenTelemetry.AutoInstrumentation.Native.so"
            - name: DOTNET_ADDITIONAL_DEPS
              value: "/otel-dotnet-auto/AdditionalDeps"
            - name: DOTNET_SHARED_STORE
              value: "/otel-dotnet-auto/store"
            - name: DOTNET_STARTUP_HOOKS
              value: "/otel-dotnet-auto/net/OpenTelemetry.AutoInstrumentation.StartupHook.dll"
            - name: OTEL_DOTNET_AUTO_HOME
              value: "/otel-dotnet-auto"
            - name: OTEL_SERVICE_NAME
              valueFrom:
                fieldRef:
                  apiVersion: v1
                  fieldPath: metadata.labels['app']
            - name: EVENT_HUB_CONNECTION_STRING
              value: EVENT_HUB_LISTEN_POLICY_CONNECTION_STRING_PLACEHOLDER
            - name: EVENT_HUB_NAME
              value: EVENT_HUB_NAME_PLACEHOLDER
            - name: STORAGE_CONNECTION_STRING
              value: STORAGE_CONNECTION_STRING_PLACEHOLDER
            - name: BLOB_CONTAINER_NAME
              value: event-hub-data
            - name: DEVICE_API_URL
              value: "http://devices-api-service:8080"
```

</details>

### Deployment Otel collector and auto-instrumented Application

Let's now first deploy the OpenTelemetry collector and then re-deploy the individual applications again.

<details markdown="1">
<summary>Click here for the Opentelemerty Colloector</summary>

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: opentelemetrycollector
spec:
  replicas: 1
  selector:
    matchLabels:
      app.kubernetes.io/name: opentelemetrycollector
  template:
    metadata:
      labels:
        app.kubernetes.io/name: opentelemetrycollector
    spec:
      hostAliases:
      containers:
        - name: otelcol
          args:
            - --config=/conf/collector-config.yaml
          image: otel/opentelemetry-collector-contrib:0.83.0
          volumeMounts:
            - mountPath: /conf
              name: config
          resources:
            requests:
              cpu: "0.2"
              memory: "200Mi"
            limits:
              cpu: "0.3"
              memory: "300Mi"
      volumes:
        - configMap:
            items:
              - key: "collector-config.yaml"
                path: collector-config.yaml
            name: config
          name: config

---
apiVersion: v1
kind: Service
metadata:
  name: opentelemetrycollector
spec:
  ports:
    - name: grpc-otlp
      port: 4317
      targetPort: 4317
      protocol: TCP
    - name: http-otlp
      port: 4318
      targetPort: 4318
      protocol: TCP
  selector:
    app.kubernetes.io/name: opentelemetrycollector
  type: ClusterIP
```

As well as the `collector-config.yaml`

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: config
data:
  collector-config.yaml: |
    receivers:
      otlp:
        protocols:
          grpc:
          http:
    processors:
      batch:
    exporters:
      azuremonitor:
        instrumentation_key: INSTRUMENTATION_KEY_PLACEHOLDER
      logging:
        verbosity: normal
    service:
      pipelines:
        traces:
          receivers: [otlp]
          processors: []
          exporters: [azuremonitor, logging]
        metrics:
          receivers: [otlp]
          processors: []
          exporters: [azuremonitor, logging]
        logs:
          receivers: [otlp]
          processors: []
          exporters: [azuremonitor, logging]
```

</details>

## Navigation

[Previous Section ⏪](../02-deploy-application/README.md) ‖ [Return to Main Index 🏠](../README.md) ‖
[Next Section ⏩️](../04-vizualisation/README.md)
