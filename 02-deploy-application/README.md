# Deploying the Application into AKS

> Goal:
>
> 1. Having the application run a full E2E and understanding the pieces.
>
> 2. Observe the application with basic "instrumentation," including emitted log lines and possibly some metrics using `kubectl`. Get familiar with out-of-the-box AKS cluster observability (e.g., workloads, pods, etc.). Look at Container Insights and its capabilities.

<!-- 1. Deploy Services to AKS
2. Create Devices and see temperature and state being updated
3. Try to access logs or get other insights into the solution as is -->

## 1. Deploy Application

Now, let's start deploying the individual services in our application. There are three components to our solution:

- Devices Manager: An API to update the values we have on our devices.
- Device API: An API that allows us to create and delete devices.
- Device Simulator: Simulates devices that publish their health and temperature.

The provisioning script should have given you a .env file at the root of your directory with all the needed environment variables for the next steps. You can source them as follows:

```bash
source .env
```

### Deploy: Devices API

Let's start by deploying the Device API application. The code for this service can be found [here](TODO). It's a Java Spring Boot REST API that allows you to list, create, update, and delete devices from your device registry.

The first step is to build and push the image to the registry.

<!-- TODO: from where to run the below commands-->

```sh
az acr login --name "$ACR_NAME".azurecr.io
docker tag "$DEVICE_API_IMAGE_NAME" "$ACR_NAME".azurecr.io/"$DEVICE_API_IMAGE_NAME"
docker push "$ACR_NAME".azurecr.io/"$DEVICE_API_IMAGE_NAME":"$TAG"
```

Referencing this image in the deployment either below or [here](TODO)

<details markdown="1">
<summary>Click here for the Device API deployment YAML</summary>

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

### Deploy: Devices Manager

Next on our agenda is the deployment of the Device Manager service. You can access the source code for this service [here](TODO). This .NET application plays a critical role by updating the temperature records of devices in the database as data flows in from each device.

As with our previous steps, we'll need to build the image and ensure it's pushed to your Azure Container Registry (ACR).

```sh
docker tag "$DEVICE_MANAGER_IMAGE_NAME" "$ACR_NAME".azurecr.io/"$DEVICE_MANAGER_IMAGE_NAME"
docker push "$ACR_NAME".azurecr.io/"$DEVICE_MANAGER_IMAGE_NAME":"$TAG"
```

<details markdown="1">
<summary>Click here for the Device API deployment YAML</summary>

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
          image: acr${project-name}.azurecr.io/device-manager:latest
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
---
apiVersion: v1
kind: Service
metadata:
  name: device-manager-service
spec:
  type: LoadBalancer
  ports:
    - port: 8090
      targetPort: 8090
  selector:
    app: device-manager
```

</details>

### Deploy: Device Simulator

To generate data from virtual devices for testing purposes, we're employing the device simulator, as also utilized in the [sample](TODO). This simulator effectively generates temperature data at defined intervals for each virtual device and transmits this data as messages to Event Hub.

<details markdown="1">
<summary>Click here for the Device Simulator deployment YAML</summary>

```yaml
kind: Deployment
apiVersion: apps/v1

metadata:
  name: devices-simulator

spec:
  replicas: 1
  selector:
    matchLabels:
      app: devices-simulator
  template:
    metadata:
      labels:
        app: devices-simulator
    spec:
      containers:
        - name: devices-simulator
          image: mcr.microsoft.com/oss/azure-samples/azureiot-telemetrysimulator:latest
          imagePullPolicy: Always
          resources:
            limits:
              cpu: 500m
              memory: 256Mi
            requests:
              cpu: 100m
              memory: 128Mi
          env:
            - name: EventHubConnectionString
              value: EVENT_HUB_CONNECTION_STRING_PLACEHOLDER
            - name: DeviceList
              value: "DEVICE_NAMES_PLACEHOLDER" # Specify your device names
            - name: MessageCount
              value: "0" # send unlimited
            - name: Interval
              value: "60000" # each device sends a message every 1 minute
            - name: Template
              value: '{"deviceId": "$.DeviceId", "deviceTimestamp": "$.Time", "temp": $.DoubleValue}'
            - name: Variables
              value: '[{"name": "DoubleValue", "randomDouble":true, "min":20.00, "max":28.00}]'
```

</details>

## Out of the box observability

Now, given we have enabled the [Container Insights feature](TBD) on AKS, we can already have a look at what Azure will give us out of the box.

<!-- TODO: go into details here -->

- Overview of pods and services
- Logs that are scraped
- Metrics that are available

Now this looks all good and great. There is an awesome overview of our cluster, but other than the logs (in a, let's be honest, rather unenice format), we have no real visibility on the application. No way to know if messages are being sent across the system, etc.
But luckily there is a simple way to fix this, which we will look at in the next chapter.
