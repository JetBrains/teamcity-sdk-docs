[//]: # (title: Controlled UI Plugins)
[//]: # (auxiliary-id: Controlled+UI+Plugins.html)

This guide explains how to create a controlled UI plugin based on the new [front-end extensions](front-end-extensions.md) paradigm.

__Source branch with the example project: [example/controlled-plugin](https://github.com/JetBrains/teamcity-sakura-ui-plugins/tree/example/controlled-plugin)__.

The name _controlled_ explains the main advantage of these plugins – a developer controls the plugin behavior. A controlled plugin knows how to react on the lifecycle events. It uses the Plugin API to update its content, to subscribe and unsubscribe to events, and to abort requests. In other words, controlled plugins allow creating rich applications within the TeamCity UI. Moreover, when Plugin Wrapper knows that a plugin is controlled by a developer, it stops requesting the plugin content every time and reduce lifecycle events.

>Sakura UI is a bundled TeamCity plugin. You can see that in the Plugins menu: it's listed here under the name "Overview Plugin". The Sakura UI plugin registers few endpoints and provides JavaScript. So, from some perspective, Sakura UI is a controlled plugin itself.
>
{type="tip"}

The plugin becomes a _controlled_ when you add a handler to the `ON_CONTEXT_UPDATE` lifecycle hook:

```java

get controlled() {
  return (
    this.callbacks.ON_CONTEXT_UPDATE.length > 0 || isValidPluginReactElementType(this?.content)
  )
}

```

Let's start with the controller:

```java

public class SakuraUIPluginController {
    private static final String PLUGIN_NAME = "SakuraUI-Plugin";

    public SakuraUIPluginController(
            @NotNull PluginDescriptor descriptor,
            @NotNull PagePlaces places
    ) {
        new SimplePageExtension(places, PlaceId.SAKURA_HEADER_NAVIGATION_AFTER, PLUGIN_NAME, descriptor.getPluginResourcesPath("controlled-plugin.jsp"))
                .addCssFile("controlled-plugin.css")
                .addJsFile("controlled-plugin-core.js")
                .register();
    }
}
```

The code is almost the same to the [basic plugin v.1](basic-ui-plugins.md#Version+1.+Simple+plugin). We've only explicitly added the JS file `controlled-plugin-core.js` with the following content:

```jsp

<%@ include file="/include.jsp" %>

<bs:linkScript> // 1
  ${teamcityPluginResourcesPath}controlled-plugin-jsp.js
</bs:linkScript>

<div class="controlled-plugin-wrapper">Here is a controlled plugin.</div>

```

The `<bs:linkScript>` helper generates a correct link to the script file (including `base_url`). You can load the scripts via the Java Controller or in the JSP file. Files registered in the Java Controller should be requested before any other JavaScript file listed in the JSP.

>In 2020 EAP1 build, the script loading order is not correct. The scripts files provided in the JSP via `<bs:linkScript>` are getting loaded before the JavaScript files specified in Java Controller. This logic will be inverted in the next EAP builds.
>
{type="warning"}

`controlled-plugin-core.js`:

```jsp

(() => {
    console.log("My Controlled plugin script from a Core file");

    const plugin = TeamcityReactAPI.pluginRegistry.searchByPlaceId("SAKURA_HEADER_NAVIGATION_AFTER", "SakuraUI-Plugin") // 1

    const template = (context) => `<div class="controlled-plugin-wrapper">Here is a dummy plugin.${JSON.stringify(context)}</div>` // 2

    plugin.onContextUpdate((context) => { // 3
        plugin.replaceContent(template(context))
    })
})()
```

1. Using `pluginRegistry`, we find the pugin instance by specifying `PlaceID` and the plugin name.
2. We prepare the ES6 template.
3. We subscribe the plugin to the context update event. The subscription handler provides the last context as the first argument. Whenever the context changes, we ask the plugin to update the content with the new string generated at point 3.

>As you can see, we use Arrow functions. Since 2020.2, TeamCity officially drops support for Internet Explorer, so you can safely use ES6 features.
>
{type="tip"}

`controlled-plugin-jsp.js`:

```js

(() => {
    console.log("My Controlled plugin script from a JSP file");

    const plugin = TeamcityReactAPI.pluginRegistry.searchByPlaceId("SAKURA_HEADER_NAVIGATION_AFTER", "SakuraUI-Plugin")

    const template = (context) => `<div class="controlled-plugin-wrapper">Here is a dummy plugin.${JSON.stringify(context)}</div>`

    plugin.onContextUpdate((context) => {
        plugin.replaceContent(template(context))
    })
})()
```

The same content goes to the `controlled-plugin-core.js`, but it has a different `console.log` at the top of IIFE (Immediately Invoked Function Expression).

Let’s build a plugin and look at this behavior:

<img src="controlled-plugin-1.png" animated="true"/>

First of all, the console now looks a little different:

<img src="controlled-plugin-2.png"/>

During the `ON_CREATE` phase, Plugin Wrapper starts loading all attached scripts and styles. Script loading is an asynchronous function, so all scripts getting loaded after the synchronous `ON_MOUNT`. We used two JavaScript files, both added a subscription. That's why we see 2 subscription lines after ON_MOUNT:

_Plugin debugging. SakuraUI-Plugin / SAKURA_HEADER_NAVIGATION_AFTER. Subscribe to Lifecycle._

Then we go to navigation. Notice the major difference between the basic and controlled plugins here: Plugin Wrapper never recreate plugins from a scratch (`ON_CREATE`, `ON_DELETE`). Instead, it updates the content accordingly the new context. We used the Plugin API method `replaceContent` two times, and we also received a notification that the content has been updated two times. 

If a component should disappear (for example, you develop a plugin not for the header but for `SAKURA_PROJECT_BEFORE_CONTENT`), it will be properly dismounted.   
Let's change all `PlaceID`'s used in this plugin from `SAKURA_HEADER_NAVIGATION_AFTER` to `SAKURA_PROJECT_BEFORE_CONTENT`.

<img src="controlled-plugin-3.png"/>

Here we have a plugin in the new `PlaceID`. Every time we navigate, we also face the `ON_UNMOUNT` lifecycle event. Unmount happens every time the plugin should disappear from the screen. Or, in terms of frontend, Unmount happens whenever you remove a node from the DOM.

You can subscribe to any lifecycle hooks with the Plugin API:

* `onMount` - element is mounted to the DOM.
* `onContentUpdate` - plugin content has been updated via the `replaceContent` method.
* `onContextUpdate` - PluginUI context has been updated.
* `onUnmount` - element is unmounted.
* `onDelete` - plugin is completely removed from the memory.

The newly added handler for `ON_MOUNT` event fires at least one time, even if `ON_MOUNT` has been triggered before.   
This hook is a good place to add event listeners because it fires right after the moment the content has been applied to the DOM tree. Try to add this code to the plugin's JS:

```js

plugin.onMount((context) => {
   plugin.registerEventHandler(plugin.container, "click", () => {
      console.log("Clicked!")
   });
});
```

Here, we use the Plugin API method `registerEventHandler` to add the click handler. We recommend using this method because it guarantees that the handler will be detached before `ON_UNMOUNT` and attached again next time the component is `ON_MOUNT`.

That is what you get:

<img src="controlled-plugin-4.png"/>

As you see, right before the component is unmounted, Plugin Wrapper removes all handlers.

Every lifecycle event subscription takes a handler as an argument. This handler receives the current context. Every subscription returns an unsubscribe function, so if you do not want to keep receiving updates, just call the unsubscriber:

```js

const unsubscribe = plugin.onContextUpdate((context) => {
    plugin.replaceContent(template(context))
});

// After a while    
unsubscribe();

```

We've learned how the lifecycle hooks work and how to consume them. Using this API, you can control async operations and event handling. This API is sufficient to build any complicated plugins for TeamCity. 

In [this section](spa-ui-plugins.md) you can read about React-based plugins, learn more about internal plugin mechanisms and how to reuse ready components in your plugin.
