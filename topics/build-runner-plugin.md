[//]: # (title: Build Runner Plugin)
[//]: # (auxiliary-id: Build+Runner+Plugin.html)

A [build runner](https://www.jetbrains.com/help/teamcity/?build-runner) plugin consists of two parts: agent\-side and server\-side. The server side part of the plugin provides meta information about the build runner, the web UI for the build runner settings and the build runner properties validator. The agent\-side part launches builds.

On this page:

A build runner can have various settings which must be edited by the user in the web UI and passed to the agent. These settings are called __runner parameters__ (or __runner properties__) and provided as a Map&lt;String, String&gt; to the agent part of the runner.

<tip>

Hint: some build runners whose source code can be used as a reference:
* [Rake Runner](https://github.com/JetBrains/teamcity-rake)
* [FxCop runner sources](https://github.com/JetBrains/teamcity-fxcop)  Other build runner [plugins](https://plugins.jetbrains.com/teamcity).
</tip>

## Server-side part of the runner

The main entry point for the runner on the server side is `jetbrains.buildServer.serverSide.RunType`. A build runner plugin must provide its' own RunType and register it in the `jetbrains.buildServer.serverSide.RunTypeRegistry`.

RunType has a __type__ which must be unique among all build runners and correspond to the __type__ returned by the agent\-side part of the runner (see `jetbrains.buildServer.agent.AgentBuildRunnerInfo`).

The __getEditRunnerParamsJspFilePath__ and __getViewRunnerParamsJspFilePath__ methods return paths to JSP files for editing and viewing runner settings. These JSP files must be bundled with plugin in __buildServerResources__ subfolder, [read more](plugins-packaging.md). The paths should be relative to the __buildServerResources__ folder.

<note>

Since TeamCity 5.1, the path to the build runner resources files should be a full path without context. This path could be either a path to a .jsp file or a path that is handled by a controller. The plugin class may use __PluginDescriptor#getPluginResourcesPath()__ method to create a path to a .jsp file from the buildServerResources folder of the plugin.
</note>

<note>

TeamCity 5.0.x and earlier uses the following rule to compute a full path to the runner's jsp:


```
 <context path>/plugins/<runType>/<returned jsp path>

```


</note>

<tip>

Hint: before writing your own JSP for a custom build runner, take a look at the JSP files of the existing runners bundled with TeamCity.
</tip>

When a user fills in your runner settings and submits the form, `jetbrains.buildServer.serverSide.PropertiesProcessor` returned by the __getRunnerPropertiesProcessor__ method will be called. This processor will be able to the verify user settings and indicate which of them are invalid.

Usually a JSP page is simple and does not provide much controls except for fields, checkboxes and so on. But if you need more control on how the page is processed on the server side, then you should register your own extension to the runner editing controller: `jetbrains.buildServer.controllers.admin.projects.EditRunTypeControllerExtension`.

And finally if you need to prefill some settings with default values, you can do this with the help of the __getDefaultRunnerProperties__ method.

## Agent-side part of the runner

The main interface for agent\-side runners is `jetbrains.buildServer.agent.AgentBuildRunner`. However, if your custom runner runs an external process, it is simpler to use the following classes:
1. `jetbrains.buildServer.agent.runner.CommandLineBuildServiceFactory`
2. `jetbrains.buildServer.agent.runner.CommandLineBuildService`
3. `jetbrains.buildServer.agent.runner.BuildServiceAdapter`
You should implement the __CommandLineBuildServiceFactory__ factory interface and make your class a Spring bean. The factory also provides some meta information about the runner via `jetbrains.buildServer.agent.AgentBuildRunnerInfo`.

__CommandLineBuildService__ is an abstract class which simplifies external processes launching and allows listening for process events (output, finish and so on). Your runner should extend this class. Snce TeamCity 6.0, we introduced the `jetbrains.buildServer.agent.runner.BuildServiceAdapter` class that extends __CommandLineBuildService__ and provides utility methods to access build and runner context parameters.

__AgentBuildRunnerInfo__ has two methods: __getType__ which must return the same __type__ as the one returned by the server\-side part of the plugin, and __canRun__ which is called to determine whether the custom runner can run on the agent (in the agent environment).

If the command line build service is not suitable for your needs, you can still implement the __AgentBuildRunner__ interface and define it in the Spring context. Then it will be loaded automatically.

## Extending the Ant runner

The TeamCity Ant runner, while being a plugin itself, can also be extended with the help of __jetbrains.buildServer.agent.ant.AntTaskExtension__. This extension works in the same JVM where Ant is running. Using this extension, you can watch for Ant tasks, modify/patch them and log various messages to the build log.

Your class implementing __AntTaskExtension__ interface must be defined in the Spring bean and it will be picked up by the Ant runner automatically. You need to add a dependency to &lt;teamcity&gt;/webapps/ROOT/WEB\-INF/plugins/ant/agent/antPlugin.zip!antPlugin/ant\-runtime.jar jar.

## Your Build Runner Results in TeamCity

### Build log

Usually a build runner starts an external process, and logging is performed from that process. The simplest way to log messages in this case is to use __service messages__, [read more](https://www.jetbrains.com/help/teamcity/?build-script-interaction-with-teamcity). In brief, a service message is a specially formatted text with attributes; when such text is logged to the process output, it is parsed and the associated processing is performed. With the help of these messages you can create a TeamCity hierarchical build log, report tests, errors and so on.

If an external process launched by your runner is Java and you can't use service messages, it is possible to obtain `jetbrains.buildServer.agent.BuildProgressLogger` in the class running in this JVM. For this, the following jar files must be added in the classpath of the external Java process: runtime\-util.jar, server\-logging.jar. Then you should use the `jetbrains.buildServer.agent.LoggerFactory` method to construct the logger: LoggerFactory.createBuildProgressLogger(parentClassloader). Since this way is more involved, it is recommended to use service messages instead.

If logging of the messages is done in the agent JVM (not from within the external process started by your runner), you can obtain `jetbrains.buildServer.agent.BuildProgressLogger` from the `jetbrains.buildServer.agent.AgentRunningBuild#getBuildLogger` method.

### Artifacts

You can instruct your build runner to publish the resulting artifacts to TeamCity using [service messages](https://www.jetbrains.com/help/teamcity/?build-script-interaction-with-teamcity). Note that artifacts are uploaded to the TeamCity server in the background, so to verify that your artifacts are uploaded, you'll have to wait until your build is finished.  

### Reports

#### XML Report processing

If your runner reports build results in a format supported by TeamCity, they can be displayed in the TeamCity web UI on the Build Results page. There are two ways to approach this:\- using the [XML Report Processing](https://www.jetbrains.com/help/teamcity/?xml-report-processing) build feature\- via [service messages ](https://www.jetbrains.com/help/teamcity/?build-script-interaction-with-teamcity)

#### HTML Report processing

If your build runner produces some static HTML content, it can be displayed in the TeamCity web UI.  Configure a custom [report tab](https://www.jetbrains.com/help/teamcity/?including-third-party-reports-in-the-build-results) to show the results on a project or build level.

 
