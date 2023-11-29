# Observability Workshop

🔍 Welcome to the Observability Workshop! Prepare to delve deep into the realms of observability and Azure's powerful toolkit.

Understanding the inner workings of your Kubernetes cluster can be an intriguing yet challenging pursuit. 🌐 Join us as we uncover the layers of your applications, delve into data visualization, and craft bespoke dashboards. In this hands-on workshop, we aim to equip you with the tools to gain critical insights into your system's performance.

## Observability of a K8s cluster

This workshop will includes:

- 🛠️ Auto-instrumenting applications with OpenTelemetry
- 🖥️ Inspect Azure's AKS observability features
- 📊 Explore Application Insights and crafting custom telemetry visualizations using Azure Monitor Workbooks and Dashboards

## Navigating this Journey

> 📝 **Note:** Each section of this workshop has a designated branch in the corresponding code  Github repository [observability-lab](https://github.com/observability-lab-cse/observability-lab). The **📌 Starting point 📌** paragraph in each section will give you further instructions on how to get your workshop environment up to speed 😉.

Sections:

- [⚒️ Pre Requisites](./00-pre-requisite/README.md) - Covering the tools that will be
  needed.
- [⚙️ Provision Infrastructure](./01-provision-infrastructure/README.md) - Provision AKS cluster, Application Insights etc.
- [🧩 Deploy application to AKS](./02-deploy-application/README.md) - Deploy all required components of the application
- [🔎 Add basic observability instrumentation](./03-add-basic-observability-instrumentation/README.md) - Using OpenTelemetry instrument your application
- [📈 Visualization](./04-visualization/README.md) - Use out-of-the-box Azure visualizations
- [📋 Dashboards](./05-dashboards/README.md) - Create your custom dashboard
- [🚨 Alerts](./06-alert/README.md) - Create alerts
- [🌟 Custom metrics](./07-custom-metrics/README.md) - Add custom metrics to your application
