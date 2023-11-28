# 3. Adding Basic Observability Instrumentation

>🎯 **Goal:** Exploring different options to instrument applications with standard tooling and easily integrate them into Azure for data collection and observability.

Now, let's dive into the exciting part of this workshop.

The goal of this section is to instrument the application we developed earlier to gain more visibility into its availability and health.

There are a few steps we need to take to achieve this:

1. ⚙️ Provision necessary resources.
2. 🛰️ Set up a mechanism to send application level telemetry data to these created resources.

There are multiple ways to instrument your application, or even whole solution in more general terms.
Our application is running in AKS, which already provides workload level visibility regarding the health of the resources. However, what we're missing, as previously mentioned, are application-level insights.

When we look at the AKS workloads, pods, or other data tables, it doesn't provide insights into how these pods communicate with each other, or with other services, and whether their communication is successful.

To gain this lower level of visibility into our solution, we have various tools at our disposal. To demonstrate how easily you can instrument an existing application, we'll explore an auto-instrumentation approach using OpenTelemetry's automated instrumentation agents and the OpenTelemetry collector to send telemetry data upstream. 😉

> 📝 **Note:**  This article ["Cluster observability"](https://internal.playbook.microsoft.com/code-with-platformops/capabilities/observability/k8s-observability/?h=instrumentation+based#individual-instrumentation-based-approaches) on the PlatformOps Playbook offers a comparison of a few other  approaches for instrumenting applications, along with links to relevant resources and samples.

But first, let's provision our resources so that we have a destination to send the newly gathered data.

## ⚙️ Provision Resources

> **📌 Starting point 📌**
>
> In case you have not completed the previous section, check out this branch: [section/02-deploy-application](https://github.com/observability-lab-cse/observability-lab/tree/section/02-deploy-application), and run `make` from the root folder.

Can you guess the first resource we need for this section? If you said Application Insights, then 🛎️ ding, ding, ding... 💯 points to you!

As expected, we require Application Insights. However, we'll also need a Log Analytics workspace to support our Application Insights instance.

1. First, let's create a Log Analytics workspace in our resource group. Feel free to use either the Azure portal or any other tools for this task.
1. Afterwards, you can create an Application Insights resource, and make sure to specifically select the just-created Log Analytics workspace as the backing instance for the new Application Insights resource.

## 🛰️ OpenTelemetry

As mentioned, there are many ways to instrument applications. Some require writing more custom code, some less.

For this workshop, let's use an approach that requires no changes to the application code, as it will come in handy when working with preexisting applications.
To do so, we will make use of the OpenTelemetry Auto instrumentation for different programming languages and the [OpenTelemetry Collector](https://opentelemetry.io/docs/collector/).

This will require our application to publish their telemetry data via [otlp protocol](https://opentelemetry.io/docs/specs/otel/protocol/) (or any other of the available [receivers](https://opentelemetry.io/docs/collector/configuration/#receivers)) to the collector, which can than send them upstream into one or more of its [exporters](https://opentelemetry.io/docs/collector/configuration/#exporters). The collector allows you to do much more than just forwards telemetry data from your application to Azure for example. Using [processors](https://opentelemetry.io/docs/collector/configuration/#processors) and [connectors](https://opentelemetry.io/docs/collector/configuration/#connectors), there is a lot of preprocessing you can do before sending your data to is end location.
However this is a conversation for another time or come back later when this workshop has extended to also cover these topics 😉.

> 📝 **Note:** One final note on this topic however, for people that wonder why you would want to pre process your data. So in scenarios when your cluster is not an AKS cluster but running on an edge device which low connectivity or if your applications produce a lot of telemetry your would like to filter/control before being send upstream these features of the collector come in very handy. SO if you are interested go check out the [opentelemetry-collector](https://github.com/open-telemetry/opentelemetry-collector/tree/main) and [opentelemetry-collector-contrib](https://github.com/open-telemetry/opentelemetry-collector-contrib/tree/main) for all the options and tools you have at your disposal.

Now that we know how we want to instrument our solution, lets start by auto-instrumenting our applications and then deploying the new version along side an OpenTelemetry Collector

### 📱 Configure Applications

For auto-instrumentation, our applications needs to get the OpenTelemetry SDK injected and configured, so it is able to gather telemetry data from common logging, metric, and tracing libraries.

Let's have a look at the applications we would like to instrument.

- The [devices-state-manager](https://github.com/observability-lab-cse/observability-lab/tree/section/03-add-basic-observability-instrumentation/sample-application/devices-state-manager) is written in C#. Following the instructions on [OpenTelemetry Auto-instrumentation for C#](https://opentelemetry.io/docs/instrumentation/net/automatic/), you can auto-instrument the plain vanilla version of the Docker file in such a way that we can scrape the logs and send them to the OpenTelemetry collector.
- The [devices-api](https://github.com/observability-lab-cse/observability-lab/tree/section/03-add-basic-observability-instrumentation/sample-application/devices-api) is a Java application. The same instructions for Java services can be found here [OpenTelemetry Auto-instrumentation for Java](https://github.com/open-telemetry/opentelemetry-java-instrumentation). Like before, try adding the auto-instrumentation Agent JAR into the build, so it can  attach to your application and dynamically inject bytecode to capture telemetry data.

In case you have issues, you can see how to do it when opening the section below or check out the next section's branch [section/04-visualization](TODO).

<details markdown="1">
<summary> 🔦 Dockerfile with the auto-instrumentation for Java</summary>

In the case of our Java application, we need to download the OpenTelemetry Agent Jar and load it along side our application at start up.

> 📝 **Note:** Take a note on which environment variables need to be set for the Java Agent to inject the correct bytecode so it collects all the telemetry we need.

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

<details markdown="1">
<summary>🔦 Dockerfile with the auto-instrumentation for C#</summary>

For our C# service the auto-instrumentation look slightly different.
Here we need to download a `bash` script that will install and inject the [OpenTelemetry .NET SDK](https://github.com/open-telemetry/opentelemetry-dotnet/blob/main/src/OpenTelemetry/README.md#opentelemetry-net-sdk) into our application, using the startup hook.
> 📝 **Note:** Take a note on which environment variables need to be set for the scripts to inject the libraries so it collects all the telemetry we need.

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

Now, let's build the images with the auto-instrumentation like in our previous chapter. In case you don't remember, go back to this section [🚀  Deploy Application](../02-deploy-application/README.md#🚀-deploy-application)

### 🎻 Deployment OpenTelemetry collector

Before we redeploy our newly auto-instrumented applications, lets have a look at the OpenTelemetry Collector. Especially how we need to deployment and configure it so our data gets send to our Application Insights.

Let see if, given the documentation ["Install Collector"](https://opentelemetry.io/docs/collector/installation/) on how to deploy the collector, you can manage to create its deployment and configure it correctly. If you don't feel very comfortable yet with this, open the `🔍 Hints` section below and get some more guidance how to approach this task 😉

<details markdown="1">
<summary> 🔍 Hints: </summary>
Start with creating your collector deployment and then proceed with configuring the collector as you need using the `collector-config.yaml`
A few question that could guide you to figure out what you need to configure:

1. What is our upstream and downstream sources?
1. What receivers and exporters do we need to configure?
1. What what type of data do we want to collect?

</details>

<details markdown="1">
<summary> 🛠️ Step-by-Step guide on deploying the OpenTelemetry Collector</summary>

1. So first of all we can focus on the deployment
TODO

2. Now lets have a look at the configuration.

    i. As mentioned before, the auto-instrumentation exposes the telemetry data using the otlp protocol. Now according to this docs on [receivers](https://opentelemetry.io/docs/collector/configuration/#receivers) our `receiver` section needs to contain the following

      ```yaml
        otlp:
          protocols:
            grpc:
            http:
      ```

    ii. Given we don't really want to process the data withing the collector, we can skip all the other sections and directly go to the [exporter](https://opentelemetry.io/docs/collector/configuration/#exporters). Interestingly enough we wont find anything regarding Azure in this docs 🤔. As mention above, there are two repositories when it comes to the `exporter`,`receivers`, etc. the [opentelemetry-collector](https://github.com/open-telemetry/opentelemetry-collector/tree/main) and the [opentelemetry-collector-contrib](https://github.com/open-telemetry/opentelemetry-collector-contrib/tree/main). If we look in the `contrib` repository, we can find the [`azuremonitorexporter`](https://github.com/open-telemetry/opentelemetry-collector-contrib/tree/main/exporter/azuremonitorexporter). So this now means we need the following section in the `exporter` configuration:

    ```yaml
    azuremonitor:
        instrumentation_key: INSTRUMENTATION_KEY_PLACEHOLDER
    ```

    To be able to debug the deployment and see if the metric is also flowing through the collector, lets also set the log level of the collector to `normal` by adding this below snippet to the `exporter` configuration:

      ```yaml
      logging:
              verbosity: normal
      ```

    iii. Finally, we need to tell the Collector what to do with the data it ingest. This is done using the [service.pipelines](https://opentelemetry.io/docs/collector/configuration/#service) section of the configuration yaml. For now we want all our telemetry data to be ingested form the `otlp` source (also because we don't have another any other configure 😆), not processed and the forwarded to our two exporters. This is how that would look for each metric.

    ```yaml
          receivers: [otlp]
          processors: []
          exporters: [azuremonitor, logging]
    ```

    Add this in for all telemetry data we need and you have your `collector-config.yaml` 🥳.

</details>

If you managed, great 🎉! Else, not to worry. Open the section below and you have the deployment yaml as well as the configuration for your collector. Go ahead and deploy that, before you redeploy your applications.

> 📝 **Note:** Don't forget to replace the `INSTRUMENTATION_KEY_PLACEHOLDER` with your Application Insights instrumentation key. 

<details markdown="1">
<summary>🔦 Deployment yaml OpenTelemetry Collector</summary>

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

### 🐳 Configure Deployment

For the service to properly connect to the Collector, there are a few environment variables that need to be set. For now, let's only set the mandatory ones.

In the case of the Java agent, not much is needed.

<details markdown="1">
<summary>🔦 Deployment yaml for auto-instrumentation <code>devices-api</code> service</summary>

TODO: There are two new variables `OTEL_EXPORTER_OTLP_ENDPOINT` and `OTEL_LOGS_EXPORTER`.

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
<summary>🔦 Deployment yaml for auto-instrumentation <code>device-state-manager</code> service</summary>

TODO

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

Go grab another coffee ☕ (or tea 🍵, we don't discriminate) and come back to a bunch of telemetry send already upstream! To learn how to use all this cool new data, head to the next section where we see how cool Application Insights actually is 😜.

## Navigation

[Previous Section ⏪](../02-deploy-application/README.md) ‖ [Return to Main Index 🏠](../README.md) ‖
[Next Section ⏩️](../04-visualization/README.md)
