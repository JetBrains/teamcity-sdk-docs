[//]: # (title: Plugin Development FAQ)
[//]: # (auxiliary-id: Plugin+Development+FAQ.html)

## How to use logging

The TeamCity code uses the Log4j logging library with a centralized configuration on the [server](https://www.jetbrains.com/help/teamcity/?teamcity-server-logs) and [agent](https://www.jetbrains.com/help/teamcity/?viewing-build-agent-logs). Logging is usually done via a utility wrapper `com.intellij.openapi.diagnostic.Logger` rather than the default Log4j classes. You can use the [`jetbrains.buildServer.log.Loggers`](http://javadoc.jetbrains.net/teamcity/openapi/current/jetbrains/buildServer/log/Loggers.html) class to get instances of the Loggers, e.g. use `jetbrains.buildServer.log.Loggers.SERVER` to add a message to the `teamcity-server.log` file.

For plugin\-specific logging, it is recommended to log into a log category matching the full name of your class. This is usually achieved by defining the logger field in a class as `private static Logger LOG = Logger.getInstance(YourClass.class.getName());`.

If your plugin source code is located under the `jetbrains.buildServer` package, the logging will automatically go into `teamcity-server.log`.

If you use another package, you might need to add a corresponding category handling into the `conf/teamcity-server-log4j.xml` file (mentioned at [TeamCity Server Logs](https://www.jetbrains.com/help/teamcity/?teamcity-server-logs) or the corresponding agent file).

For debugging you might consider creating a customized Log4j configuration file and put it as a logging preset into [`<TeamCity Data Directory>\config\logging`](https://www.jetbrains.com/help/teamcity/?teamcity-server-logs) directory. This way one will be able to activate the preset via the __Administration__ | __Diagnostics__ page, __Troubleshooting__ tab.

## How to adapt plugin for secondary node
   
TeamCity administrators can set up [secondary nodes](https://www.jetbrains.com/help/teamcity/configuring-secondary-node.html) to work alongside the main TeamCity server. Such nodes provide a read-only user interface to ensure high availability when the main node is unavailable. Optionally, they can be assigned to certain responsibilities and perform writing operations.

Not all custom plugins can run on a secondary node. Here is the expected configuration:

* The `teamcity-plugin.xml` file must have the following attribute: `<deployment node-responsibilities-aware="true"/>`.
* A plugin should use `jetbrains.buildServer.serverSide.IOGuard` to wrap network and I/O operations which do not change the state of the external systems.   
A secondary node runs under the Security Manager and often does not have permissions to change anything outside it. For instance, if it runs as read-only, all I/O operations – even reading – will be prohibited by default. To allow reading operations, wrap them with IOGuard in the plugin code.   

If a plugin only shows information in the UI, these measures should be sufficient and will allow loading it on a secondary node. With these settings, the plugin will also work fine on the main node.

If a plugin needs to perform any actions, it should check the context of its assigned responsibilities. This API is still in progress – see [TW-58322](https://youtrack.jetbrains.com/issue/TW-58322) for details.


[//]: # (See "Plugin Development FAQd251e83.txt" for more information.)    



