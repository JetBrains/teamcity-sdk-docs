[//]: # (title: Custom Statistics)
[//]: # (auxiliary-id: Custom+Statistics.html)

TeamCity provides a number of ways to customize statistics. You can add your own custom metrics to integrate your tools/processes, insert any statistical chart/report into statistic page extension places and so on.

This page describes programmatic approaches to statistics customization. For user-level customizations, please refer to the [Custom Chart](https://www.jetbrains.com/help/teamcity/?custom-chart) page.


## Customize TeamCity Statistics Page

### Add Chart

To add a chart to the __Statistics__ tab for a project or build configuration, use the `ChartProviderRegistry` bean:


```java
public MyExtension(ChartProviderRegistry registry, ProjectManager manager) {
 registry.getChartProvider("project-graphs").registerChart(manager.findProjectByExternalId("externalId"), createGraphBean());
// "project-graphs" for Project Statistics Tab
// "buildtype-graphs" for Build Configuration Statistics Tab
}

public GraphBean createGraphBean() {
 // creates GraphBean
}

```



### Add Custom Content

To add custom content to the __Statistics__ tab for a project or build configuration, use the following example [here](web-ui-extensions.md) and the appropriate `PlaceId`:
*  * for Build Configuration Statistics tab, use `PlaceId.BUILD\_CONF\_STATISTICS\_FRAGMENT`
 * for Project Statistics tab, use `PlaceId.PROJECT\_STATS\_FRAGMENT`.
## Add Statistics to your Custom Pages

To add charts to your custom JSP pages, use the `<buildGraph>` tag and a special controller accessible on `"/buildGraph.html"`. It requires the `jsp` attribute leading to your page:


```jsp
new ModelAndView("/buildGraph.html?jsp=" +myDescriptor.getPluginResourcesPath("sampleChartPage.jsp"))'

```



To insert statistics chart into the `sampleChartPage.jsp`:


```jsp
<%@taglib prefix="stats" tagdir="/WEB-INF/tags/graph"%>
<stats:buildGraph id="g1" valueType="BuildDuration"/>

```



### Customize Chart Appearance

<table><tr>

<td>
Attribute


</td>

<td>
Description


</td>

<td>
Usage


</td></tr><tr>

<td>
`width`, `height`


</td>

<td>
modify the chart image size


</td>

<td>
Integer value


</td></tr><tr>

<td>
`hideFilters`


</td>

<td>
suppress filter controls


</td>

<td>
Comma separated filters names: 'all', 'series', 'average', 'showFailed', 'range', 'yAxisType', 'forceZero' or 'yAxisRange'


</td></tr><tr>

<td>
`defaultFilter`


</td>

<td>
default filter state


</td>

<td>
Comma separated names: 'showFailed', 'averaged', 'logYAxis', 'autoscale'


</td></tr><tr>

<td>
`hints`


</td>

<td>
chart style


</td>

<td>
Set to 'rendererB' for a bar chart


</td></tr></table>

## Add Custom Build Metrics

To add a custom build metric, in addition to [the built-in methods](https://www.jetbrains.com/help/teamcity/https://www.jetbrains.com/help/teamcity/?custom-chart), you can extend `BuildValueTypeBase` to define your build metric calculation method, appearance, and key. After that you can reference this metric by its key in statistics chart/report tags.

  __See also:__

__Extending TeamCity__: [Build Script Interaction with TeamCity](https://www.jetbrains.com/help/teamcity/?build-script-interaction-with-teamcity)
