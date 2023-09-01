# Azure Observability 101

> Goal: Understanding the resources involved, when each setup makes sense, and having key information on when to choose the appropriate visualization tool.

## Azure Monitoring: Resources and Their Interplay

- Main components: Data storage (Log Analytics workspace, Metrics workspaces), App Insights, Visualization (refer to Azure docs for details).
- Overlap between these components, e.g., App Insights vs Log Analytics workspace.

## Data Storage

- Differentiating between Metrics-based and Logs-based telemetry in Azure.

### Azure Metric Workspace

- Introduction to the new Metrics Workspace for Prometheus metrics.

### Azure Log Analytics Workspace

- Understanding Log Analytics workspace-backed resources and their effective utilization. For example, telemetry data of non-workspace-backed resources can only be used within the scope of the same resource.
- Comparing a centralized workspace vs. individual (in-built) ones.

### Non-Internal Data and Incorporating them into Azure

- Utilizing methods such as Data Collection Rule, Quota, etc., to bring external data into Azure.

## Visualizations

- Exploring data sources for each visualization tool.
- Highlighting the pros and cons of each tool.

### Dashboards

### Workbooks

### Third-Party Tools

e.g., Grafana, Prometheus
