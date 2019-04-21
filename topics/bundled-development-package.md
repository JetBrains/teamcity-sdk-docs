[//]: # (title: Bundled Development Package)
[//]: # (auxiliary-id: Bundled+Development+Package.html)



TeamCity comes bundled with a Development Package that can be used to start developing TeamCity plugins.

To get the package, use the `.tar.gz` or `.exe.` distribution. Upon installation, `<TeamCity Home Directory>` will have the `devPackage` directory which contains TeamCity open API binaries, javadoc, sources and archive with a sample plugin.

## devPackage directory description

There are mainly two types of plugins in TeamCity: server\-side plugins and agent\-side plugins. To develop an agent\-side plugin, you need the following part of the Open API:
* `serviceMessages.jar`
* `common-api.jar`
* `agent-api.jar`

Correspondingly for the server\-side plugin, you need:

* `serviceMessages.jar`
* `common-api.jar`
* `server-api.jar`
Note that sometimes a part of an agent\-side plugin has to work in the same JVM where the build tool is executing. For example, some custom test runner can be executed in the JVM where the tests are running. The `runtime` directory of `devPackage` contains some jars that can be used in this case.

`devPackage` also contains some base classes for tests under the `tests` directory.

## Sample Plugin

### Building and deploying sample plugin

#### Building plugin with Apache Ant

* Unpack `<TeamCity Home Directory>\devPackage\samplePlugin-src.zip` into a directory of your choice
* Edit the `build.properties` file and set the value for `path.variable.teamcitydistribution` property to the path of `<TeamCity Home Directory>`
* Run `ant dist` in the plugin directory (Ant 1.7\+ is recommended). The plugin distribution should be created in the `dist` directory.

#### Building sample plugin in IntelliJ IDEA

* Unpack `<TeamCity Home Directory>\devPackage\samplePlugin-src.zip` into a directory of your choice
* Open the project in IDEA (the .idea project should work OK in IntelliJ IDEA 9 and later (including IntelliJ IDEA 9.0 Community Edition))
* On prompt to add the path variable, set the "TeamCityDistribution" path variable to the directory where TeamCity with devPackage is installed (&lt;[TeamCity Home Directory](https://www.jetbrains.com/help/teamcity/?teamcity-home-directory)&gt;).
* Open Project Structure and ensure you have Project SDK with name "1.6" pointing to Sun JDK version 1.6

#### Running the server with plugin from IDEA

* Either edit the `build.properties` file to set the `path.variable.teamcitydistribution` property or regenerate the build script from IDEA (execute "Generate Ant Build" with the settings: single file, all other options unchecked).
If you use the Ultimate edition of IntelliJ IDEA, you can start TeamCity's Tomcat right form the IDE:
* Go to the "server" run configuration settings and configure Application Server pointing it to `<TeamCity Home Directory>`
* Run the "server" run configuration. It will run Ant create distribution task, deploy the plugin into `${user.home}/.BuildServer` directory and run the TeamCity server.

If you use the Community edition, see [Building plugin with Apache Ant](#Building plugin with Apache Ant) \- you can run "deploy" Ant build target right from `Ant Build` IDEA tool window and then start TeamCity manually.

### Sample Plugin Functionality

The sample plugin adds "Click me!" button in the bottom of "Projects" page. Click it to navigate to the plugin description page.
