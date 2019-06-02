[//]: # (title: Storing Plugin State and Settings)
[//]: # (auxiliary-id: Storing+Plugin+State+and+Settings.html)




Most of the plugins need a location to store their state and settings.   

By __state__ we mean the internal plugin data which cannot be easily computed from scratch and is usually not visible to the users of the plugin.    
The state can be either global or associated with some TeamCity entity, such as a build or a build configuration.   

By __settings__ we understand user-defined settings, specified either via the user interface or DSL.    
The plugin settings can be global or associated with TeamCity entities, such as a build configuration or a project.   

Usually both the state and settings should survive the server restart. 

In most cases plugin settings will be serialized and deserialized automatically by the TeamCity server itself; however, to serialize the state, some dedicated code should be provided by the plugin author, except for the cases when the plugin state can be calculated based on other TeamCity entities.      
For instance, a plugin whose state depends on build tests can restore its state during the server startup by processing recent builds. 

## Plugin Settings

### Global Settings

In TeamCity it is possible for a plugin to define its settings on a global level. These settings can be stored on a disk in the xml or some other plugin-specific format. The traditional place for storing all global settings is [`<TeamCity Data Directory>/config`](https://www.jetbrains.com/help/teamcity/teamcity-data-directory.html#TeamCityDataDirectory-StructureofTeamCityDataDirectory).   
TeamCity does not provide an API to serialize or deserialize settings under the `config` directory. It's up to the plugin author to write this code.

To provide a user interface to edit global settings, a plugin can define a custom tab in the Administration area by implementing the [`AdminPage`](http://javadoc.jetbrains.net/teamcity/openapi/current/jetbrains/buildServer/controllers/admin/AdminPage.html) extension.    
See also [Web UI Extensions](web-ui-extensions.md).

<note>
 
In many cases, instead of having settings on a global level, it is better to associate settings with a project.

Since every TeamCity installation always has __&lt;Root project&gt;__ which is the top of the projects' hierarchy, defining settings at the __&lt;Root project&gt;__ level is essentially the same as defining them globally. 
 </note>

### Project-level Settings

To associate settings with a project, a plugin can use [`SProjectFeatureDescriptor`](http://javadoc.jetbrains.net/teamcity/openapi/current/jetbrains/buildServer/serverSide/SProjectFeatureDescriptor.html).

A project feature is a map of parameters of type `String` with some `id` and some `type` unique among all plugins. If plugin settings can be represented as a map, then project features provide a convenient way of associating plugin settings with a project or project hierarchy. In this case, there is no need to think about serialization or deserialization of settings: they will be saved and restored automatically when the project itself is saved or restored. 


Project features can be added/removed at any given point of time with the help of methods:
* [`addFeature`](http://javadoc.jetbrains.net/teamcity/openapi/current/jetbrains/buildServer/serverSide/SProject.html#addFeature-java.lang.String-java.util.Map-): this method will create [`SProjectFeatureDescriptor`](http://javadoc.jetbrains.net/teamcity/openapi/current/jetbrains/buildServer/serverSide/SProjectFeatureDescriptor.html) instance and assign an ID to it.    
  Note: changes won't be stored on disk automatically, the [`persist`](http://javadoc.jetbrains.net/teamcity/openapi/current/jetbrains/buildServer/serverSide/SProject.html#persist--) method must be called to save project settings on disk.
* [`removeFeature`](http://javadoc.jetbrains.net/teamcity/openapi/current/jetbrains/buildServer/serverSide/SProject.html#removeFeature-java.lang.String-): this method can be used to remove feature from a project, as with the `addFeature` method, `persist` must be called to save settings on disk.

There are also convenience methods [`updateFeature`](http://javadoc.jetbrains.net/teamcity/openapi/current/jetbrains/buildServer/serverSide/SProject.html#updateFeature-java.lang.String-java.lang.String-java.util.Map-) to update feature settings without recreating it, as well as different search methods:
* [`findFeatureById`](http://javadoc.jetbrains.net/teamcity/openapi/current/jetbrains/buildServer/serverSide/SProject.html#findFeatureById-java.lang.String-)
* [`getOwnFeatures`](http://javadoc.jetbrains.net/teamcity/openapi/current/jetbrains/buildServer/serverSide/SProject.html#getOwnFeatures--) 
* [`getOwnFeaturesOfType`](http://javadoc.jetbrains.net/teamcity/openapi/current/jetbrains/buildServer/serverSide/SProject.html#getOwnFeaturesOfType--) 
* [`getAvailableFeatures`](http://javadoc.jetbrains.net/teamcity/openapi/current/jetbrains/buildServer/serverSide/SProject.html#getAvailableFeatures--) 
* [`getAvailableFeaturesOfType`](http://javadoc.jetbrains.net/teamcity/openapi/current/jetbrains/buildServer/serverSide/SProject.html#getAvailableFeaturesOfType-java.lang.String-) 

<note>
There is an important distinction between the so-called `own` and `available` project features. 
 
Since project features often affect not only the project where they are defined but all subprojects too, it is convenient to have a method returning the project features of a current project and all its parents.

This is how `getAvailableFeatures*` methods work. The features of the current project will be the first in the collection, then there will be the features of its parent, and so on until __&lt;Root project&gt;__ is reached.

On the contrary,  `getOwnFeatures*` methods will return the features of the current project only.


To provide a user interface for editing project feature settings, a plugin can define a custom tab by implementing the [`EditProjectTab`](http://javadoc.jetbrains.net/teamcity/openapi/current/jetbrains/buildServer/controllers/admin/projects/EditProjectTab.html) extension.    
See also [Web UI Extensions](web-ui-extensions.md) and [TeamCity Invitations plugin](https://github.com/JetBrains/teamcity-invitations-plugin).

### Associating Files with Project

Sometimes plugin settings are represented as files. For instance, SSH keys or Maven settings are just files and ideally they should be stored as such.    
TeamCity provides a directory for each project where files associated with the project can be stored.

Use the [`getPluginDataDirectory`](http://javadoc.jetbrains.net/teamcity/openapi/current/jetbrains/buildServer/serverSide/SProject.html#getPluginDataDirectory-java.lang.String-) method to access this directory.

The directory location on the disk is: `<`[`TeamCity Data Directory>/config`](https://www.jetbrains.com/help/teamcity/2019.1/teamcity-data-directory.html#TeamCityDataDirectory-StructureofTeamCityDataDirectory)`/<Project external ID>/pluginData/<plugin name>`. 
It's up to the plugin author to write code to store/remove/modify and delete files under this directory. TeamCity does not provide additional API in this case.

### Build Configuration-level Settings

If a plugin needs to associate some settings with a build configuration or template, then it should use [Build Features](build-features.md).

In this case, similarly to project features, serialization and deserialization of settings will be performed automatically when a build configuration or 
template is saved on the disk or restored.

## Plugin State


### Global State

If a plugin needs to save some state globally, then it should be stored under the [`<TeamCity Data Directory>/system/`](https://www.jetbrains.com/help/teamcity/2019.1/teamcity-data-directory.html#TeamCityDataDirectory-systemDir)`pluginData/<plugin name>` directory.      
A plugin is free to use any serialization mechanism to store its state; TeamCity itself does not provide common utilities to save or access it.

To access the directory containing state of all plugins, use the [`ServerPaths#getPluginDataDirectory`](http://javadoc.jetbrains.net/teamcity/openapi/current/jetbrains/buildServer/serverSide/ServerPaths.html#getPluginDataDirectory--) method where [`ServerPaths`](http://javadoc.jetbrains.net/teamcity/openapi/current/jetbrains/buildServer/serverSide/ServerPaths.html) is a Spring bean.

### Project or Build Configuration-level State

Both [`SProject`](http://javadoc.jetbrains.net/teamcity/openapi/current/jetbrains/buildServer/serverSide/SProject.html) and [`SBuildType`](http://javadoc.jetbrains.net/teamcity/openapi/current/jetbrains/buildServer/serverSide/SBuildType.html) provide access to [`CustomDataStorage`](http://javadoc.jetbrains.net/teamcity/openapi/current/jetbrains/buildServer/serverSide/CustomDataStorage.html).

Custom data storage is a key-value storage with some `id`. Any map with String keys and values can be stored there. There are no restrictions on the amount of data or length of keys and values, although it is not recommended to store megabytes of data there as it can affect the server performance and memory usage. 


An example how a custom data storage can be used with a build configuration (code is similar in case of project):


```shell
SBuldType bt = ...
CustomDataStorage pluginData = bt.getCustomDataStorage("my-plugin-data");
pluginData.putValue("some key", "some value");

// optionally persist state
pluginData.flush()

```

Custom data storage is persisted into the database automatically with some periodicity or when server is stopped. But if it is important to save the state as soon as possible, then the [`flush`](http://javadoc.jetbrains.net/teamcity/openapi/current/jetbrains/buildServer/serverSide/CustomDataStorage.html#flush--) method can be used.

To remove custom data storage, use the [`dispose`](http://javadoc.jetbrains.net/teamcity/openapi/current/jetbrains/buildServer/serverSide/CustomDataStorage.html#dispose--) method.

### Build-level State

If a plugin needs to associate some state with a build, a possible approach is to store the state under the build artifacts directory on the disk. To access this directory, use the [`SBuild#getArtifactsDirectory`](http://javadoc.jetbrains.net/teamcity/openapi/current/jetbrains/buildServer/serverSide/SBuild.html#getArtifactsDirectory--) method.

By convention, plugins should store their state under the hidden artifacts directory `.teamcity/<plugin name>`:

```shell
SBuild build = ...
File artifactsDir = build.getArtifactsDirectory();
File pluginFolder = new File(artifactsDir, jetbrains.buildServer.ArtifactsConstants.TEAMCITY_ARTIFACTS_DIR + File.separatorChar + "myPluginName");
pluginFolder.mkdirs();
... some code to serialize or deserialize state ...

```



 
