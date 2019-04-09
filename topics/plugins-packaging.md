[//]: # (title: Plugins Packaging)
[//]: # (auxiliary-id: Plugins+Packaging.html)



This page is intended for plugin developers and explains how to package TeamCity plugins and agent tools. See [Installing Additional Plugins](https://www.jetbrains.com/help/teamcity/?installing-additional-plugins) and [Installing Agent Tools](https://www.jetbrains.com/help/teamcity/?installing-agent-tools) for installation instructions.

On this page:

## Introduction

To write a TeamCity plugin, the knowledge of [Spring Framework](http://static.springsource.org/spring/docs/3.0.x/reference/beans.html) is beneficial.

There are [server-side and agent-side](plugin-types-in-teamcity.md) plugins in TeamCity. Server\-side and agent\-side plugins are initialized in their own Spring containers; this means that every plugin needs a Spring bean definition file describing the main services of the plugin. Bean definition files are to be placed into the `META\-INF` folder of the JAR archive containing the plugin classes.

There is a convention for naming the definition file:
* __build\-server\-plugin\-&lt;plugin name&gt;__\*.xml — for server\-side plugins
* __build\-agent\-plugin\-&lt;plugin name&gt;__\*.xml — for agent\-side plugins
where the asterisk can be replaced with any text, for example: __build\-server\-plugin\-cvs.xml__.

<tip>

If you want to get started with an empty plugin quickly, try the template plugin in the JetBrains Subversion repository [http://svn.jetbrains.org/teamcity/plugins/template-plugin/templateProject](http://svn.jetbrains.org/teamcity/plugins/template-plugin/templateProject). Refer to `readme.txt` for instructions.
</tip>

## Plugins Location

TeamCity is able to load plugin from the following directories:
* `<TeamCity data directory>/plugins` – [user-installed](https://www.jetbrains.com/help/teamcity/?installing-additional-plugins) plugins
* `<TeamCity web application>/WEB\-INF/plugins` — default directory for bundled TeamCity plugins
Plugins with the same name (for example, a newer version) located in `<TeamCity data directory>/plugins` will override the plugins in the `<TeamCity web application>/WEB\-INF/plugins` directory.

## Plugins Loading

TeamCity creates a child Spring Framework context per plugin. There are two options to load plugins classes: __standalone__ and __shared__:
* Standalone classloading (__recommeneded__) allows loading every plugin to a separate classloader. This approach allows a plugin to have additional libraries without the risk of affecting the server or other plugins.
* Shared classloading allows loading all plugins into same classloader. It is not allowed to override any libraries here.
You may specify desired the classloading mode in the `teamcity\-plugin.xml` file, see the [section below]().

 The TeamCity plugin loader supports plugin dependencies, described [below]().

<note>

Earlier, to load your plugin, the server had to be restarted. __Since TeamCity 2018.2__, no server restart is need.
</note>

## Server-Side Plugins

A server\-side plugin may affect the server only, or may include a number of agent\-side plugins that will be automatically distributed to all build agents.

### Plugin Structure

A plugin can be a zip archive (__recommended__) or a separate folder.

If you use a _zip file_:
* TeamCity will use the name of the zip file as the plugin name
* The plugin zip file will be automatically unpacked to a temporary directory on the server start\-up
If you use a _separate folder_:
* TeamCity will use the folder name as the plugin name
The plugin zip archive/directory includes:
* `teamcity\-plugin.xml` containing meta information about the plugin, like its name and version, see the [section below]().
* the `server` directory containing the server\-side part of the plugin, i.e, a number of jar files.
* the `agent` directory containing `<agent plugin zip>` if your plugin affects agents too, see the [section below]().
The plugin directory should have the following structure:

The server\-only plugin:
server
  |
  \-\-&gt; &lt;server plugin jar files&gt;
teamcity\-plugin.xml

The plugin affecting the server and agents:
agent
  |
  \-\-&gt; &lt;agent plugin zip files&gt; (see \[below|#agentDirectory\])
server
  |
  \-\-&gt; &lt;server plugin jar files&gt;
teamcity\-plugin.xml

#### Web resources packaging

In most cases a plugin is just a number of classes packed into a JAR file.

If you wish to write a custom page for TeamCity, most likely you'll need to place images, CSS, JavaScript, JSP files or binaries somewhere. The files that you want to access via hyperlinks and JSP pages are to be placed into the `buildServerResources` subfolder of the plugin's .jar file. Upon the server startup, these files will be extracted from the archive. You may use `jetbrains.buildServer.web.openapi.PluginDescriptor` spring bean to get the paths to the extracted resources ([read more](web-ui-extensions.md) on how to construct paths to your JSP files).

It is a good practice to put all resources into a separate.jar file.

### Plugin Descriptor

The `teamcity\-plugin.xml` file must be located in the root of the plugin directory or .zip file. You can refer to the XSD schema for this file which is unpacked to `<TeamCity data directory>/config/teamcity\-plugin\-descriptor.xsd`

An example of __teamcity\-plugin.xml__:


```
<?xml version="1.0" encoding="UTF\-8"?>
<teamcity\-plugin xmlns:xsi="http://www.w3.org/2001/XMLSchema\-instance"
                 xsi:noNamespaceSchemaLocation="urn:shemas\-jetbrains\-com:teamcity\-plugin\-v1\-xml">
  <info>

    <name>PluginName</name> <!\-\- the name of plugin used in teamcity \-\->
    <display\-name>This name may be used for UI</display\-name>
    <description>Some description goes here</description>
    <version>0.239.42</version>
  </info>
  <requirements min\-build="46654" max\-build="57000" /> <!\-\- specify compatible TeamCity server builds \-\->
  <deployment use\-separate\-classloader="true" allow\-runtime\-reload="true" /> <!\-\- load server plugin's classes in a separate classloader and reload plugin without the server restart.\-\->
  <parameters>
    <parameter name="key">value</parameter>
    <!\-\- ... \-\->
  </parameters>
</teamcity\-plugin>

```



 It is recommended to set the `use\-separate\-classloader="true"` parameter to `true` for server\-side plugins. To reload plugin without the server restart, use the `allow\-runtime\-reload="true"` parameter for deployment.

The plugin parameters can be accessed via the `jetbrains.buildServer.web.openapi.PluginDescriptor#getParameterValue(String)` method.

## Agent-Side Plugins

TeamCity build agents support the following plugin structures:
* new plugins (with the `teamcity\-plugin.xml` descriptor), including _tool_ plugins * tool plugins (with the `teamcity\-plugin.xml` descriptor). This is a kind of plugin without any classes loaded into the runtime. Tool plugins for agents are used to only distribute binary files to agents, e.g. the NuGet plugin for TeamCity creates a tool plugin for agents to redistribute the downloaded NuGet.exe to TeamCity agents. See more at [Installing Agent Tools](https://www.jetbrains.com/help/teamcity/?installing-agent-tools).
* deprecated plugins (with the plugin name folder in the .zip file)
### Plugin Structure

The `agent` directory must have one file only: _&lt;agent plugin zip&gt;_ structured the following way:

#### Deprecated Plugin Structure

The old plugin structure implied that all plugin files and directories were placed into the single root directory, i.e. there had to be one root directory in the archive, the\_&lt;plugin name directory&gt;\_, and no other files at the top level. All .jar files required by the plugin on agents were placed into the `lib` subfolder:
&lt;plugin name directory&gt;
  |
  \-\-&gt; lib
       |
       \-\-&gt; &lt;jar files&gt;

There must be no other items in the root of .zip but the directory with the plugin name. TeamCity build agent detects and loads such plugins using the shared classloader.

#### New Plugins

Now a new, more flexible schema of packing is recommended. The plugin name root directory inside the plugin archive is no longer required. The agent plugin name now is obtained from the `PluginName.zip` file name. The archive needs to include the plugin descriptor, `teamcity\-plugin.xml`, [see below]().
agent\-plugin\-name.zip
  |
    \- teamcity\-plugin.xml
    \- lib
      |
       plugin.jar
       plugin.lib 

### Plugin Descriptor

It is required to have the `teamcity\-plugin.xml` file under the root of the agent plugin `.zip` file. The agent tries to validate the plugin\-provided `teamcity\-plugin.xml` file against the xml schema. If `teamcity\-plugin.xml` is not valid, the plugin will be loaded, but some data from the descriptor may be lost.

#### Plugins

This `teamcity\-plugin.xml` file provides the plugin description (same as it is done on the server\-side):


```
<?xml version="1.0" encoding="UTF\-8"?>
<teamcity\-agent\-plugin 
  xmlns:xsi="http://www.w3.org/2001/XMLSchema\-instance"
  xsi:noNamespaceSchemaLocation="urn:shemas\-jetbrains\-com:teamcity\-agent\-plugin\-v1\-xml">
  <plugin\-deployment use\-separate\-classloader="true"/>
</teamcity\-agent\-plugin>

```



#### Tools

To deploy a tool, use the following `teamcity\-plugin.xml` file:


```
<?xml version="1.0" encoding="UTF\-8"?>
<teamcity\-agent\-plugin
 xmlns:xsi="http://www.w3.org/2001/XMLSchema\-instance"
 xsi:noNamespaceSchemaLocation="urn:shemas\-jetbrains\-com:teamcity\-agent\-plugin\-v1\-xml">
  <tool\-deployment/>
</teamcity\-agent\-plugin>

```



#### Making File Executable

 There is experimental ability (can be removed in the future versions!) to set executable bit to some files after unpacking on the agent. Watch [TW-21673](https://youtrack.jetbrains.com/issue/TW-21673) for proper solution. To make some files of a tool executable, use the following `teamcity\-plugin.xml` file:


```
<?xml version="1.0" encoding="UTF\-8"?>
        <teamcity\-agent\-plugin xmlns:xsi="http://www.w3.org/2001/XMLSchema\-instance"
                         xsi:noNamespaceSchemaLocation="urn:shemas\-jetbrains\-com:teamcity\-agent\-plugin\-v1\-xml">
          <tool\-deployment>
            <layout>
              <executable\-files>
                <include name='path\_to\_a\_file'/>
              </executable\-files>
            </layout>
          </tool\-deployment>
        </teamcity\-agent\-plugin>

```



where `<include name='path\_to\_a\_file' />` relative to your tool folder ( e.g. `<Agent home>/tools/<your tool name>`) specifies the list of files to be made executable on Linux/Unix/Mac.   Note that wildcards are not supported.

See [Installing Agent Tools](https://www.jetbrains.com/help/teamcity/?installing-agent-tools) for installation instructions.

## Plugin Dependencies

Plugin dependencies are present on both the server and agent side: some components are separated from the core into separate bundled plugins: Ant runner, IDEA runner, .NET runners, JUnit, and TestNG support.If you need some functionality of one of these plugins, use the plugin dependencies feature.

To use plugin dependencies, add the\`dependencies\` tag into the plugin xml descriptor:

Example of the server\-side plugin descriptor using plugin dependencies:


```
<?xml version="1.0" encoding="UTF\-8"?>
<teamcity\-plugin xmlns:xsi="http://www.w3.org/2001/XMLSchema\-instance"
xsi:noNamespaceSchemaLocation="urn:schemas\-jetbrains\-com:teamcity\-plugin\-v1\-xml">
<info>

<name>Plugin Name</name>
<!\-\- Some tags skipped \-\->
</info>
<deployment use\-separate\-classloader="true"/>
<dependencies>
<plugin name="dotNetRunners"/>
</dependencies>
</teamcity\-plugin>

```



Example of agent\-side plugin descriptor:


```
<?xml version="1.0" encoding="UTF\-8"?>
<teamcity\-agent\-plugin xmlns:xsi="http://www.w3.org/2001/XMLSchema\-instance"
xsi:noNamespaceSchemaLocation="urn:schemas\-jetbrains\-com:teamcity\-agent\-plugin\-v1\-xml">
<plugin\-deployment use\-separate\-classloader="true"/>
<dependencies>
<tool name="ant"/>
<plugin name="ant\-runner"/>
</dependencies>
</teamcity\-agent\-plugin>

```



<note>

Using separate classloader is required (and will be enforced) to use dependencies.Transitive dependencies are not supported, you should specify all dependencies.
</note>

The names of the bundled tools and plugins are just the names of the corresponding folders in `TeamCity Home/webapps/ROOT/WEB\-INF/plugins` for the server\-side plugins and `<Agent home>/plugins/` or `<Agent home>/tools/` for the agent\-side plugins and tools.

<note>

Some bundled plugin names may change in future releases.
</note>

## Agent Upgrade on Updating Plugins

TeamCity server monitors all agent plugins .zip files for a change (plugin files changed, added or removed). Once a change is detected, agents receive the upgrade command from the server and download the updated files automatically. It means that if you want to deploy an updated agent part of your plugin without the server restart, you can put your agent plugin into this folder.

After a successful upgrade, your plugin will be unpacked into the `<Agent home>/plugins/` or `<Agent home>/tools/` folders. Note that if an agent is busy running a build, it will upgrade only after the build finishes. No new builds will start on the agent if it is to be upgraded.
