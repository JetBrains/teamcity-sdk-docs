[//]: # (title: Front-End Extensions)
[//]: # (auxiliary-id: Front-End+Extensions.html)

Initially, we shared our view on the TeamCity plugins and the motivation behind revising our plugin development approach in the [dedicated blog post]().   
This document explains the new way of the plugin development in TeamCity. The new plugin system lets you write any sophisticated plugin and integrate it both in the [experimental UI](https://www.jetbrains.com/help/teamcity/teamcity-experimental-ui.html) (code-named Sakura) and classic UI.   
In addition to the previous plugin development workflow, we made a great improvement, concentrating on the front-end aspects of the plugin development.

>
This document is currently in the draft stage. We introduce the new plugin development approach in terms of our [Early Access Program](https://confluence.jetbrains.com/display/TW/TeamCity+EAP) (EAP) for TeamCity 2020.2. The new API will be improved constantly during the next few months. By the time of 2020.2 release, some API will have been changed. We will warn about any changes in this guide and in the relevant repositories. Feel free to report any known issues in [YouTrack](https://youtrack.jetbrains.com/issues/TW?q=tag:%20SakuraUI-Plugins%20) using the `SakuraUI-plugins` tag.   
Although we are working on reducing boilerplate, currently we are concentrated on things like API stability, testing, and clear documentation.
>
{type="note"}

We guarantee that previously written plugins will work as they worked before 2020.2 EAP. New plugins will work starting from TeamCity 2020.2 EAP1.

Before starting, you have to prepare your environment. The entire preparation comprises 6 steps that are explained [here](getting-started-with-plugin-development.md).

## Key benefits

Key features of the new plugin development approach:
1. There is a way to integrate plugins to the Sakura UI and classic UI.
2. All existing plugins continue to work as they worked before 2020.2.
3. UI plugins could be written in a more frontend-centric way, which involves modern web techs.
4. UI plugins are framework-agnostic, so you can use any library, framework, and bundler.
5. For those who prefer React, we expose our internal components, so you can write a plugin composing existing components from the Sakura UI – not only by writing everything yourself.

## Terminology

`PlaceID` – ID of a containter that will comprise the plugin. Since 2020.2 EAP1, we provide a new set of PlaceIDs, which have a prefix `SAKURA_`. For example, `SAKURA_HEADER_RIGHT`. The full list of `PlaceID`'s is available in the NPM module `@teamcity/react-api`.   
This list is not final. We are open to get [feedback](https://confluence.jetbrains.com/display/TW/Feedback) about your expectations and needs.

The following `PlaceID`'s work both in the Sakura and classic UI:
* `SAKURA_BEFORE_CONTENT`
* `SAKURA_PROJECT_BEFORE_CONTENT`
* `SAKURA_BUILD_BEFORE_CONTENT`
* `SAKURA_HEADER_NAVIGATION_AFTER`
* `SAKURA_HEADER_USERNAME_BEFORE`
* `SAKURA_HEADER_RIGHT`
* `SAKURA_FOOTER_RIGHT`

Some of `PlaceID`'s are available only in the Sakura UI:
* `SAKURA_SIDEBAR_TOP`
* `SAKURA_PROJECT_TRENDS`
* `SAKURA_PROJECT_BUILDS`
* `SAKURA_BUILD_CONFIGURATION_TREND_CARD`
* `SAKURA_BUILD_CONFIGURATION_BEFORE_CONTENT` \*
* `SAKURA_BUILD_CONFIGURATION_BUILDS`
* `SAKURA_BUILD_CONFIGURATION_BRANCHES`
* `SAKURA_BUILD_LINE_EXPANDED` \*

\* These `PlaceID`'s will be added to the classic UI later.

* `PluginUIContext` – context object which represents the plugin location. It contains currently selected `projectId`, `buildId`, `buildTypeId`, `agentId`, `agentPoolId`, `agentTypeId`. TeamCity guarantees that each plugin will receive the latest context.

* _Plugin lifecycle_ – set of events, each of them is invoked when the plugin content passes a certain step. For example, _the plugin is mounted to a DOM_, _context is updated_, _plugin is unmounted_.

* _Plugin wrapper_ – Sakura UI entity, a React component which manages a certain `PlaceID`. It reacts on the plugin UI context changes, passes updates to a plugin and manages plugin lifecycles.

* `TeamсityReactAPI` – publicly exposed JS toolset which helps developers to manage plugins.

* _Plugin registry_ – JavaScript object which stores data about each rendered plugin. This register could be used to search, retrieve, and remove plugins.

* _Plugin constructor_ – JavaScript prototype used to create a plugin instance.

* `Basic plugin` – simplest plugin. Its behaviour and reaction to the `PluginUIContext` are defined implicitly. It re-renders automatically every time `PluginUIContext` updates.

* _Controlled plugin_ – plugin which behaving and reaction on `PluginUIContext` updates explicitly defined by a developer.

* _Development mode_ – special mode which shows `PlaceID` containers in DOM and makes the plugin write debug information in console. Accessible via the GET property `pluginDevelopmentMode=true`.

## Public API Reference

`window.TeamcityReactAPI` is a set of handy tools, which will help you retrieve, update and manipulate plugins. It consists of:

* _Components_ - set of React components, we are ready to expose. Right now there is only the one component AllBuilds. We are looking forward for your feedback on this.
* _React_ - exposed React library. It’s vital to use the same React library version to integrate your Plugin into the TeamCity React vDOM tree. The full explanation is in the React Plugin section.
* _ReactDOM_ - exposed ReactDOM library. It’s vital to use the same React library version to integrate your Plugin into the TeamCity React vDOM tree. The full explanation is in the React Plugin section.
* _utils_- set of utilities
  * `requestJSON` - function to request and parse a JSON from the Server. It already contains all the headers for the request and automatically parses the response.
  * `requestTEXT` - function to request and parse a TEXT from the Server. It already contains all the headers for the request and automatically parses the response.
* _Plugin_ - plugin constructor. It expects you to specify `PlaceID` and content options as arguments. We will look at this constructor later in the Advanced Plugins section.
* `pluginRegistry` - plugin Registry, which you could use to find a certain instance of your plugin.

Before starting the development, please checkout [this repository](https://github.com/JetBrains/teamcity-sakura-ui-plugins). It will help you to avoid tons of a boilerplate Java code.

## Types of Plugins

There are three types of plugins you can write with the new API:
* _Basic plugins_ rely on a simple JSP / HTML code that is requested automatically on every reload of a UI page.
* _Controlled plugins_ use more sophisticated logics and adjustable behavior. 
* _(Single-page Application) SPA plugins_ based on React.

You can read an introduction describing these types in our [blog post]().

The following documentation sections contain guides on how to create a plugin of each type:

* [Basic plugins](basic-ui-plugins.md)
* [Controlled plugins](controlled-ui-plugins.md)
* [SPA plugins](spa-ui-plugins.md)