# Observability Workshop

🔍 Welcome to the Observability Workshop! Prepare to delve deep into the realms of observability and Azure's powerful toolkit.

Understanding the inner workings of your Kubernetes cluster can be an intriguing yet challenging pursuit. 🌐 Join us as we uncover the layers of your applications, delve into data visualization, and craft bespoke dashboards. In this hands-on workshop, we aim to equip you with the tools to gain critical insights into your system's performance.

## Observability of a K8s cluster

This workshop will include:

- 🛠️ Auto-instrumenting applications with OpenTelemetry
- 🖥️ Inspect Azure's AKS observability features
- 📊 Explore Application Insights and crafting custom telemetry visualizations using Azure Monitor Workbooks and Dashboards

## Navigating this Journey

> 📌 **Starting Point:** If you follow through the workshop from the start all you need is the starting branch [00-workshop](https://github.com/observability-lab-cse/observability-lab/tree/00-workshop) that is checkout by default when cloning the repository. Everything else you will build as you go.
>
> 📝 **Note:** In case you are only interested in certain section or parts of the workshop, each section has a designated branch in the corresponding GitHub code repository [observability-lab](https://github.com/observability-lab-cse/observability-lab). Check out the branch of the section your are interested in and the **⏩ Catch-up corner: If you missed previous sections 🏇** paragraph in that section will give you further instructions on how to get your workshop environment up to speed, as if you had done all previous sections all yourself 😉.


### 📱 Application

- [⚒️ Pre Requisites](./00-pre-requisite/README.md) - Covering the tools that will be
  needed.
- [⚙️ Provision Infrastructure](./01-provision-infrastructure/README.md) - Provision AKS cluster, Application Insights etc.
- [🧩 Deploy application to AKS](./02-deploy-application/README.md) - Deploy all required components of the application

### 🎻 Instrumentation

- [🔎 Add basic observability instrumentation](./03-add-basic-observability-instrumentation/README.md) - Using OpenTelemetry instrument your application
- [🌟 Custom metrics](./04-custom-metrics/README.md) - Add custom metrics to your application

### 🎨 Visualization

- [📈 Visualization](./05-visualization/README.md) - Use out-of-the-box Azure visualizations
- [📋 Dashboards](./06-dashboards/README.md) - Create your custom dashboard
- [🚨 Alerts](./07-alert/README.md) - Create alerts
