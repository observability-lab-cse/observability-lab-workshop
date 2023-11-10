# Visualization

> Goal: Gain a basic understanding of `Kusto` and learn how to use it effectively when creating vizualisations of your data.

<!-- 1. Define SLIs and other usefull tools on how to vizualize the state of your application
2. Use azure dashboards and workbooks to visualize
3. Compare visibility of your solution from step 03 to now -->

In the previous section we focused on setting up a collection of telemetry data. In this section we will focus on how to explore and visualize data so we can easily understand the state of our application.

Azure provides many out-of-the-box visualizations. In most of the Azure services, when you open an Overview section, you will see a couple of metrics charts. Let's first have a look at Event Hub. The Overview will look like this:

![EventHub-overview](./images/EventHub-overview.png)

When you click on any of the charts, you will be redirected to Metrics section. There you have various possibilities, like selecting other available metrics with aggregations, or applying a filter, or changing a chart type (to area, bar or scatter chart). You can also remove any of the pre-selected metrics. Explore these options and create your own metric charts.

![EventHub-metrics](./images/EventHub-metrics.png/)

Now let's look at the Application Insights. Application Insights offers more curated monitoring experience. Under the Overview section it also shows a couple of metric charts.

![AppInsights-overview](./images/AppInsights-overview.png)

But when you click on any of them, you will be redirected to so-called Investigate experiance. For example, when you click on the Failed requests chart, you will be redirected to Investigate Failures view

![AppInsights-overview](./images/AppInsights-failures.png)

while Server response time chart redirects to Investigate Performance view.   

![AppInsights-overview](./images/AppInsights-performance.png)
