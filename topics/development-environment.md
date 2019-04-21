[//]: # (title: Development Environment)
[//]: # (auxiliary-id: Development+Environment.html)



### Plugin Debugging

You can debug your plugin in a running TeamCity just like a regular Java application debug: start TeamCity server with debug\-enabling JVM options and then connect to a remote debug port from the IDE. If you start TeamCity server from outside of your IDE, in IntelliJ IDE you can use "Remote" run configuration, check related [external blog post](http://jaketrent.com/post/debugging-remote-tomcat-intellij/). The JVM options for the server can be set via [TEAMCITY_SERVER_OPTS](https://www.jetbrains.com/help/teamcity/?configuring-teamcity-server-startup-properties) environment variable.

### Plugin Reloading

Earlier, if you make changes to a plugin, you will generally need to shut down the server, update the plugin, and restart the server. __Since TeamCity 2018.2__, you can update your plugin without the server restart.

To enable TeamCity development mode, pass the "`teamcity.development.mode=true`" [internal property](https://www.jetbrains.com/help/teamcity/?configuring-teamcity-server-startup-properties). Using the option you will:
* Enforce application server to quicker recompile changed `.jsp` classes
* Disable JS and CSS resources merging/caching

In the [plugin descriptor](plugins-packaging.md), set the `use-separate-classloader="true"` and `allow-runtime-reload="true"` parameters for deployment.

__Prior to TeamCity 2018.2__, use the following hints to eliminate the server restart in the certain cases:

* if you do not change code affecting plugin initialization and change only body of the methods, you can attach to the server process with a debugger and use Java hotswap to reload the changed classes from your IDE without web server restart. Note that the standard hotswap does not allow you to change method signatures.
* if you make a change in some resource (jsp, js, images) you can copy the resources to `webapps/ROOT/plugins/<plugin-name>` directory to allow Tomcat to reload them.
[//]: # (See "Development Environmentd119e72.txt" for more information.)    
* change in build agent part of plugin will initiate build agents upgrade.

If you replace a deployed plugin .zip file with changed class files while TeamCity server is running, this can lead to NoClassDefFound errors. To avoid this, set "`teamcity.development.shadowCopyClasses=true`" [internal property](https://www.jetbrains.com/help/teamcity/?configuring-teamcity-server-startup-properties). 

This will result in:
* creating "`.teamcity_shadow`" directory for each plugin .jar file;
* avoiding .jar files update on changing the plugin archive.



[//]: # (See "Development Environmentd119e101.txt" for more information.)    


  __See also:__

__Extending TeamCity__: [Developing TeamCity Plugins](https://confluence.jetbrains.com/display/TCD18/Developing+TeamCity+Plugins) | [Plugins Packaging](plugins-packaging.md)
