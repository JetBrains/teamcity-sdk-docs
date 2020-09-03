[//]: # (title: Basic UI Plugins)
[//]: # (auxiliary-id: Basic+UI+Plugins.html)

This guide explains how to create a basic UI plugin based on the new [front-end extensions](front-end-extensions.md) paradigm.

Source branch with an example project: [example/basic-plugin](https://github.com/JetBrains/teamcity-sakura-ui-plugins/tree/example/basic-plugin).

Every TeamCity UI Plugin must contain at least two files: the Controller itself (`.java`) and a resource file JSP. Every time we create a Plugin, we create a relation between Place ID and plugin resources. Using the TeamCity Open API, we let the TeamCity Core know, that there is a newly registered plugin for a certain Place ID; Whenever TeamCity meets this Place ID in JSP / TAG source code, it should render the plugin content.

Please, open the `src/main/java/com/demoDomain/teamcity/demoPlugin/controllers/SakuraUIPluginController.java` file.

Here you’ll see the boilerplate, where the controller constructor creates a SimplePageExtension instance:

```java
public class SakuraUIPluginController {
    private static final String PLUGIN_NAME = "SakuraUI-Plugin";

    public SakuraUIPluginController(
            @NotNull PluginDescriptor descriptor,
            @NotNull PagePlaces places
            ) {
            
        new SimplePageExtension(
            places,
            PlaceId.SAKURA_HEADER_NAVIGATION_AFTER, // 1
            PLUGIN_NAME, // 2
            descriptor.getPluginResourcesPath("basic-plugin.jsp") // 3
        )
            .addCssFile("basic-plugin.css") // 4
            .register();
    }
}

```

This piece of code do the next things:

1. It tells the TeamCity Core, that the plugin should be placed in `PlaceId.SAKURA_HEADER_NAVIGATION_AFTER`. Where is it? Just open your TeamCity instance with a GET param pluginDevelopmentMode=true. In our case this is `http://localhost:8111/bs/project/_Root?mode=builds&pluginDevelopmentMode=true`. The `PlaceID` is available both in Sakura UI and in Main UI:

[Image: image.png][Image: image.png]

1. The UI Plugin will be named, as it defined in the Private constant PLUGIN_NAME. 
2. This plugin uses a “basic-plugin.jsp” as an entry point. Next time Plugin Wrapper will try to load your Plugin, it will request [server]/plugins/SakuraUI-Plugin/basic-plugin.jsp as an entry point.
3. <div class="basic-plugin-wrapper">Here is a basic plugin.</div>
4. This plugin should load a “basic-plugin.css”
5. @keyframes rainbow {
      from {
        color: red;
      }
      to {
        color: blue;
      }
    }
    
    .basic-plugin-wrapper {
        animation: rainbow 5s infinite;
        display: inline;
        word-break: break-all;
    }

*Note: *It’s not forbidden to add the JavaScript file (using the addJsFile) to this plugin and attach some logics like “make a request every time context updated”. For example, it could be a code snippet for advertisement systems to send a pageView event. Whatever you imagine. But keep in mind: if you attach any listeners, intervals, subscriptions, timeouts or make any async operations in this JavaScript file, they will be fired every time plugin re-renders. It would result in race conditions and memory leaks. Looking ahead: we have a solution for this type of plugins. It relies on Plugin Lifecycle event and we will describe in section “Advanced plugins”.

Now you can compile your plugin using the Intellij IDEA Run Configuration “Build Plugin” or using  CLI command. 

mvn package

After a few seconds maven will output a *.zip archive. Then, simply add the plugin via the Administrator Panel.
That’s it. Your basic plugin is ready!
[Image: image.png]
This is a perfect place to hold on and investigate, how the plugin works under the hood. Please, open the page one more time, now in Development Mode. Then open the Browser Developer Tools. You will see a lot of debugging information for your plugin. 

Usually, plugin goes through 4 lifecycle events:

*ON_CREATE* - this is an internal phase, when TeamCity Core requests Plugin Metadata such as a Plugin Controller URL, attached CSS / JS files, parse JS / CSS.

*ON_CONTENT_UPDATE* - during this phase Plugin Wrapper takes the content from the JSP file and puts it in the separate HTML container.

*ON_CONTEXT_UPDATE* - this event fires every time plugin receives new Plugin UI Context. In our case it receives it for the first time.

*ON_MOUNT* - invokes, when HTML content is attached to the DOM. 
[Image: image.png]Let’s go further. If you navigate to another Project / Build Configuration, you will notice, that plugin disappeared and then appeared again. At the same time, in console appeared few more lifecycle events:
[Image: image.png]*ON_CONTEXT_UPDATE* - because you changed the navigation Context

*ON_DELETE* - this phase is opposite to the ON_CREATE phase. During this phase the Plugin Wrapper removes the plugin from the PluginRegistry and removes the HTML Elements for the plugin.

Then follows ON_CREATE, ON_CONTENT_UPDATE, ON_CONTEXT_UPDATE, ON_MOUNT... Every time you change the location, the plugin is constructed from a scratch, by passing through the same steps.


*Basic plugin (ver. 2, using PluginUIContext)*

branch: example/basic-plugin-v2

The basic plugin, we wrote a moment ago, provides not much benefits, unless yours the only goal is to draw kittens or rainbow texts in the header. In most cases plugin should provide useful data about the current selected entity, whether it is a Build Configuration ID, Project ID or other ID. The data, we use to fill the HTML elements, called Model. When we use a Basic plugin v.1, we have empty Model. Let’s fill it!

src/main/java/com/demoDomain/teamcity/demoPlugin/controllers/SakuraUIPluginController.java

```java
public class SakuraUIPluginController extends BaseController {
    private static final String PLUGIN_NAME = "SakuraUI-Plugin";
    private final PluginDescriptor myPluginDescriptor;
    private final ProjectManager myProjectManager;

    public SakuraUIPluginController(
        @NotNull PluginDescriptor descriptor,
        @NotNull PagePlaces places,
        @NotNull WebControllerManager controllerManager,
        @NotNull ProjectManager projectManager
        ) {
        myPluginDescriptor = descriptor;
        myProjectManager = projectManager;

        String url = "/demoPlugin.html"; // 1
        final SimplePageExtension pageExtension = new SimplePageExtension(places); // *
        pageExtension.setPluginName(PLUGIN_NAME); // *
        pageExtension.setPlaceId(PlaceId.SAKURA_BUILD_CONFIGURATION_BEFORE_CONTENT); // *
        pageExtension.setIncludeUrl(url); // *
        pageExtension.addCssFile("basic-plugin.css"); // *
        pageExtension.register(); // *

        controllerManager.registerController(url, this);
    }

    @Nullable
    @Override
    protected ModelAndView doHandle(@NotNull HttpServletRequest request, @NotNull HttpServletResponse response) throws Exception { // 2
        final ModelAndView mv = new ModelAndView(myPluginDescriptor.getPluginResourcesPath("basic-plugin.jsp")); // 3
        PluginUIContext pluginUIContext = PluginUIContext.getFromRequest(request); // 4
        String btId = pluginUIContext.getBuildTypeId(); // 5
        if (btId != null) {
            SBuildType buildType = myProjectManager.findBuildTypeByExternalId(btId); // 6
            mv.getModel().put("buildType", buildType); // 7
        }
        return mv;
    }
}

```

The important change is: now the SakuraUIPluginController extends the BaseController. It makes the Plugin to override the method doHandle where the Controller gets access to a Request data.

So, let’s go step by step. Steps marked with asterisk are the multiline representation of a previous Controller. We changed PlaceID to SAKURA_BUILD_CONFIGURATION_BEFORE_CONTENT to make plugin appeared in a new place. 

1. Instead of a basic-plugin.jsp we now use /demoPlugin.html. It makes plugin to register controller at the [server]/demoPlugin.html. 
2. Every time a request comes to /demoPlugin.html, a method doHandle intercepts this request and processes it
3. Plugin creates ModelAndView and passes the link to a View container. It’s the same JSP file we used before.
4. PluginUIContext controller parses the Request parameters 
5. Plugin receives current buildTypeId
6. If buildTypeId is not empty, we ask the Core to find the build configuration data
7. Plugin controller passes the build type to a JSP in a variable called “buildType” and return the result

There is also a slight change in JSP. If there is a build configuration, then we show its’ name:

<c:if test="${not empty buildType}">
    <div>
        The selected build configuration: <c:out value="${buildType.name}"/>
    </div>
</c:if>

<div class="dummy-plugin-wrapper">Here is a basic plugin.<c:out value="${param['pluginUIContext']}"></c:out></div>

Let’s compile and update the plugin and then open any build configuration page:

[Image: image.png]It works the same to the previous Basic UI Plugin, but now it contains the data from a TeamCity Database. 

*What do we have now? *

At this moment we have a simple plugin. In many cases it’s enough good: it integrates to the Sakura UI and to the Main UI, it could be fulfilled on a backend and it reacts on any Context Updates. If you wrote TeamCity UI Plugins before 2020.2, you may notice, that the major things that were changed - the Place ID and the brand-new PluginUIContext. So, if you have your own Plugin, try to update it in the same manner and, probably, it becomes “Sakura UI - ready” Plugin. 

Actually, that was our the very first intention - just provide the way to integrate plugins in the Sakura UI with a minimum efforts. 

But it’s not ideal though. If you use your plugin in Header, you observe layout shifting. If you use JavaScript to enrich the default plugin behaviour - it’s not clear in what certain moment you should add the event handlers. There are some solutions come to mind: it’s possible to check the DOM every few seconds; or use Mutation Observer; For sure, there are many more ways to do that, but they have their limits and, to be fair, some of them should never come to the production.

And what to do, if you send the request, which should update the plugin content, but during the request execution you moved from one Build Configuration to another. For sure, if you didn’t cancel the Promise handling, it will be resolved and, depends on the logics and race conditions, you will receive a newly generated Plugin, filled with data from a previous request. 

We have an answer to all issues. That is the part, where controlled plugins come in front.