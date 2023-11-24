# Observability Lab Workshop

This is a hands-on workshop to get familiar with the observability and the relevant tooling on Azure.

## Observability of a K8s cluster

This path will introduce :

- How to instrument applications using OpenTelemetry on Azure
- How to instrument your AKS cluster
- How to use your visualize your telemetry data by creating Azure Monitor Workbooks, Azure Dashboards

## How to use this workshop

For very section there is a designated branch on the following [Github repository](https://github.com/observability-lab-cse/observability-lab).
To set your environment to the state of the branch you can always just run `make` in the root directory and it should bring you up to speed.

Sections:

- [⚒️ Pre Requisites](./00-pre-requisite/README.md) - Covering the pre set up and tools that will be
  needed.
- [⚙️ Provision Infrastructure](./01-provision-infrastructure/README.md) - Provision AKS cluster, Application Insights etc.
- [🧩 Deploy application to AKS](./02-deploy-application/README.md) - Deploy all required components of the application
- [🔎 Add basic observability instrumentation](./03-add-basic-observability-instrumentation/README.md) - Using OpenTelemetry instrument your application
- [📈 Visualization](./04-vizualisation/README.md) - Visualize data with dashboards
- [📋 Dashboards](./05-dashboards/README.md) - Visualize data with dashboards
- [🚨 Alerts](./05-alert/README.md) - Creating alerts
- [🌟 Advanced Instrumentation](./06-advanced-observability-instrumentation/README.md) - Extend solution with more advanced instrumentation eg. Custom metrics, distributed tracing.

Additional Read:

- [📖 Azure Observability 101](./10-azure-observabity-101/README.md) - Covering the basics of the azure observability suite
