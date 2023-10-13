Here's your text with spelling and minor grammar issues corrected:

# Azure Observability 101

<!-- > Goal: Understanding the resources involved, when each setup makes sense, and having key information on when to choose the appropriate visualization tool.-->

This section should be a small cheatsheet about the observability landscape of Azure and what you need to know when you start creating resources for a solution. It will go over how your observability data is stored and the different visualization options.

## Azure Monitoring: Resources and Their Interplay

<!-- - Main components: Data storage (Log Analytics workspace, Metrics workspaces), App Insights, Visualization (refer to Azure docs for details).
- Overlap between these components, e.g., App Insights vs Log Analytics workspace. -->

<!-- ## Data Storage

 - Differentiating between Metrics-based and Logs-based telemetry in Azure. -->

Azure has a few observability resources that can be used when creating an observability suite for your solution. For now, let us focus on the following two, as they are most commonly used and cover most use cases:

- [Log Analytics Workspace](https://learn.microsoft.com/en-us/azure/azure-monitor/logs/log-analytics-workspace-overview)
- [Application Insights](https://learn.microsoft.com/en-us/azure/azure-monitor/app/app-insights-overview)
- Insights Agents: [VM Insights Agents](https://learn.microsoft.com/en-us/azure/azure-monitor/vm/vminsights-overview), [Container Insights Agents](https://learn.microsoft.com/en-us/azure/azure-monitor/containers/container-insights-overview), [Network Insights Agents](https://learn.microsoft.com/en-us/azure/network-watcher/network-insights-overview)

Let's look at what normally happens when people need observability.

1. Create Application Insights resource.
2. Pass the connection string of Application Insights to the application.
3. And bam 💥! Automatically, you get your data in beautiful dashboards on Application Insights.

If this is enough for you, great! But often these dashboards don't give you all the information about your application you need. So, in cases of creating new dashboards, alerts, or just out of pure curiosity, you want to know where you can find all the awesome data your Application Insights uses to create these dashboards.
And spoiler, Application Insights can but does not have to store your data!

Before we go further, it's important to keep in mind that Azure handles some telemetry data differently. In particular, built-in metrics are handled differently than things like logs, traces, events, and even custom metrics.

Let's have a look first at the whole group of logs, traces, events, custom metrics, etc.
Our application sends data to Application Insights, but what then?

### Azure Log Analytics Workspace

<!--
- Understanding Log Analytics workspace-backed resources and their effective utilization. For example, telemetry data of non-workspace-backed resources can only be used within the scope of the same resource.
- Comparing a centralized workspace vs. individual (in-built) ones. -->

> Note: If we refer to data in this section, we mean anything _but_ built-in metrics (e.g., logs, traces, events, custom metrics, etc.). How built-in metrics are handled is described in the next section.

As mentioned, Application Insights does not have to store your data; but it is always in some form backed by a Log Analytics workspace. Now there are two modes in which you can set up your Application Insights.

- Classic
- Workspace-based

### Classic

In this case, your Application Insights workspace is not directly backed by a workspace, and you have not configured your resource to be backed by a particular Log Analytics workspace; the data is stored by your individual resources or Application Insights itself. When you go to an existing resource, you will always see the tab `Logs`. That is a resource-internal Log Analytics workspace where the data is stored. As for the data you send Application Insights, they are stored in its own internal Log Analytics workspace.
This is easy and handy, as you don't need a specific new resource. However, when you want to write queries or create dashboards that need data from more than one resource type, you run into some limitations.

### Workspace-based

For the workspace-based approach, there is a central workspace. All resources and applications can send their data to that centralized place. The advantage there is that you can access data from multiple types of resources at the same time.

### Azure Metric Workspace

Now when it comes to in-built metrics, the story is slightly different.

- Introduction to the new Metrics Workspace for Prometheus metrics.

### Non-Internal Data and Incorporating Them into Azure

- Utilizing methods such as Data Collection Rule, Quota, etc., to bring external data into Azure.

## Visualizations

- Exploring data sources for each visualization tool.
- Highlighting the pros and cons of each tool.

### Dashboards

### Workbooks

### Third-Party Tools

e.g., Grafana 🚀