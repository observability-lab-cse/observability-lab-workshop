# 3. Adding Basic Observability Instrumentation

>🎯 **Goal:** Explore different options to instrument applications with standard tooling and easily integrate them into Azure for data collection and observability.

Now, let's dive into the exciting part of this workshop.

The goal of this section is to instrument the application we developed earlier to gain more visibility into its availability and health.

There are a few steps we need to take to achieve this:

1. ⚙️ Provision necessary resources.
2. 🛰️ Set up a mechanism to send application level telemetry data to these created resources.

There are multiple ways to instrument your application, or even whole solution in more general terms.
Our application is running in AKS, which already provides workload level visibility regarding the health of the resources. However, what we're missing, as previously mentioned, are application-level insights.

When we look at the AKS workloads, like pods, it doesn't provide insights into how these pods communicate with each other, or with other services, and whether their communication is successful.

To gain this lower level of visibility into our solution, we have various tools at our disposal. To demonstrate how easily you can instrument an existing application, we'll use the OpenTelemetry auto-instrumentation approach and the OpenTelemetry collector to send telemetry data upstream. 😉

> 📝 **Note:**  This article ["Cluster observability"](https://internal.playbook.microsoft.com/code-with-platformops/capabilities/observability/k8s-observability/?h=instrumentation+based#individual-instrumentation-based-approaches) on the PlatformOps Playbook offers a comparison of a few other  approaches for instrumenting applications, along with links to relevant resources and samples.

But first, let's provision our resources so that we have a destination to send the newly gathered data.

## ⚙️ Provision Resources

> **📌 Starting point 📌**
>
> In case you have not completed the previous section:
> - Check out this branch: [section/03-add-basic-observability-instrumentation](https://github.com/observability-lab-cse/observability-lab/tree/section/03-add-basic-observability-instrumentation)
> - Copy .env.example file into .env and update the file with your values
> - Run `make` from the root folder.

Can you guess the first resource we need for this section? If you said Application Insights, then 🛎️ ding, ding, ding... 💯 points to you!

As expected, we require Application Insights. However, we'll also need a Log Analytics workspace to support our Application Insights instance.

1. First, let's create a Log Analytics workspace in our resource group. Feel free to use either the Azure portal or any other tools for this task.
1. Afterwards, you can create an Application Insights resource, and make sure to specifically select the just-created Log Analytics workspace as the backing instance for the new Application Insights resource.

## 🛰️ OpenTelemetry

As mentioned, there are many ways to instrument applications. Some require writing more custom code, some less.

For this workshop, let's use an approach that requires no changes to the application code, as it will come in handy when working with preexisting applications.
To do so, we will make use of the [OpenTelemetry Auto instrumentation](https://opentelemetry.io/docs/concepts/instrumentation/automatic/) for different programming languages and the [OpenTelemetry Collector](https://opentelemetry.io/docs/collector/).

This will require our application to publish their telemetry data via [otlp protocol](https://opentelemetry.io/docs/specs/otel/protocol/) (or any other of the available [receivers](https://opentelemetry.io/docs/collector/configuration/#receivers)) to the collector, which can than send them upstream into one or more of its [exporters](https://opentelemetry.io/docs/collector/configuration/#exporters). The collector allows you to do much more than just forward telemetry data from your application to Azure. Using [processors](https://opentelemetry.io/docs/collector/configuration/#processors) and [connectors](https://opentelemetry.io/docs/collector/configuration/#connectors),
there is a lot of preprocessing you can do before sending your data to its end location.
However, this is a conversation for another time or come back later when this workshop has extended to also cover these topics 😉.

> 📝 **One final note on this topic if you wonder why you would want to preprocess your data.**
> These features of the collector come in very handy especially in scenarios when your cluster is not an AKS cluster but running on an edge device with low connectivity or
> if your applications produce a lot of telemetry you need to filter/control before being sent upstream.
> If you are interested, go check out the [opentelemetry-collector](https://github.com/open-telemetry/opentelemetry-collector/tree/main) and [opentelemetry-collector-contrib](https://github.com/open-telemetry/opentelemetry-collector-contrib/tree/main) for all the options and tools you have at your disposal.

Now that we know how we want to instrument our solution, lets start by auto-instrumenting our applications and then deploying the new version alongside an OpenTelemetry Collector.

### 📱 Configure Applications

For auto-instrumentation, our applications need to get the OpenTelemetry SDK injected and configured, so it is able to gather telemetry data from common logging, metric, and tracing libraries.

Let's have a look at the applications we would like to instrument.

- The [devices-state-manager](https://github.com/observability-lab-cse/observability-lab/tree/section/03-add-basic-observability-instrumentation/sample-application/devices-state-manager) is a .NET application.
- The [devices-api](https://github.com/observability-lab-cse/observability-lab/tree/section/03-add-basic-observability-instrumentation/sample-application/devices-api) is a Java application.

Let's be honest and agree that the OpenTelemetry documentation is not the easiest to work with or find stuff in. So, to give you some help, we have listed the steps below on what you need to do and where to grab the information from.

For both applications we will add auto-instrumentation through `Dockerfile`.

#### Add Java auto-instrumentation to the Devices API

OpenTelemetry provides an instrumentation Java agent JAR that can be attached to your Java 8+ application. This agent dynamically injects bytecode to capture telemetry from a number of popular libraries and frameworks.

To set it up, you’ll need to download the agent and enable it in the Devices API `Dockerfile`. Detailed instructions can be found on GitHub: [OpenTelemetry Auto-instrumentation for Java](https://github.com/open-telemetry/opentelemetry-java-instrumentation#getting-started).

Here’s how to modify your Devices API `Dockerfile`:

* Download the Instrumentation Agent. We recommend using version [v1.31.0](https://github.com/open-telemetry/opentelemetry-java-instrumentation/releases/tag/v1.31.0), which is the latest version we’ve tested.

  <details markdown="1">
  <summary> 🔍 Hint: step for downloading the instrumentation agent </summary>

  ```Docker
  RUN curl -L -O https://github.com/open-telemetry/opentelemetry-java-instrumentation/releases/download/v1.31.0/opentelemetry-javaagent.jar
  ```
  </details>

* Enable the instrumentation agent using the `-javaagent` flag to the JVM. 

  <details markdown="1">
  <summary> 🔍 Hint: update to entrypoint for enabling the instrumentation agent </summary>

  ```Docker
  ENTRYPOINT ["java", "-javaagent:opentelemetry-javaagent.jar", "-jar","build/libs/devices-api.jar"]
  ```
  </details>

> 📝 **Note:** Take a note on which environment variables need to be set for injected OpenTelemetry SDK to correctly collect all the telemetry we need.

#### Add .NET auto-instrumentation to the Devices State Manager

OpenTelemetry .NET Automatic Instrumentation injects and configures the OpenTelemetry .NET SDK into the application and adds OpenTelemetry Instrumentation to key packages and APIs used by the application.

How to set it up? There are various options, we’ll focus on modifications within the `Dockerfile` for consistency reasons. An installer script is available for download and can be run as part of the `Dockerfile`. Detailed instructions can be found on GitHub: [OpenTelemetry .NET Automatic Instrumentation](https://github.com/open-telemetry/opentelemetry-dotnet-instrumentation#get-started). Pay special attention to the [Instrument a container section](https://github.com/open-telemetry/opentelemetry-dotnet-instrumentation#instrument-a-container), where you’ll find an example `Dockerfile`.

Here’s what you need to do in your Devices State Manager `Dockerfile`:

* Download and run the installer script. We recommend using version [v1.1.0](https://github.com/open-telemetry/opentelemetry-dotnet-instrumentation/releases/tag/v1.1.0), which is the latest version we’ve tested.

  <details markdown="1">
  <summary> 🔍 Hint: step for downloading and installing the script </summary>

  ```Docker
  ARG OTEL_VERSION=1.1.0
  ADD https://github.com/open-telemetry/opentelemetry-dotnet-instrumentation/releases/download/v${OTEL_VERSION}/otel-dotnet-auto-install.sh otel-dotnet-auto-install.sh
  RUN apt-get update && apt-get install -y unzip && apt-get install -y curl && \
      OTEL_DOTNET_AUTO_HOME="/otel-dotnet-auto" sh otel-dotnet-auto-install.sh
  ```

  </details>

> 📝 **Note:** Take a note on which environment variables need to be set for the scripts to inject the libraries so that it collects all the telemetry we need.

That’s all for the Dockerfile changes! We’ll need to make additional configuration changes later. For now, you’re ready to build images with auto-instrumentation injected. Build them as described in the previous section [🚀  Deploy Application](../02-deploy-application/README.md#🚀-deploy-application), or use the `make push` command from the root folder to build and push the devices-api and devices-state-manager images.

If you’d like to see complete Dockerfiles with auto-instrumentation injected, you can find them here:

<details markdown="1">
<summary> 🔦 Dockerfile with the auto-instrumentation for Java</summary>

```Docker
FROM eclipse-temurin:17

RUN mkdir /app
COPY . /app
WORKDIR /app

RUN ./gradlew build

RUN curl -L -O https://github.com/open-telemetry/opentelemetry-java-instrumentation/releases/download/v1.31.0/opentelemetry-javaagent.jar

ENTRYPOINT ["java", "-javaagent:opentelemetry-javaagent.jar", "-jar","build/libs/devices-api.jar"]
```

</details>

<details markdown="1">
<summary>🔦 Dockerfile with the auto-instrumentation for .NET</summary>

```Docker
FROM mcr.microsoft.com/dotnet/runtime:6.0 AS base
WORKDIR /app

FROM mcr.microsoft.com/dotnet/sdk:6.0 AS build
WORKDIR /src
COPY ["DevicesStateManager.csproj", "."]
RUN dotnet restore "./DevicesStateManager.csproj"
COPY . .
WORKDIR "/src/."
RUN dotnet build "DevicesStateManager.csproj" -c Release -o /app/build

FROM build AS publish
RUN dotnet publish "DevicesStateManager.csproj" -c Release -o /app/publish /p:UseAppHost=false

FROM base AS final
WORKDIR /app
COPY --from=publish /app/publish .

ARG OTEL_VERSION=1.1.0
ADD https://github.com/open-telemetry/opentelemetry-dotnet-instrumentation/releases/download/v${OTEL_VERSION}/otel-dotnet-auto-install.sh otel-dotnet-auto-install.sh
RUN apt-get update && apt-get install -y unzip && apt-get install -y curl && \
    OTEL_DOTNET_AUTO_HOME="/otel-dotnet-auto" sh otel-dotnet-auto-install.sh

ENTRYPOINT ["dotnet", "DevicesStateManager.dll"]
```

</details>


### 🎻 Deploy OpenTelemetry collector

Let’s now focus on the deployment and configuration of the OpenTelemetry Collector. This will allow our telemetry data to be received and subsequently exported to our Application Insights.

There are several methods to deploy the collector to a Kubernetes cluster, such as using the OpenTelemetry Helm chart or the OpenTelemetry Operator. However, to thoroughly understand the collector configuration, we will deploy the collector using a simple Kubernetes Deployment with ConfigMap.

Feel free to explore the [Install the Collector](https://opentelemetry.io/docs/collector/installation/#kubernetes) documentation, though below you will find step-by-step guidance on how to approach this task 😉.

#### Collector configuration

Let's start with the configuration. Create a `collector-config.yaml` in `k8s-files` folder with the following content:

  ```yaml
  apiVersion: v1
  kind: ConfigMap
  metadata:
    name: config
  data:
    collector-config.yaml: |
  ```

Next, we need to populate it with a valid OpenTelemetry Collector configuration. Familiarize yourself with [the Collector configuration structure](https://opentelemetry.io/docs/collector/configuration/#basics). There’s an example config available, which serves as an excellent starting point. Copy it and add it into your `collector-config.yaml` file (ensure it’s properly indented).

As previously mentioned, the auto-instrumentation exposes the telemetry data using the OTLP protocol. Thanks to the `otlp` receivers that we just configured, the OpenTelemetry collector will be able to receive telemetry from .NET and Java auto-instrumentations.

Next, let’s focus on exporters. There is one exporter: `otlp` in our config. However, we want to send data to Azure Monitor. So, let’s refer to the documentation: [exporters](https://opentelemetry.io/docs/collector/configuration/#exporters). Interestingly, we won’t find anything regarding Azure in this documentation 🤔. Like already hinted at, there are two repositories when it comes to exporters, receivers, etc.: [opentelemetry-collector](https://github.com/open-telemetry/opentelemetry-collector/tree/main) and [opentelemetry-collector-contrib](https://github.com/open-telemetry/opentelemetry-collector-contrib/tree/main). If we look in the `contrib` repository, we can find the [`azuremonitorexporter`](https://github.com/open-telemetry/opentelemetry-collector-contrib/tree/main/exporter/azuremonitorexporter), which allows to export telemetry to Application Insights. This means we need to replace the `otlp` exporter with `azuremonitor`:

  ```yaml
  azuremonitor:
    instrumentation_key: INSTRUMENTATION_KEY_PLACEHOLDER
  ```
  Make sure to replace the `INSTRUMENTATION_KEY_PLACEHOLDER` with your Application Insights instrumentation key, which can be found on the Application Insights Overview page.

To debug the deployment and see if the telemetry is flowing through the collector, let’s also set up another useful exporter: [Debug Exporter](https://github.com/open-telemetry/opentelemetry-collector/blob/main/exporter/debugexporter/README.md). It exports telemetry to the console output of the collector. Add this snippet to the `exporter` configuration:

  ```yaml
  debug:
    verbosity: detailed
  ```

Finally, we need to instruct the Collector on how to handle the ingested data. This is done using the [service.pipelines](https://opentelemetry.io/docs/collector/configuration/#service) section of the configuration YAML. Update it with your two previously configured exporters. This is how it should look like for each signal.

  ```yaml
  receivers: [otlp]
  processors: [batch]
  exporters: [azuremonitor, debug]
  ```

If you’d like to view the complete `collector-config.yaml`, please click on the link below.

<details markdown="1">
<summary>🔦 OpenTelemetry Collector Configuration YAML</summary>

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
      debug:
        verbosity: detailed
    extensions:
      health_check:
      pprof:
      zpages:

    service:
      extensions: [health_check, pprof, zpages]
      pipelines:
        traces:
          receivers: [otlp]
          processors: [batch]
          exporters: [azuremonitor, debug]
        metrics:
          receivers: [otlp]
          processors: [batch]
          exporters: [azuremonitor, debug]
        logs:
          receivers: [otlp]
          processors: [batch]
          exporters: [azuremonitor, debug]
```

</details>


#### Collector deployment

Now, we need to create the collector Deployment with the previously created `collector-config.yaml` ConfigMap. This requires specifying a volume in our Deployment and pointing the collector container to it. Since we used the `azuremonitor` exporter, we must deploy the OpenTelemetry Collector Contrib version. We recommend deploying version [0.88.0](https://github.com/open-telemetry/opentelemetry-collector-contrib/releases/tag/v0.88.0), as this is the latest version we have tested.

In addition to the Deployment, we will need a Service that exposes the `grpc-otlp` and `http-otlp` ports. 

Create a `collector-deployment.yaml` under `k8s-files` folder and feel free to figure out the syntax, or use the file provided below.

<details markdown="1">
<summary>🔦 OpenTelemetry Collector Deployment YAML </summary>

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
      containers:
      - name: otelcol
        args:
        - --config=/conf/collector-config.yaml
        image: otel/opentelemetry-collector-contrib:0.88.0
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

</details>


Great job! We’re now ready to deploy the collector 🎉! Go ahead and execute the following commands to deploy:

```yaml
kubectl apply -f k8s-files/collector-config.yaml
kubectl apply -f k8s-files/collector-deployment.yaml 
```

> 📝 **Note:** Don't forget to replace the `INSTRUMENTATION_KEY_PLACEHOLDER` with your Application Insights instrumentation key.

### 🐳 Deploy auto-instrumented Applications

Now that we have the collector deployed, let's redeploy our new shiny auto-instrumented services ✨.

Remember the environment variables you looked up to configure the SDK? Now it's time to use those and pass them into your deployment.

In case you haven't found them, these are important SDK Configuration OpenTelemetry environment variables: 

Common to all languages: 
* [OTEL_EXPORTER_OTLP_ENDPOINT](https://opentelemetry.io/docs/concepts/sdk-configuration/otlp-exporter-configuration/#endpoint-configuration) - this environment variables let you configure an OTLP/gRPC or OTLP/HTTP endpoint for your traces, metrics, and logs. In our case, we want to send it to OpenTelemetry Collector, so we need to specify here OpenTelemetry Collector endpoint.
* [OTEL_SERVICE_NAME](https://opentelemetry.io/docs/concepts/sdk-configuration/general-sdk-configuration/#otel_service_name) - This will allow you to later distinguish which service your data originated from.
* [OTEL_LOGS_EXPORTER](https://opentelemetry.io/docs/concepts/sdk-configuration/general-sdk-configuration/#otel_logs_exporter) - Specifies which exporter is used for logs. It defaults to `otlp`, though in Java application needs to be explicitly specified.

Specific environment variables to .NET auto-instrumentation can be found [here](https://github.com/open-telemetry/opentelemetry-dotnet-instrumentation#instrument-a-net-application).

In case you are stuck, just open the section below to see what the update deployment manifest should look like.

<details markdown="1">
<summary>🔦 Deployment yaml for auto-instrumentation <code>devices-api</code> service</summary>

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
          image: acr${project-name}.azurecr.io/devices-api:latest
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
                  name: application-secrets
                  key: CosmosDBEndpoint
            - name: AZURE_COSMOS_DB_KEY
              valueFrom:
                secretKeyRef:
                  name: application-secrets
                  key: CosmosDBKey
            - name: AZURE_COSMOS_DB_NAME
              valueFrom:
                secretKeyRef:
                  name: application-secrets
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

---

apiVersion: v1
kind: Service
metadata:
  name: devices-api-service
spec:
  type: LoadBalancer
  ports:
  - port: 8080 
    targetPort: 8080
  selector:
    app: devices-api
```

</details>

<details markdown="1">
<summary>🔦 Deployment yaml for auto-instrumentation <code>device-state-manager</code> service</summary>

```yaml
kind: Deployment
apiVersion: apps/v1

metadata:
  name: devices-state-manager

spec:
  replicas: 1
  selector:
    matchLabels:
      app: devices-state-manager
  template:
    metadata:
      labels:
        app: devices-state-manager
    spec:
      containers:
        - name: devices-state-manager
          image: acr${project-name}.azurecr.io/devices-state-manager:latest
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
          volumeMounts:
            - name: secrets-store-inline
              mountPath: "/mnt/secrets-store"
              readOnly: true
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
              valueFrom:
                secretKeyRef:
                  name: application-secrets
                  key: EventHubConnectionStringListen
            - name: EVENT_HUB_NAME
              valueFrom:
                secretKeyRef:
                  name: application-secrets
                  key: EventHubName
            - name: STORAGE_CONNECTION_STRING
              valueFrom:
                secretKeyRef:
                  name: application-secrets
                  key: StorageAccountConnectionString
            - name: BLOB_CONTAINER_NAME
              value: event-hub-data
            - name: DEVICE_API_URL
              value: "http://devices-api-service:8080"
      volumes:
        - name: secrets-store-inline
          csi:
            driver: secrets-store.csi.k8s.io
            readOnly: true
            volumeAttributes:
              secretProviderClass: "kvprovider"
---

apiVersion: v1
kind: Service
metadata:
  name: devices-state-manager-service
spec:
  type: LoadBalancer
  ports:
  - port: 8090
    targetPort: 8090
  selector:
    app: devices-state-manager
```

</details>

Using these new deployment yaml's, you can redeploy the applications into your AKS cluster.  In case you forgot how to do that, section  [🚀  Deploy Application](../02-deploy-application/README.md#🚀-deploy-application) is your friend! 😉

Nearly done! Go grab another coffee ☕ (or tea 🍵, we don't discriminate) and come back to a bunch of telemetry  already send upstream and in your Application Insights!

To learn how to use all this cool new data, head to the next section where we see how cool Application Insights actually is 😜.

## Navigation

[Previous Section ⏪](../02-deploy-application/README.md) ‖ [Return to Main Index 🏠](../README.md) ‖
[Next Section ⏩️](../04-visualization/README.md)
