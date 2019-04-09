[//]: # (title: Plugin Settings)
[//]: # (auxiliary-id: Plugin+Settings.html)



## Server-wide settings

A plugin can store server\-wide setting in the `main\-config.xml` file (stored in `TEAMCITY\_DATA\_PATH/config` directory). To use this file, the plugin should register an [extension](extensions.md) which implements `jetbrains.buildServer.serverSide.MainConfigProcessor`. This interface has methods which allow loading and saving some data in the XML format (via JDOM). Please note, that the plugin will be asked to reinitialize data if the file has been changed on the disk while TeamCity is up and running.

## Project-wide settings

Per\-project settings can be stored in the `TEAMCITY\_DATA\_PATH/config/<project\-name>/plugin\-settings.xml` directory.

To manage the settings in this file, you should implement a `jetbrains.buildServer.serverSide.settings.ProjectSettingsFactory` interface and register this implementation in `jetbrains.buildServer.serverSide.settings.ProjectSettingsManager` (which can be obtained via the constructor injection). Upon registration, you should specify the name of the XML node under where the settings will be stored.

Your settings should be serialized to the XML format by your implementation of the `jetbrains.buildServer.serverSide.settings.ProjectSettings` interface. The \{readFrom\}\} and `writeTo` methods should be implemented consistently.

When your code needs the stored XML settings, they should be loaded via `ProjectSettingsManager#getSettings` call. Your registered factory will create these settings in memory.

You can save this project's settings explicitly via the `jetbrains.buildServer.serverSide.SProject`#persist() call, or via `ProjectManager#persistAllProjects`. This can be done, for instance, upon some event (see `jetbrains.buildServer.serverSide.BuildServerAdapter`#serverStartup()).
