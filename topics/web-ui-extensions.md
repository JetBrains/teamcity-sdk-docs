[//]: # (title: Web UI Extensions)
[//]: # (auxiliary-id: Web+UI+Extensions.html)



This section covers:

<tip>

Hint: you can use source code of the existing plugins as a reference, for example:
* [https://plugins.jetbrains.com/plugin/8979-server-profiling](https://plugins.jetbrains.com/plugin/8979-server-profiling)
</tip>

## Getting Started

The simplest way of adding your own custom tab is to derive from one of the classes:

* [`jetbrains.buildServer.web.openapi.project.ProjectTab`](http://javadoc.jetbrains.net/teamcity/openapi/current/jetbrains/buildServer/web/openapi/project/ProjectTab.html)
* [`jetbrains.buildServer.web.openapi.buildType.BuildTypeTab`](http://javadoc.jetbrains.net/teamcity/openapi/current/jetbrains/buildServer/web/openapi/buildType/BuildTypeTab.html)
* [`jetbrains.buildServer.web.openapi.ViewLogTab`](http://javadoc.jetbrains.net/teamcity/openapi/current/jetbrains/buildServer/web/openapi/ViewLogTab.html)
* [`jetbrains.buildServer.controllers.admin.AdminPage`](http://javadoc.jetbrains.net/teamcity/openapi/current/jetbrains/buildServer/controllers/admin/AdminPage.html)

This will add your tab to the project, build type (build configuration), build or administration page respectively. Here's an example of the Diagnostics admin page:


```java
public class DiagnosticsAdminPage extends AdminPage {
  public DiagnosticsAdminPage(@NotNull PagePlaces pagePlaces, @NotNull PluginDescriptor descriptor) {
    super(pagePlaces);
    setPluginName("diagnostics");
    setIncludeUrl(descriptor.getPluginResourcesPath("/admin/diagnosticsAdminPage.jsp"));
    setTabTitle("Diagnostics");
    setPosition(PositionConstraint.after("clouds", "email", "jabber"));
    register();
  }

  @Override
  public boolean isAvailable(@NotNull HttpServletRequest request) {
    return super.isAvailable(request) && checkHasGlobalPermission(request, Permission.CHANGE\_SERVER\_SETTINGS);
  }

  @NotNull
  public String getGroup() {
    return SERVER\_RELATED\_GROUP;
  }
}

```



There are a couple of things to note here:
* it is important to call "register" method; ProjectTab, BuildTypeTab and ViewLogTab will do that for you automatically, AdminPage won't, that's why the call is there;
* TeamCity might have difficulties with finding your resources (JSP, CSS, JS) if you don't refer to your resources through the PluginDescriptor.
* the page above doesn't provide any model to the JSP. If you need one, just override the "fillModel" method.
Here's another example of the project tab:


```
public class CurrentProblemsTab extends ProjectTab {
  public CurrentProblemsTab(@NotNull PagePlaces pagePlaces,
                            @NotNull ProjectManager projectManager,
                            @NotNull PluginDescriptor descriptor) {
    super("problems", "Current Problems", pagePlaces, projectManager, descriptor.getPluginResourcesPath("problems.jsp"));
    // add your CSS/JS here
  }

  @Override
  protected void fillModel(@NotNull Map<String, Object> model, @NotNull HttpServletRequest request,
                           @NotNull SProject project, @Nullable SUser user) {
    // add your data here
  }
}

```



That's it! Just specify your tab as a Spring bean, and you'll be able to see your tab in TeamCity.

<tip>

We are using [Spring MVC](http://static.springframework.org/spring/docs/2.5.x/reference/mvc.html) web framework.
</tip>

## Under the Hood

If you download and take a look at the TeamCity open API sources, you'll notice that all tabs above derive from the `jetbrains.buildServer.web.openapi.SimpleCustomTab`. And the only major difference between them all is a `jetbrains.buildServer.web.openapi.PlaceId` they specify in constructor. Here's what they use:
* PlaceId.PROJECT\_TAB
* PlaceId.BUILD\_CONF\_TAB
* PlaceId.BUILD\_RESULTS\_TAB
* PlaceId.ADMIN\_SERVER\_CONFIGURATION\_TAB
Don't get confused by the variety of names, it's a long story. The main thing is there are more than 30 other place ids that you can hook into!
* PlaceId.ALL\_PAGES\_HEADER
* PlaceId.AGENT\_DETAILS\_TAB
* PlaceId.LOGIN\_PAGE
* ...
There is a convention that a place id named as a TAB can be used with the SimpleCustomTab. Others cannot, and to use them you will have to deal with low level `jetbrains.buildServer.web.openapi.SimplePageExtension`. But that's pretty much the only change, take a look at the example: 


```
public class ChangedFileExtension extends SimplePageExtension {
  public ChangedFileExtension(@NotNull PagePlaces pagePlaces,
                              @NotNull PluginDescriptor descriptor) {
    super(pagePlaces, PlaceId.CHANGED\_FILE\_LINK, "changeViewers", descriptor.getPluginResourcesPath("changedFileLink.jsp"));
    register();
  }

  @Override
  public boolean isAvailable(@NotNull HttpServletRequest request) {
    return super.isAvailable(request);
  }

  @Override
  public void fillModel(@NotNull Map<String, Object> model, @NotNull HttpServletRequest request) {
    // fill model
  }
}

```



This extension provides a custom HTML (usually a link) near the each file in the modification's list. We use it to add "Open in IDE", "Open in External Change Viewer", etc links. In this particular case the file itself is passed via "changedFile" attribute of the request, but this is different for different extensions.

A couple of useful notes:
* `isAvailable(HttpServletRequest)` method is called to determine whether page extension content should be shown or not.
* in case `isAvailable(HttpServletRequest)` is true, the `fillModel(Map, HttpServletRequest)` method will always be called and JSP will be rendered in UI. You cannot abort the process after `isAvailable(HttpServletRequest)` is done, that's why it's usually inconvenient to handle POST requests in extensions. Use a custom controller for that (see below).
* One more case when you might need a custom controller is when you need to process HTTP response manually, e.g. stream a file content. `fillModel(Map, HttpServletRequest)` won't allow you to do that.
## Developing a Custom Controller

Sometimes page extensions provide interaction with user and require communication with server. For example, your page extension can show a form with a "Submit" button. In this case in addition to writing your own page extension, you should provide a _controller_ which will process requests from such forms, and use path to this controller in the form action attribute (the path is a part of URL without context path and query string).

Example:


```
public class ServerConfigGeneralController extends BaseFormXmlController {
  public ServerConfigGeneralController(@NotNull SBuildServer server,
                                       @NotNull WebControllerManager webControllerManager) {
    super(server);
    webControllerManager.registerController("/my/path/", this);
  }

  @Override
  @Nullable
  protected ModelAndView doGet(@NotNull final HttpServletRequest request, @NotNull final HttpServletResponse response) {
    return null;
  }

  @Override
  protected void doPost(@NotNull final HttpServletRequest request, @NotNull final HttpServletResponse response, @NotNull final Element xmlResponse) {
    return null;
  }
}

```



To simplify things your controller can extend our `jetbrains.buildServer.controllers.BaseController` class and implement `BaseController.doHandle(HttpServletRequest, HttpServletResponse)` method.

With the custom controller you can provide completely new pages.

## Obtaining paths to JSP files

Plugin resources are unpacked to `<TeamCity web application>/plugins` directory when server starts. However to construct paths to your JSP or images in Java it is recommended to use `jetbrains.buildServer.web.openapi.PluginDescriptor`. This descriptor can be obtained as any other Spring service.

In JSP files to construct paths to your resources you can use `${teamcityPluginResourcesPath}`. This attribute is provided by TeamCity automatically, you can use it like this:

```jsp
<img src="${teamcityPluginResourcesPath}your_image.gif" height="16" width="16" border="0">

```

Note: &lt;c:url/&gt; is required to construct correct URL in case if TeamCity is deployed under the non root context.

## Classes and interfaces from TeamCity web open API

<table><tr>

<td>
Class / Interface


</td>

<td>
Description


</td></tr><tr>

<td>[`jetbrains.buildServer.web.openapi.PlaceId`](http://javadoc.jetbrains.net/teamcity/openapi/current/jetbrains/buildServer/web/openapi/PlaceId.html)

</td>

<td>
A list of page place identifiers / extension points


</td></tr><tr>

<td>[`jetbrains.buildServer.web.openapi.PagePlace`](http://javadoc.jetbrains.net/teamcity/openapi/current/jetbrains/buildServer/web/openapi/PagePlace.html)

</td>

<td>
A single page place associated with __PlaceId__, allows to add / remove extensions


</td></tr><tr>

<td>[`jetbrains.buildServer.web.openapi.PageExtension`](http://javadoc.jetbrains.net/teamcity/openapi/current/jetbrains/buildServer/web/openapi/PageExtension.html)

</td>

<td>
Page extension interface


</td></tr><tr>

<td>[`jetbrains.buildServer.web.openapi.SimplePageExtension`](http://javadoc.jetbrains.net/teamcity/openapi/current/jetbrains/buildServer/web/openapi/SimplePageExtension.html)

</td>

<td>
Base class for page extensions


</td></tr><tr>

<td>[`jetbrains.buildServer.web.openapi.CustomTab`](http://javadoc.jetbrains.net/teamcity/openapi/current/jetbrains/buildServer/web/openapi/CustomTab.html)

</td>

<td>
Custom tab extension interface


</td></tr><tr>

<td>[`jetbrains.buildServer.web.openapi.PagePlaces`](http://javadoc.jetbrains.net/teamcity/openapi/current/jetbrains/buildServer/web/openapi/PagePlaces.html)

</td>

<td>
Maintains a collection of page places and allows to locate __PagePlace__ by __PlaceId__


</td></tr><tr>

<td>[`jetbrains.buildServer.web.openapi.WebControllerManager`](http://javadoc.jetbrains.net/teamcity/openapi/current/jetbrains/buildServer/web/openapi/WebControllerManager.html)

</td>

<td>
Maintains a collection of custom controllers, allows to register custom controllers


</td></tr><tr>

<td>[`jetbrains.buildServer.controllers.BaseController`](http://javadoc.jetbrains.net/teamcity/openapi/current/jetbrains/buildServer/controllers/BaseController.html)

</td>

<td>
Base class for controllers


</td></tr></table>
