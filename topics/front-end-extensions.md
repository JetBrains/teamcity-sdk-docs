[//]: # (title: Front-End Extensions)
[//]: # (auxiliary-id: Front-End+Extensions.html)

Initially, we shared our view on the TeamCity plugins and the motivation behind revising our plugin development approach in the [dedicated blog post](https://blog.jetbrains.com/teamcity/2020/09/teamcity-2020-2-updated-plugin-development).
   
This document explains the new way of the plugin development in TeamCity. The updated plugin system lets you write any sophisticated plugin and integrate it both in the [experimental UI](https://www.jetbrains.com/help/teamcity/teamcity-experimental-ui.html) (code-named Sakura) and classic UI. There is also a [Workshop](https://www.youtube.com/watch?v=-oa_8WLYFnE) from the TeamCity Technology Day.   

In addition to the previous plugin development workflow, we made a significant improvement, concentrating on the front-end aspects of the plugin development.

<note>

Since the release of TeamCity 2020.2, we strive to keep the Plugin API consistent and compatible across different TeamCity versions (2020.2 and further). If you find any issue, please, report them in [YouTrack](https://youtrack.jetbrains.com/issues/TW?q=tag:%20SakuraUI-Plugins%20) using the `SakuraUI-plugins` tag.
  
</note>

We guarantee that previously written plugins will work as they worked before 2020.2 EAP. New plugins will work starting from TeamCity 2020.2.

### Useful links
- [Prepare environment](getting-started-with-plugin-development.md) – instruction on how to get ready to plugin development
- [TeamCity 2020.2: updated Plugin Development](https://blog.jetbrains.com/teamcity/2020/09/teamcity-2020-2-updated-plugin-development) – a blog post with Plugin Development Overview
- [TeamCity Technology Day 2020: Creating Plugins Using the New TeamCity Plugin UI Framework](https://www.youtube.com/watch?v=-oa_8WLYFnE) – an online workshop with the Plugin Ecosystem overview and live coding session
- [Feedback and issue reporting](https://youtrack.jetbrains.com/issues/TW?q=tag:%20SakuraUI-Plugins%20)
- [Demo Plugin repository](https://github.com/JetBrains/teamcity-sakura-ui-plugins) – a repo with 5 dedicated branches, explaining Basic, Controlled, and React plugins
- [NPM module](https://github.com/JetBrains/teamcity-api-js) – a Node Package Manager module suitable to build rich UI plugins
- [Explanation how plugins are loaded](spa-ui-plugins.md#How+Plugins+are+loaded)

## Key benefits

Key features of the new plugin development approach:
* There is a way to integrate plugins both to the Sakura UI and the Classic UI.
* All existing plugins continue to work as they worked before 2020.2.
* UI plugins could be written in a more frontend-centric way, which involves modern web techs.
* UI plugins are framework-agnostic, so you can use any library, framework, and bundler.
* For those who prefer React, we expose our internal components, so you can write a plugin composing existing components from the Sakura UI – not only by writing everything yourself.

## Terminology

`PlaceID` – ID of a container that will render the plugin. Since 2020.2, we provide a new set of PlaceIDs, which have a prefix `SAKURA_`. For example, `SAKURA_HEADER_RIGHT`. The full list of SAKURA `PlaceID`'s is available in the NPM module `@jetbrains/teamcity-api`. Those PlaceIDs are accessible only in the Sakura UI. To integrate your Frontend Plugin in the Classic UI, use the classic PlaceID (the full list is available in TeamCity OpenAPI definitions).

This list is not final. We are open to get the [feedback](https://confluence.jetbrains.com/display/TW/Feedback) about your expectations and needs.

The following `PlaceID`'s work both in the Sakura and classic UI:
* `ALL_PAGES_FOOTER_PLUGIN_CONTAINER` - a specific PlaceID to place plugin JavaScripts to
* `SAKURA_HEADER_NAVIGATION_AFTER`
* `SAKURA_HEADER_USERNAME_BEFORE`
* `SAKURA_HEADER_RIGHT`

Some of `PlaceID`'s are available only in the Sakura UI:
* `SAKURA_FOOTER_RIGHT`
* `SAKURA_BEFORE_CONTENT`
* `SAKURA_SIDEBAR_TOP`
* `SAKURA_PROJECT_TRENDS`
* `SAKURA_BUILD_CONFIGURATION_TREND_CARD`
* `SAKURA_PROJECT_BUILDS`
* `SAKURA_BUILD_CONFIGURATION_BUILDS`
* `SAKURA_BUILD_CONFIGURATION_BRANCHES`
* `SAKURA_BUILD_LINE_EXPANDED`
* `SAKURA_BUILD_OVERVIEW`

_PluginUIContext_ – context object which represents the plugin location. It contains currently selected `projectId`, `buildId`, `buildTypeId`, `agentId`, `agentPoolId`, `agentTypeId`. TeamCity guarantees that each plugin will receive the latest context.

_Plugin Lifecycle_ – set of events, each of them is invoked when the plugin content passes a certain step. For example, _the plugin is mounted to a DOM_, _context is updated_, _plugin is unmounted_.

_Plugin Wrapper_ – Sakura UI entity, a React component which manages a certain `PlaceID`. It reacts on the plugin UI context changes, passes updates to a plugin and manages plugin lifecycles.

_TeamCityAPI_ – [publicly exposed](https://www.npmjs.com/package/@jetbrains/teamcity-api) JS toolset which helps developers to manage plugins. Available as an NPM module and as a global JS variable (`window.TeamCityAPI`).

_Plugin Registry_ – JavaScript object which stores data about each rendered plugin. This register should be used to search, retrieve, and remove plugins.

_Plugin Constructor_ – JavaScript prototype used to create a plugin instance.

_Basic plugin_ – the simplest plugin. Its behavior and reaction to the `PluginUIContext` are defined implicitly. It re-renders automatically every time `PluginUIContext` updates.

_Controlled plugin_ – plugin which behavior and reaction on `Plugin Lifecycle` are defined by the developer explicitly.

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

Before starting the development, please checkout [this demo repository](https://github.com/JetBrains/teamcity-sakura-ui-plugins). It will help you to avoid tons of a boilerplate Java code.

## Types of Plugins

There are three types of plugins you can write with the new API:
* _Basic plugins_ rely on a simple JSP/HTML code that is requested automatically on every navigation event, for example, when a user switches between projects or build configurations or moves to a new page.
* _Controlled plugins_ – using this type, a developer explicitly defines, how the Plugin should react to the navigation events, for example, define the Context Update handler, add / remove custom event listeners.
* _(Single-page Application) SPA plugins_ are "controlled plugins" on steroids, which use React under the hood and offer you to use shared Components and Libraries.

The following documentation sections contain more detailed explanations and guides on how to create a plugin of each type:

* [Basic plugins](basic-ui-plugins.md)
* [Controlled plugins](controlled-ui-plugins.md)
* [SPA plugins](spa-ui-plugins.md)

See also [Basic vs. controlled plugins](basic-ui-plugins.md#Basic+vs.+controlled+plugins).
