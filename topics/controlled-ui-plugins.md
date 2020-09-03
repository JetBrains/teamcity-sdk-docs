[//]: # (title: Controlled UI Plugins)
[//]: # (auxiliary-id: Controlled+UI+Plugins.html)

This guide explains how to create a basic UI plugin based on the new [front-end extensions](front-end-extensions.md) paradigm.

Source branch with an example project: [example/controlled-plugin](https://github.com/JetBrains/teamcity-sakura-ui-plugins/tree/example/controlled-plugin).

Among the TeamCity Frontend team we call those plugins “Controlled plugins”, because this name explains the main advantage - a developer controls the Plugin behaviour. The Controlled Plugin knows how to react on the lifecycle events; It uses Plugin API to update its’ content, to subscribe and unsubscribe on events and to abort requests. In other words, Controlled Plugins offer an opportunity to write rich Applications within the TeamCity UI. And, in other hand, when Plugin Wrapper does know, that a Plugin is controlled by a developer, it stops requesting plugin content every time and reduce lifecycle events.

*Hint: *do you know, that the Sakura UI itself is a bundled TeamCity plugin? You can see that in the Plugins menu: it’s listed here under the name “Overview Plugin”. The Sakura UI plugin registers few endpoints and provides JavaScript. So, from some perspective, under the hood the Sakura UI is a controlled plugin. 

*What makes the plugin Controlled? *

Look at the line, which checks controlled attribute:

```java
get controlled() {
  return (
    this.callbacks.ON_CONTEXT_UPDATE.length > 0 || isValidPluginReactElementType(this?.content)
  )
}

```

The Plugin becomes a Controlled Plugin in time you add a handler to ON_CONTEXT_UPDATE lifecycle hook. Or if you pass the React Component as a content. We will get to the React later. For now let’s have a look at the source code.

As usual, we start with the controller.

```java

public class SakuraUIPluginController {
    private static final String PLUGIN_NAME = "SakuraUI-Plugin";

    public SakuraUIPluginController(
            @NotNull PluginDescriptor descriptor,
            @NotNull PagePlaces places
    ) {
        new SimplePageExtension(places, PlaceId.SAKURA_HEADER_NAVIGATION_AFTER, PLUGIN_NAME, descriptor.getPluginResourcesPath("advanced-plugin.jsp"))
                .addCssFile("advanced-plugin.css")
                .addJsFile("advanced-plugin-core.js")
                .register();
    }
}
```

The code is almost the same to the Basic Plugin v.1. We’ve just explicitly added JS file “advanced-plugin-core.js”. 

There is a advanced-plugin.jsp content:

<%@ include file="/include.jsp" %>

<bs:linkScript> // 1
  ${teamcityPluginResourcesPath}advanced-plugin-jsp.js
</bs:linkScript>

<div class="advanced-plugin-wrapper">Here is a controlled plugin.</div>

Here we use the <bs:linkScript> helper, which generates a correct link to a script file (including base_url). Actually, it’s not really important how to load the scripts - via the Java Controller or in JSP file. The only thing we are confident about - files registered in the Java Controller should be requested before any other JavaScript file listed in JSP. 

*Warning: *after the EAP 1 has been released, we figured out, that the script loading order is not correct. In current version, the scripts files provided in the JSP via the <bs:linkScript> are getting loaded before the JavaScript files, specified in Java Controller. This logics will be inverted in coming releases.

advanced-plugin-core.js

(() => {
    console.log("My Controlled plugin script from a Core file");

    const plugin = TeamcityReactAPI.pluginRegistry.searchByPlaceId("SAKURA_HEADER_NAVIGATION_AFTER", "SakuraUI-Plugin") // 1

    const template = (context) => `<div class="controlled-plugin-wrapper">Here is a dummy plugin.${JSON.stringify(context)}</div>` // 2

    plugin.onContextUpdate((context) => { // 3
        plugin.replaceContent(template(context))
    })
})()

1. Using the pluginRegistry we find Plugin instance by specify Place ID and Plugin Name;
2. We prepare the ES6 template;
3. We subscribe plugin to the context update event. Subscription handler provides the last context as a first argument. Whenever context changes, we ask plugin to update the content with a new string, generated at point 3;

*Note: *as you can see, we use Arrow functions. Since 2020.2 TeamCity officially doesn’t support Internet Explorer, so you can use ES6 features. 

advanced-plugin-jsp.js

(() => {
    console.log("My Controlled plugin script from a JSP file");

    const plugin = TeamcityReactAPI.pluginRegistry.searchByPlaceId("SAKURA_HEADER_NAVIGATION_AFTER", "SakuraUI-Plugin")

    const template = (context) => `<div class="controlled-plugin-wrapper">Here is a dummy plugin.${JSON.stringify(context)}</div>`

    plugin.onContextUpdate((context) => {
        plugin.replaceContent(template(context))
    })
})()

The same content to the advanced-plugin-core.js, but it has a different console.log at the top of IIFE (Immediately Invoked Function Expression).

Let’s build a plugin and look at this behavior:

[Image: Screen Recording 2020-09-02 at 01.48.34.mov]First of all, console now looks a little different:
[Image: image.png]During the ON_CREATE phase Plugin Wrapper starts loading all attached scripts and styles. Script loading is an asynchronous function, so all scripts getting loaded after the synchronous ON_MOUNT. We used two JavaScript files, both added a subscription. That’s why we see 2 subscription lines after ON_MOUNT:


Plugin debugging. SakuraUI-Plugin / SAKURA_HEADER_NAVIGATION_AFTER. Subscribe to Lifecycle.


Then we start navigation. Look at the major difference between Basic and Controlled plugins here: Plugin Wrapper never re-create plugins from a scratch (ON_CREATE, ON_DELETE). Instead, it updates the content accordingly the new Context. As long as we used Plugin API method “replaceContent” two times, we also receive a notification, that content has been updated two times. 

If component should disappear (for example, you develop a plugin not for the Header, but for the SAKURA_PROJECT_BEFORE_CONTENT), it will be properly dismounted. Let’s change all PlaceID used in this Plugin from SAKURA_HEADER_NAVIGATION_AFTER to SAKURA_PROJECT_BEFORE_CONTENT.

[Image: image.png]Here we have a plugin in a new PlaceID and every time we navigate, we also face the ON_UNMOUNT lifecycle event. Unmount happens every time, the Plugin should disappear from a screen. Or, in terms of Frontend, Unmount happens whenever you remove a Node from the DOM.

You can subscribe to any lifecycle hooks with Plugin API:

*onMount* - element is mounted to the DOM
*onContentUpdate* - plugin content has been updated via replaceContent method
*onContextUpdate* - PluginUI Context has been updated
*onUnmount* - element is unmounted
*onDelete* - plugin completely removed from the memory

*A little more about onMount:*
The newly added handler for ON_MOUNT event fires at least one time, even if the ON_MOUNT has been triggered before.
This hook is a good place to add event listeners, because it fires right after the moment the content has been applied to the DOM tree. Try to add this code to Plugin’s JS:

plugin.onMount((context) => {
   plugin.registerEventHandler(plugin.container, "click", () => {
      console.log("Clicked!")
   });
});

Here we use Plugin API method “registerEventHandler” to add the click handler. We recommend to use this method, because it guarantees, that handler will be detached before ON_UNMOUNT and attached again next time the component is ON_MOUNT.

That is what you get:
[Image: image.png]As you see, right before the Component will be unmounted, Plugin Wrapper removes all handlers.

Every lifecycle event subscription takes a handler as an argument. This handler receives current context. And every subscription returns an unsubscribe function, so if you do not want to keep receiving updates anymore, you just call the unsubscriber:

const unsubscribe = plugin.onContextUpdate((context) => {
    plugin.replaceContent(template(context))
});

// After a while    
unsubscribe();

We’ve learned how Lifecycle hooks work and how to consume them. Using this API you can control async operations and event handling. This API is sufficient to build any complicated plugins for the TeamCity. In the next chapter we will find out how to write React Plugins, will research something about internal mechanisms, and will learn how to reuse components developed in JetBrains.

Before we move further, you can spend a little time to examine Plugin lifecycle. It’s a good time to play around with navigation and look, how different TeamCity UI Plugins react on Context Update. Thank you!
