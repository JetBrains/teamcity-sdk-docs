[//]: # (title: Developing TeamCity Plugins)
[//]: # (auxiliary-id: Developing+TeamCity+Plugins.html)

>TeamCity is hiring! Learn about the [available vacancies](https://www.jetbrains.com/careers/jobs/?team=TeamCity) on the JetBrains site.

TeamCity functionality can be significantly extended by custom plugins. TeamCity plugins are written in Java (any JVM-based language like Kotlin or Groovy can be used), run within the TeamCity application and have access to internal entities of the TeamCity server or agent.

Aside from this documentation, refer to the following sources:
* [Open API Javadoc](http://javadoc.jetbrains.net/teamcity/openapi/current/)
* bundled [sample plugin](bundled-development-package.md#Sample+Plugin)
* [list](https://plugins.jetbrains.com/teamcity) of existing plugins and [bundled open-source plugins](#Bundled+Open-Source+Plugins)

If you need more information or have a question regarding the API, please do not hesitate to post your question into [TeamCity Plugins forum](https://teamcity-support.jetbrains.com/hc/en-us/community/topics/200366719-TeamCity-Plugin-Development). Use the search before posting to avoid possible duplication of discussions.

Consider making your plugin public and submit it to [JetBrains Marketplace](https://plugins.jetbrains.com).

Please refer to corresponding section for further details.

[//]: # (See "Developing TeamCity Pluginsd118e57.txt" for more information.)    

## Plugin Quick Start

To quickly create your first plugin with Maven, see [Getting Started with Plugin Development](getting-started-with-plugin-development.md). For more details, refer to the section [Developing Plugins Using Maven](developing-plugins-using-maven.md) featuring a  Maven archetype supported by JetBrains.

 
You may also find [Gradle TeamCity plugin](https://github.com/rodm/gradle-teamcity-plugin) useful, which supports agent and server\-side plugins, and helps to download, install a TeamCity server, perform tasks to deploy, start and stop the server and agent.


## Bundled Open-Source Plugins

The following plugins bundled with TeamCity are developed by JetBrains and are open-sourced under Apache 2.0 license.

<dl>

<dt><b>REST API</b></dt>
<dd>
<p>Exposes the TeamCity API via REST.</p>
<p><a href="https://github.com/JetBrains/teamcity-rest">Sources on GitHub</a> | <a href="https://www.jetbrains.com/help/teamcity/rest/teamcity-rest-api-documentation.html">REST API Documentation</a></p>
</dd>

<dt><b>XML Test Reporting</b></dt>
<dd>
<p>
Allows TeamCity to parse XML-based report files produced by external tools and display these reports as build results.
</p>
<p><a href="https://github.com/JetBrains/teamcity-xml-tests-reporting">Sources on GitHub</a> | <a href="https://www.jetbrains.com/help/teamcity/xml-report-processing.html">TeamCity Documentation</a></p>
</dd>

<dt><b>Git VCS Support</b></dt>
<dd>
<p>Adds <a href="https://git-scm.com">Git</a> support.</p>
<p><a href="https://github.com/JetBrains/teamcity-git">Sources on GitHub</a> | <a href="https://www.jetbrains.com/help/teamcity/git.html">TeamCity Documentation</a></p>
</dd>


<dt><b>Build Queue Priorities</b></dt>
<dd>
<p>Allows you to set priorities for build configurations.</p>
<p><a href="https://github.com/JetBrains/teamcity-priority-queue">Sources on GitHub</a> | <a href="https://www.jetbrains.com/help/teamcity/working-with-build-queue.html#Managing+Build+Priorities">TeamCity Documentation</a></p>
</dd>


<dt><b>Swabra</b></dt>
<dd>
<p>Allows you to clean files created during a build.</p>
<p><a href="https://github.com/JetBrains/teamcity-swabra">Sources on GitHub</a> | <a href="https://www.jetbrains.com/help/teamcity/build-files-cleaner-swabra.html">TeamCity Documentation</a></p>
</dd>



<dt><b>PowerShell Runner</b></dt>
<dd>
<p>Runs PowerShell scripts.</p>
<p><a href="https://github.com/JetBrains/teamcity-powershell">Sources on GitHub</a> | <a href="https://www.jetbrains.com/help/teamcity/powershell.html">TeamCity Documentation</a></p>
</dd>



<dt><b>Shared Resources</b></dt>
<dd>
<p>Allows you to limit the number of running builds that use the same shared resource (for example, an external database or a server with a limited number of connections).</p>
<p><a href="https://www.jetbrains.com/help/teamcity/shared-resources.html">Sources on GitHub</a> | <a href="https://www.jetbrains.com/help/teamcity/shared-resources.html">TeamCity Documentation</a></p>
</dd>


<dt><b>Queue Manager</b></dt>
<dd>
<p>Allows users with corresponding permissions to pause and resume build queues.</p>
<p><a href="https://github.com/JetBrains/teamcity-queue-pauser">Sources on GitHub</a> | <a href="https://www.jetbrains.com/help/teamcity/working-with-build-queue.html#Pausing+and+Resuming+Build+Queue">TeamCity Documentation</a></p>
</dd>


<dt><b>Commit Status Publisher</b></dt>
<dd>
<p>Allows TeamCity to automatically send build statuses of your commits to an external system (GitHub, GitLab, Azure DevOps, Bitbucket, and others).</p>
<p><a href="https://github.com/JetBrains/commit-status-publisher">Sources on GitHub</a> | <a href="https://www.jetbrains.com/help/teamcity/commit-status-publisher.html">TeamCity Documentation</a></p>
</dd>


<dt><b>VMware vSphere Cloud</b></dt>
<dd>
<p>Allows you to host TeamCity build agents on VMware vSphere/vCenter cloud. Cloud-hosted agents are created, started, stopped, and deleted depending on queued builds.</p>
<p><a href="https://github.com/JetBrains/teamcity-vmware-plugin">Sources on GitHub</a> | <a href="https://www.jetbrains.com/help/teamcity/setting-up-teamcity-for-vmware-vsphere-and-vcenter.html">TeamCity Documentation</a></p>
</dd>



<dt><b>Investigations Auto Assigner</b></dt>
<dd>
<p>Allows TeamCity to automatically assign investigators for build problems and test failures.</p>
<p><a href="https://github.com/JetBrains/teamcity-investigations-auto-assigner">Sources on GitHub</a> | <a href="https://www.jetbrains.com/help/teamcity/investigations-auto-assigner.html">TeamCity Documentation</a></p>
</dd>

</dl>
