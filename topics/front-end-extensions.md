[//]: # (title: Front-End Extensions)
[//]: # (auxiliary-id: Front-End+Extensions.html)

<warning>

This document is introduced in terms of [Early Access Program](https://confluence.jetbrains.com/display/TW/TeamCity+EAP) (EAP) for TeamCity 2020.2.

</warning>

Initially, we shared our view on the TeamCity plugins and the motivation behind revising our plugin development approach in the [dedicated blog post](https://blog.jetbrains.com/teamcity/2020/09/teamcity-2020-2-updated-plugin-development).
   
This document explains the new way of the plugin development in TeamCity. The updated plugin system lets you write any sophisticated plugin and integrate it both in the [experimental UI](https://www.jetbrains.com/help/teamcity/teamcity-experimental-ui.html) (code-named Sakura) and classic UI.   

In addition to the previous plugin development workflow, we made a significant improvement, concentrating on the front-end aspects of the plugin development.

<note>

This document is currently in the draft stage. We introduce the new plugin development approach in terms of our 2020.2 EAP. The new API will be improved constantly during the next few months. By the time of 2020.2 release, some API will have been changed. We will warn about any changes in this guide and in the relevant repositories. Feel free to report any known issues in [YouTrack](https://youtrack.jetbrains.com/issues/TW?q=tag:%20SakuraUI-Plugins%20) using the `SakuraUI-plugins` tag.
   
Although we are working on reducing boilerplate, currently we are concentrated on things like API stability, testing, and clear documentation.

</note>

We guarantee that previously written plugins will work as they worked before 2020.2 EAP. New plugins will work starting from TeamCity 2020.2 EAP1.

Before starting, you have to prepare your environment. The entire preparation comprises 6 steps that are explained [here](getting-started-with-plugin-development.md). We also recommend you to checkout the [Demo Plugin repository](https://github.com/JetBrains/teamcity-sakura-ui-plugins)

## Key benefits

Key features of the new plugin development approach:
* There is a way to integrate plugins to the Sakura UI and classic UI.
* All existing plugins continue to work as they worked before 2020.2.
* UI plugins could be written in a more frontend-centric way, which involves modern web techs.
* UI plugins are framework-agnostic, so you can use any library, framework, and bundler.
* For those who prefer React, we expose our internal components, so you can write a plugin composing existing components from the Sakura UI – not only by writing everything yourself.

## Terminology

`PlaceID` – ID of a container that will comprise the plugin. Since 2020.2 EAP1, we provide a new set of PlaceIDs, which have a prefix `SAKURA_`. For example, `SAKURA_HEADER_RIGHT`. The full list of `PlaceID`'s is available in the NPM module `@jetbrains/teamcity-api`.   
This list is not final. We are open to get the [feedback](https://confluence.jetbrains.com/display/TW/Feedback) about your expectations and needs.

The following `PlaceID`'s work both in the Sakura and classic UI:
* `SAKURA_BEFORE_CONTENT`
* `SAKURA_PROJECT_BEFORE_CONTENT`
* `SAKURA_BUILD_BEFORE_CONTENT`
* `SAKURA_HEADER_NAVIGATION_AFTER`
* `SAKURA_HEADER_USERNAME_BEFORE`
* `SAKURA_HEADER_RIGHT`
* `SAKURA_FOOTER_RIGHT`
* `SAKURA_BUILD_CONFIGURATION_BEFORE_CONTENT` \*
* `SAKURA_AGENTS_OVERVIEW_BEFORE_CONTENT` \*
* `SAKURA_AGENTS_UNAUTHORIZED_BEFORE_CONTENT` \*
* `SAKURA_AGENT_BEFORE_CONTENT` \*

Some of `PlaceID`'s are available only in the Sakura UI:
* `SAKURA_SIDEBAR_TOP`
* `SAKURA_PROJECT_TRENDS`
* `SAKURA_PROJECT_BUILDS`
* `SAKURA_BUILD_CONFIGURATION_TREND_CARD`
* `SAKURA_BUILD_CONFIGURATION_BUILDS`
* `SAKURA_BUILD_CONFIGURATION_BRANCHES`
* `SAKURA_BUILD_LINE_EXPANDED`
* `SAKURA_AGENT_CLOUD_IMAGE_BEFORE_CONTENT` \*
* `SAKURA_AGENT_POOL_BEFORE_CONTENT` \*

PlaceIDs marked with Asterisk (\*) will be available starting the TeamCity 2020.2 EAP 3

`PluginUIContext` – context object which represents the plugin location. It contains currently selected `projectId`, `buildId`, `buildTypeId`, `agentId`, `agentPoolId`, `agentTypeId`. TeamCity guarantees that each plugin will receive the latest context.

_Plugin Lifecycle_ – set of events, each of them is invoked when the plugin content passes a certain step. For example, _the plugin is mounted to a DOM_, _context is updated_, _plugin is unmounted_.

_Plugin Wrapper_ – Sakura UI entity, a React component which manages a certain `PlaceID`. It reacts on the plugin UI context changes, passes updates to a plugin and manages plugin lifecycles.

_TeamCityAPI_ – [publicly exposed](https://www.npmjs.com/package/@jetbrains/teamcity-api) JS toolset which helps developers to manage plugins.

_Plugin Registry_ – JavaScript object which stores data about each rendered plugin. This register could be used to search, retrieve, and remove plugins.

_Plugin Constructor_ – JavaScript prototype used to create a plugin instance.

_Basic plugin_ – simplest plugin. Its behavior and reaction to the `PluginUIContext` are defined implicitly. It re-renders automatically every time `PluginUIContext` updates.

_Controlled plugin_ – plugin which behavior and reaction on `PluginUIContext` updates are explicitly defined by a developer.

_Development mode_ – special mode which shows `PlaceID` containers in DOM and makes the plugin write debug information in the console. Accessible via the `GET` property `pluginDevelopmentMode=true`.

## Public API Reference

`window.TeamCityAPI` is a set of handy tools, which will help you retrieve, update, and manipulate plugins. It consists of:

* _Components_ - internal TeamCity React components we are ready to expose. Right now, there is only the one component `AllBuilds`. We are looking forward to your [feedback](https://confluence.jetbrains.com/display/TW/Feedback) on this.
* _React_ - exposed React library. It's vital to use the same React library version to integrate your plugin into the TeamCity React vDOM tree (see the [full explanation](spa-ui-plugins.md)).
* _ReactDOM_ - exposed ReactDOM library. It's vital to use the same React library version to integrate your plugin into the TeamCity React vDOM tree (see the [full explanation](spa-ui-plugins.md)).
* _utils_- set of utilities:
  * `requestJSON` - function to request and parse a JSON from the server. It already contains all the headers for the request and automatically parses the response.
  * `requestTEXT` - function to request and parse a TEXT from the server. It already contains all the headers for the request and automatically parses the response.
* _Plugin_ - plugin constructor. It expects you to specify `PlaceID` and content options as arguments (read more about [controlled plugins](controlled-ui-plugins.md)).
* `pluginRegistry` - plugin registry which you could use to find a certain instance of your plugin.

Before starting the development, please checkout [this repository](https://github.com/JetBrains/teamcity-sakura-ui-plugins). It will help you to avoid tons of a boilerplate Java code.

## Types of Plugins

There are three types of plugins you can write with the new API:
* _Basic plugins_ rely on a simple JSP/HTML code that is requested automatically on every navigation event.
* _Controlled plugins_ - using this type, a developer explicitly defines, how the Plugin should react on the navigation events. 
* _(Single-page Application) SPA plugins_ are extended "controlled plugins", which use React under the hood and offer you to use shared Components and Libraries

The following documentation sections contain more detailed explanations and guides on how to create a plugin of each type:

* [Basic plugins](basic-ui-plugins.md)
* [Controlled plugins](controlled-ui-plugins.md)
* [SPA plugins](spa-ui-plugins.md)

See also [Basic vs. controlled plugins](basic-ui-plugins.md#Basic+vs.+controlled+plugins).
