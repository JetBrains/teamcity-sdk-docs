[//]: # (title: Developing TeamCity Plugins)
[//]: # (auxiliary-id: Developing+TeamCity+Plugins.html)


<tip>

TeamCity is hiring! Learn about [the available vacancies](https://www.jetbrains.com/careers/jobs/?team=TeamCity) on the JetBrains site. [Read](https://blog.jetbrains.com/blog/2015/04/22/jetbrains-is-hiring-for-teamcity-join-our-team/) about working in the TeamCity team.
</tip>

TeamCity functionality can be significantly extended by custom plugins. TeamCity plugins are written in Java (any JVM language with Java invulnerability like Kotlin or Groovy can be used), run within the TeamCity application and have access to internal entities of the TeamCity server or agent.

Aside from this documentation, please refer to the following sources:
* [Open API Javadoc](http://javadoc.jetbrains.net/teamcity/openapi/current/)
* bundled [sample plugin](bundled-development-package.md)
* [list](https://plugins.jetbrains.com/teamcity) of existing plugins and [bundled open-source plugins](https://confluence.jetbrains.com/display/TW/Open-source+Bundled+Plugins)

If you need more information or have a question regarding the API, please do not hesitate to post your question into [TeamCity Plugins forum](https://teamcity-support.jetbrains.com/hc/en-us/community/topics/200366719-TeamCity-Plugin-Development). Please use the search before posting to avoid possible duplication of discussions.

Consider making your plugin public and submit it to the [TeamCity plugins repository](https://plugins.jetbrains.com/teamcity).

Please refer to corresponding section for further details.

[//]: # (See "Developing TeamCity Pluginsd118e57.txt" for more information.)    

## Plugin Quick Start

See [Getting Started with Plugin Development](getting-started-with-plugin-development.md) to create your first plugin with Maven. 
[Developing Plugins Using Maven](developing-plugins-using-maven.md) provides more details.



There are also several approaches to create plugins provided by third parties or existing out of the main TeamCity development line.
* [Developing Plugins Using Maven](developing-plugins-using-maven.md) \- Maven archetype supported by JetBrains
* [ TeamCity plugin](https://github.com/rodm/gradle-teamcity-plugin) \- Git, Gradle build. Supports agent and server\-side plugins, and helpers to download, install a TeamCity server, tasks to deploy, start and stop the server and agent.
* [template plugin 1](https://github.com/jonnyzzz/TeamCity.PluginTemplate), see also a [blog post](http://jonnyzzz.com/blog/2012/09/10/teamcity-plugin-template/) \- Git, IDEA project
* [template plugin 2](http://svn.jetbrains.org/teamcity/plugins/template-plugin/templateProject/readme.txt) \- Subversion, IDEA project and __Ant__ build, generates a plugin with custom name, see details in the readme.txt of the checkout
* [Gradle TeamCity plugin](http://github.com/nskvortsov/gradle-teamcity-plugin) \- earlier, but a bit outdated version of Gradle build for TeamCity plugins
* (obsolete) [Maven Archetype for TeamCity server plugin](http://devnet.jetbrains.net/message/5447733#5447733)

See also a [post](http://devnet.jetbrains.net/thread/439412?tstart=0) on the very first steps for setting up the plugin development environment.
