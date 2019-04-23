[//]: # (title: Developing TeamCity Plugins)
[//]: # (auxiliary-id: Developing+TeamCity+Plugins.html)


<tip>

TeamCity is hiring! Learn about the [available vacancies](https://www.jetbrains.com/careers/jobs/?team=TeamCity) on the JetBrains site.
</tip>

TeamCity functionality can be significantly extended by custom plugins. TeamCity plugins are written in Java (any JVM language with Java invulnerability like Kotlin or Groovy can be used), run within the TeamCity application and have access to internal entities of the TeamCity server or agent.

Aside from this documentation, refer to the following sources:
* [Open API Javadoc](http://javadoc.jetbrains.net/teamcity/openapi/current/)
* bundled [sample plugin](bundled-development-package.md#BundledDevelopmentPackage-SamplePlugin)
* [list](https://plugins.jetbrains.com/teamcity) of existing plugins and [bundled open-source plugins](https://confluence.jetbrains.com/display/TW/Open-source+Bundled+Plugins)

If you need more information or have a question regarding the API, please do not hesitate to post your question into [TeamCity Plugins forum](https://teamcity-support.jetbrains.com/hc/en-us/community/topics/200366719-TeamCity-Plugin-Development). Use the search before posting to avoid possible duplication of discussions.

Consider making your plugin public and submit it to the [TeamCity plugins repository](https://plugins.jetbrains.com/teamcity).

Please refer to corresponding section for further details.

[//]: # (See "Developing TeamCity Pluginsd118e57.txt" for more information.)    

## Plugin Quick Start

To quickly create your first plugin with Maven, see [Getting Started with Plugin Development](getting-started-with-plugin-development.md). 
For more details, refer to [Developing Plugins Using Maven](developing-plugins-using-maven.md) featuring a  Maven archetype supported by JetBrains

 
You may also find [Gradle TeamCity plugin](https://github.com/rodm/gradle-teamcity-plugin) useful, which supports agent and server\-side plugins, and helps to download, install a TeamCity server, perform tasks to deploy, start and stop the server and agent.

