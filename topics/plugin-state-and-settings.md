[//]: # (title: Storing Plugin State and Settings)
[//]: # (auxiliary-id: Storing+Plugin+State+and+Settings.html)

Most of the plugins should have some place where they can store their state and settings.
By state we understand current internal plugin state which can be either global or associated with some TeamCity entity, like build or build configuration.
By settings we understand user-defined settings, specified either via user interface or DSL. Settings can be global or associated with TeamCity entities like build configuration or project.

## Plugin Settings

### Global Settings

In TeamCity it is possible for a plugin to define settings on a global level. These settings can be stored on disk in xml or 
any other plugin specific format. Traditional place for storing settings is ``<TeamCity Data Directory>/config``.
TeamCity does not provide an API to serialize or deserialize settings under config directory. It's up to plugin writer to implement it.

To provide a user interface to edit global settings, plugin can define a custom tab in Administration area by implementing [`AdminPage`](http://javadoc.jetbrains.net/teamcity/openapi/current/jetbrains/buildServer/controllers/admin/AdminPage.html) extension.
See also [Web UI Extensions](web-ui-extensions.md).

Note: in many cases instead of having settings on a global level it is better to associate settings with a project. 
Since every TeamCity installation always has __&lt;Root project&gt;__ which is a top of projects hierarchy, defining settings at the __&lt;Root project&gt;__ level 
is essentially the same as defining them globally. 

### Project-level Settings

To associate settings with a project, plugin can use [`SProjectFeatureDescriptor`](http://javadoc.jetbrains.net/teamcity/openapi/current/jetbrains/buildServer/serverSide/SProjectFeatureDescriptor.html).

Project feature is just a map of parameters of type String with some id and some type. Project features provide a convenient way of associating plugin settings with a project or project hierarchy. 
In this case There is no need to think about serialization or deserialization of settings. They will be saved and restored automatically when project itself is saved or restored. 

Project features can be added/removed at any given point of time with help of methods:
* [`addFeature`](http://javadoc.jetbrains.net/teamcity/openapi/current/jetbrains/buildServer/serverSide/SProject.html#addFeature-java.lang.String-java.util.Map-) - this method 
will create [`SProjectFeatureDescriptor`](http://javadoc.jetbrains.net/teamcity/openapi/current/jetbrains/buildServer/serverSide/SProjectFeatureDescriptor.html) instance and assign an ID to it. 
Note: changes won't be stored on disk automatically, [`persist`](http://javadoc.jetbrains.net/teamcity/openapi/current/jetbrains/buildServer/serverSide/SProject.html#persist--) method must be called to save project settings on disk.
* [`removeFeature`](http://javadoc.jetbrains.net/teamcity/openapi/current/jetbrains/buildServer/serverSide/SProject.html#removeFeature-java.lang.String-) - this method can be used to remove feature from a project,
as with ``addFeature`` method ``persist`` must be called to save settings on disk.

There are also convenience methods [`updateFeature`](http://javadoc.jetbrains.net/teamcity/openapi/current/jetbrains/buildServer/serverSide/SProject.html#updateFeature-java.lang.String-java.lang.String-java.util.Map-) to update feature settings without recreating it
and different search methods:
* [`findFeatureById`](http://javadoc.jetbrains.net/teamcity/openapi/current/jetbrains/buildServer/serverSide/SProject.html#findFeatureById-java.lang.String-)
* [`getOwnFeatures`](http://javadoc.jetbrains.net/teamcity/openapi/current/jetbrains/buildServer/serverSide/SProject.html#getOwnFeatures--) 
* [`getOwnFeaturesOfType`](http://javadoc.jetbrains.net/teamcity/openapi/current/jetbrains/buildServer/serverSide/SProject.html#getOwnFeaturesOfType--) 
* [`getAvailableFeatures`](http://javadoc.jetbrains.net/teamcity/openapi/current/jetbrains/buildServer/serverSide/SProject.html#getAvailableFeatures--) 
* [`getAvailableFeaturesOfType`](http://javadoc.jetbrains.net/teamcity/openapi/current/jetbrains/buildServer/serverSide/SProject.html#getAvailableFeaturesOfType-java.lang.String-) 

Note there is important distinction among so called `own` and `available` project features. Since project features often affect not only the project where
they are defined but all sub projects too, it is convenient to access project features of a current project and all its parents at once. 
`getAvailableFeatures*` methods provide this access. Features of the current project will be the first in collection, then there will be features of its parent, and so on until Root project is reached.

On the contrary `getOwnFeatures*` methods will return features of the current project only.

To provide a user interface for editing project feature settings plugin can define a custom tab by implementing [`EditProjectTab`](http://javadoc.jetbrains.net/teamcity/openapi/current/jetbrains/buildServer/controllers/admin/projects/EditProjectTab.html) extension. 
See also [Web UI Extensions](web-ui-extensions.md) and [TeamCity Invitations plugin](https://github.com/JetBrains/teamcity-invitations-plugin).

### Build Configuration-level Settings

If a plugin needs to associate some settings with a build configuration or template, then it should use [Build Features](build-features.md).
In this case, similar to project features serialization and deserialization of settings will be performed automatically when build configuration or 
template is saved on disk or restored.

## Plugin State

### Global State

If plugin needs to save some state globally then it should be stored under `<TeamCity Data Directory>/system/pluginData/<plugin name>` directory.
Plugin is free to use any serialization mechanism to store it's state, TeamCity itself does not provide common utilities to save or access it.

### Project or Build Configuration-level State

Both [`SProject`](http://javadoc.jetbrains.net/teamcity/openapi/current/jetbrains/buildServer/serverSide/SProject.html) and [`SBuildType`](http://javadoc.jetbrains.net/teamcity/openapi/current/jetbrains/buildServer/serverSide/SBuildType.html) 
provide access to [`CustomDataStorage`](http://javadoc.jetbrains.net/teamcity/openapi/current/jetbrains/buildServer/serverSide/CustomDataStorage.html).

### Build-level State



