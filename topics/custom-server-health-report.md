[//]: # (title: Custom Server Health Report)
[//]: # (auxiliary-id: Custom+Server+Health+Report.html)

To report custom [server health](https://www.jetbrains.com/help/teamcity/?server-health) items, do the following:
* Create your own reporter
* Create a custom page extension to render items reported by you

### Reporting Server Health Items

To make a reporter, create a subclass of [`jetbrains.buildServer.serverSide.healthStatus.HealthStatusReport`](http://javadoc.jetbrains.net/teamcity/openapi/current/jetbrains/buildServer/serverSide/healthStatus/HealthStatusReport.html).

Particulary, you must override method [`jetbrains.buildServer.serverSide.healthStatus.HealthStatusReport#report(jetbrains.buildServer.serverSide.healthStatus.HealthStatusScope, jetbrains.buildServer.serverSide.healthStatus.HealthStatusItemConsumer)`](http://javadoc.jetbrains.net/teamcity/openapi/current/jetbrains/buildServer/serverSide/healthStatus/HealthStatusReport.html#report(jetbrains.buildServer.serverSide.healthStatus.HealthStatusScope,%20jetbrains.buildServer.serverSide.healthStatus.HealthStatusItemConsumer)).

The items should be reported according to the analysis scope passed as a parameter to this method using the appropriate method of resultConsumer. If you try to consume an object which is not in the scope of the current analysis, it will be filtered out by the consumer and will not appear in the report.

This method is always called with system privileges (in all permissions mode). Any permissions checks should be avoided here.

While reporting items, additional data required for further items presentation could be provided.

### Presenting Server Health Items to User

To present reported items, provide a [custom page extension](web-ui-extensions.md) and connect it to `PlaceId.HEALTH_STATUS_ITEM`. The simplest way to do it is to create a subclass of [`jetbrains.buildServer.web.openapi.healthStatus.HealthStatusItemPageExtension`](http://javadoc.jetbrains.net/teamcity/openapi/current/jetbrains/buildServer/web/openapi/healthStatus/HealthStatusItemPageExtension.html).

#### Handling Current User Permissions

To handle permissions of the user viewing the reported items, use the subclass of `HealthStatusItemPageExtension`. In order to do it, override the [`jetbrains.buildServer.web.openapi.healthStatus.HealthStatusItemPageExtension#isAvailable`](http://javadoc.jetbrains.net/teamcity/openapi/current/jetbrains/buildServer/web/openapi/healthStatus/HealthStatusItemPageExtension.html#isAvailable) method. You can also limit the set of pages where your items should be presented to the __Administration__ area of the Web UI by setting FALSE to [`jetbrains.buildServer.web.openapi.healthStatus.HealthStatusItemPageExtension#setVisibleOutsideAdminArea`](http://javadoc.jetbrains.net/teamcity/openapi/current/jetbrains/buildServer/web/openapi/healthStatus/HealthStatusItemPageExtension.html#setVisibleOutsideAdminArea)

The particular strategy of handling permissions depends on the report details. The proposed behaviour is to completely hide items from the user only if none of related objects is available. In other cases it makes sense to show the item, but filter all inaccesible objects.

#### Switching between Display Modes

There are the following places in the Web UI where Server Health items could be presented:

* the report page (__Administration | Server Health__)
* the notes section at the top of all pages (global items with severity more than 'info')
* in\-place (in the popups appearing on some pages).

To define in what display mode a server health item is presented, use [`jetbrains.buildServer.web.openapi.healthStatus.HealthStatusItemDisplayMode`](http://javadoc.jetbrains.net/teamcity/openapi/current/jetbrains/buildServer/web/openapi/healthStatus/HealthStatusItemDisplayMode.html). The `HealthStatusItemDisplayMode.GLOBAL` value is passed to the request when an item is shown on the report page, `HealthStatusItemDisplayMode_INPLACE` is used in other cases.

Here is an example of handling the display mode in a JSP page.

```JSP
<jsp:useBean id="showMode" type="jetbrains.buildServer.web.openapi.healthStatus.HealthStatusItemDisplayMode" scope="request"/>
<c:set var="inplaceMode" value="<%=HealthStatusItemDisplayMode.IN_PLACE%>"/>

<c:choose>
  <c:when test="${showMode == inplaceMode}">
    //Smth about current object
  </c:when>
  <c:otherwise>
    //Smth about related object
  </c:otherwise>
</c:choose>

```



#### Presenting Results In-place

While presenting results in-place, it might be nesessary to know the ID of an object being viewed at the moment. The ID can be retrieved using the following requests:

* on the __Edit the VCS root settings__ page


```JSP
<jsp:useBean id="vcsRootId" type="java.lang.String" scope="request"/>

```



* on the __Edit the Build Configuration__ setting page


```JSP
<jsp:useBean id="buildTypeId" type="java.lang.String" scope="request"/>

```



* on the __Edit the Build Configuration Template__ settings page


```JSP
<jsp:useBean id="templateId" type="java.lang.String" scope="request"/>

```


