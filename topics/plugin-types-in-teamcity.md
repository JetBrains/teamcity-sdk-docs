[//]: # (title: Plugin Types in TeamCity)
[//]: # (auxiliary-id: Plugin+Types+in+TeamCity.html)



TeamCity build system consists of two parts:
1. The server that gathers information while builds are running
2. Agents that run builds and send information to the server


Consequently, depending on where the code runs, there are
* server\-side plugins
* agent\-side plugins.

Besides that, plugins are divided into the following types:
* Build runners
* VCS plugins
* Notifiers
* User authentication plugins
* Build Triggers
* Extensions, which can modify some aspects of TeamCity behavior. There are several extension points on the server and on the agent, allowing you, for example, to format the stack trace on the web the way you need or modify the build status text. [Read more](extensions.md).

Plugins can also modify the TeamCity web UI. They can provide custom content to the existing pages (again, there are several extension points provided for that), or create new pages with their own UI. [Read more](web-ui-extensions.md).
