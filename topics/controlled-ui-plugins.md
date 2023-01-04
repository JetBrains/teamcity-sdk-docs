[//]: # (title: Controlled UI Plugins)
[//]: # (auxiliary-id: Controlled+UI+Plugins.html)

This guide explains how to create a controlled UI plugin based on the new [front-end extensions](front-end-extensions.md) paradigm.

__Source branch with the example project: [example/controlled-plugin](https://github.com/JetBrains/teamcity-sakura-ui-plugins/tree/example/controlled-plugin)__.

The name _controlled_ explains the main advantage of these plugins – a developer controls the plugin behavior. A controlled plugin knows how to react on the lifecycle events. It uses the Plugin API to update its content, to subscribe and unsubscribe to events, and to abort requests. In other words, controlled plugins allow creating rich applications within the TeamCity UI. Moreover, when the Plugin Wrapper knows that a plugin is controlled by a developer, it stops requesting the plugin content every time and reduce lifecycle events.

>Sakura UI is a bundled TeamCity plugin. You can see that in the Plugins menu: it's listed here under the name "Overview Plugin". The Sakura UI plugin registers few endpoints and provides JavaScript. So, from some perspective, Sakura UI is a controlled plugin itself.
>
{type="tip"}

## Composing controlled UI plugin

The main principle of Controlled plugins is that you load your JSP and JavaScripts into a hidden container (`PlaceId.ALL_PAGES_FOOTER_PLUGIN_CONTAINER`) and then, at the time of DOM Ready, you start manipulating the content using the JavaScript API. It's possible to convert your Basic plugin to the Controlled one by simply adding the subscription to `ON_CONTEXT_UPDATE`.

The `Plugin Wrapper` considers the plugin as a _controlled_ one, if it has an `ON_CONTEXT_UPDATE` handler or if it's a React Component. 

```javascript
isControlled() {
  return (
    this.callbacks.ON_CONTEXT_UPDATE.length > 0 || isValidPluginReactElementType(this?.content)
  )
}
```

Let's start with the Java Controller: 

```java
public class SakuraUIPluginController {
    private static final String PLUGIN_NAME = "SakuraUI-Plugin";

    public SakuraUIPluginController(
            @NotNull PluginDescriptor descriptor,
            @NotNull PagePlaces places
    ) {
        new SimplePageExtension(places, PlaceId.ALL_PAGES_FOOTER_PLUGIN_CONTAINER, PLUGIN_NAME, descriptor.getPluginResourcesPath("controlled-plugin.jsp"))
                .addCssFile("controlled-plugin.css")
                // There is an option to load the script using the Java Controller's 'addJsFile'.
                // We recommend to use the JSP based loading (see the controlled-plugin.jsp) though
                // to make it clearer where the script came from.
                .addJsFile("controlled-plugin-core.js")
                .register();
    }
}
```

The code is pretty close to the [basic plugin v.1](basic-ui-plugins.md#Version+1.+Simple+plugin). We've only changed `PlaceId` to `PlaceId.ALL_PAGES_FOOTER_PLUGIN_CONTAINER` and explicitly added the JS file `controlled-plugin-core.js`; the JSP now contains the next code:

```jsp

<%@ include file="/include.jsp" %>

<bs:linkScript> // 1
  ${teamcityPluginResourcesPath}controlled-plugin-jsp.js
</bs:linkScript>

<div class="controlled-plugin-wrapper">Here is a controlled plugin.</div>

```

Please note that we load two different JavaScript files using two approaches. The first one is to use `addJSFile` in the JavaController, the second one is to use `<bs:linkScript>` in JSP. The `<bs:linkScript>` helper generates a resolved path to the script file (including `base_url`).

There are no reasons to use one loader prior to other, except that .addJsFile() files are loaded and invoked before the content of a plugin is rendered. Apart from that, we at JetBrains consider using `<bs:linkScript>` as a clearer frontend-centric way of adding script files. 

`controlled-plugin-core.js`:

```javascript
(() => console.log("Controlled Plugin. Script invoked from the controlled-plugin-core.js"))()
```
`controlled-plugin-jsp.js`:

```javascript
(() => {
    console.log("Controlled Plugin. Script invoked from the controlled-plugin-jsp.js");
    const name = "SakuraUI-Plugin";
    const container = document.getElementById(name);

    const plugin =  new TeamCityAPI.Plugin(["SAKURA_BUILD_OVERVIEW", "BUILD_RESULTS_FRAGMENT"], {
        name,
        content: container,
        options: {debug:true},
    })

    const template = (context) => `<div>There is a location context: ${JSON.stringify(context)}</div>`

    plugin.onContextUpdate((context) => {
        container.classList.remove("hidden");
        const dynamicPart = container.querySelector(".js-dynamic-part");

        if (dynamicPart == null) {
            return;
        }

        while (dynamicPart.firstChild) {
            dynamicPart.firstChild.remove();
        }


        console.log("Controlled Plugin. On Context Update event fired.")
        dynamicPart.insertAdjacentHTML('afterbegin', template(context));
    })
})()

```

1. Using `TeamCityAPI`, we create the JavaScript plugin for two PlaceIds: Sakura UI and Classic UI respectively. We use the DOM element from the JSP as a container.
2. We prepare the ES6 template.
3. We subscribe the plugin to the context update event. The subscription handler provides the last context as the first argument. Whenever the context changes, we ask the plugin to update the content with the new string generated at point 3.

>As you can see, we use Arrow functions. Since 2020.2, TeamCity officially drops support for Internet Explorer, so you can safely use ES6 features.
>
{type="tip"}

Let’s build a plugin and look at this behavior:

<img src="controlled-plugin-1.png" width="1000" animated="true" alt="Plugin behavior"/>

First, the console now looks a little different:

<img src="controlled-plugin-2.png" thumbnail-same-file="true" thumbnail="true" alt="Plugin console inspection"/>

During the `ON_CREATE` phase, Plugin Wrapper starts loading all attached scripts and styles. Script loading is an asynchronous function, so all scripts getting loaded after the synchronous `ON_MOUNT`. 

After scripts are initialized, we call a subscription. That's why we see subscription lines after `ON_MOUNT`:

_Plugin debugging. SakuraUI-Plugin / SAKURA_BUILD_OVERVIEW. Subscribe to Lifecycle. ON_CONTEXT_UPDATE_ 

Now, if you try to select another build, you'll trigger the context update. There is a major difference between the basic and controlled plugins here: Plugin Wrapper never recreates controlled plugins from a scratch (skipping `ON_CREATE`, `ON_DELETE`). Instead, it updates the content accordingly to the new context using the `ON_CONTEXT_UPDATE` handlers. We used the Plugin API method `replaceContent`, so we also received a notification that the content has been updated.

<img src="controlled-plugin-3.png" thumbnail-same-file="true" thumbnail="true" alt="Plugin console inspection"/>

Every time we navigate, we also face the `ON_UNMOUNT` lifecycle event. Unmount happens every time the plugin should disappear from the screen. Or, in more specific terms of frontend, Unmount happens whenever you remove a node from the DOM.

You can subscribe to any lifecycle hooks with the Plugin API:

* `onMount` - element is mounted to the DOM.
* `onContentUpdate` - plugin content has been updated via the `replaceContent` method.
* `onContextUpdate` - PluginUI context has been updated.
* `onUnmount` - element is unmounted.
* `onDelete` - plugin is completely removed from the memory.

The newly added handler for `ON_MOUNT` event fires at least one time, even if `ON_MOUNT` has been triggered before.   
This hook is a good place to add event listeners because it fires right after the moment the content has been applied to the DOM tree. Try to add this code to the plugin's JS:

```javascript

plugin.onMount((context) => {
   plugin.registerEventHandler(plugin.container, "click", () => {
      console.log("Clicked!")
   });
});
```

Using the code above, we call the Plugin API method `registerEventHandler` to add the click handler. We recommend using this method because it guarantees that the handler will be detached before `ON_UNMOUNT` and attached again next time the component is `ON_MOUNT`. Hence, you can avoid memory leaks and dead callbacks.

That is what you get:

<img src="controlled-plugin-4.png" thumbnail-same-file="true" thumbnail="true" alt="Plugin console inspection"/>

As you see, right before the component is unmounted, Plugin Wrapper removes all handlers. Next time the Plugin will be mounted, all specified `ON_MOUNT` callbacks will be fired and the event listeners will be attached again.

Every lifecycle event subscription takes a handler as an argument. This handler receives the current context. Every subscription returns an unsubscribe function, so if you do not want to keep receiving updates, just call the unsubscriber:

```javascript

const unsubscribe = plugin.onContextUpdate((context) => {
    plugin.replaceContent(template(context))
});

// After a while    
unsubscribe();
```

We've learned how the lifecycle hooks work and how to consume them. Using this API, you can control async operations and event handling. This API is sufficient to build any complicated plugins for TeamCity. This is a path you can follow to build Plugin using your favorite UI Framework / Library. 

But if you do prefer React, we have a little more for you. In [this section](spa-ui-plugins.md) you can read about React-based plugins, learn more about internal plugin mechanisms and how to reuse our components in your plugin.
