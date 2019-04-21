[//]: # (title: Plugin Development FAQ)
[//]: # (auxiliary-id: Plugin+Development+FAQ.html)



## How to Use Logging

The TeamCity code uses the Log4j logging library with a centralized configuration on the [server](https://www.jetbrains.com/help/teamcity/?teamcity-server-logs) and [agent](https://www.jetbrains.com/help/teamcity/?viewing-build-agent-logs). Logging is usually done via a utility wrapper `com.intellij.openapi.diagnostic.Logger` rather than the default Log4j classes. You can use the [`jetbrains.buildServer.log.Loggers`](http://javadoc.jetbrains.net/teamcity/openapi/current/jetbrains/buildServer/log/Loggers.html) class to get instances of the Loggers, e.g. use `jetbrains.buildServer.log.Loggers.SERVER` to add a message to the `teamcity-server.log` file.

For plugin\-specific logging, it is recommended to log into a log category matching the full name of your class. This is usually achieved by defining the logger field in a class as `private static Logger LOG = Logger.getInstance(YourClass.class.getName());` 

If your plugin source code is located under the `jetbrains.buildServer` package, the logging will automatically go into `teamcity-server.log.` 

If you use another package, you might need to add a corresponding category handling into the `conf/teamcity-server-log4j.xml` file (mentioned at [TeamCity Server Logs](https://www.jetbrains.com/help/teamcity/?teamcity-server-logs) or the corresponding agent file.

For debugging you might consider creating a customized Log4j configuration file and put it as a logging preset into [`<TeamCity Data Directory>\config\_logging`](https://www.jetbrains.com/help/teamcity/?teamcity-server-logs) directory. This way one will be able to activate the preset via the __Administration__ | __Diagnostics__ page, __Troubleshooting__ tab.


[//]: # (See "Plugin Development FAQd251e83.txt" for more information.)    



