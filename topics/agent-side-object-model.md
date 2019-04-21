[//]: # (title: Agent-side Object Model)
[//]: # (auxiliary-id: Agent-side+Object+Model.html)



On the agent side agent is represented by [`jetbrains.buildServer.agent.BuildAgent`](http://javadoc.jetbrains.net/teamcity/openapi/current/jetbrains/buildServer/agent/BuildAgent.html) interface. BuildAgent is available as a Spring bean and can be obtained by autowiring.

Build agent configuration can be read from the [`jetbrains.buildServer.agent.BuildAgentConfiguration`](http://javadoc.jetbrains.net/teamcity/openapi/current/jetbrains/buildServer/agent/BuildAgentConfiguration.html), it can be obtained from the __BuildAgent#getConfiguration()__ method.

## Agent side events

There is [`jetbrains.buildServer.agent.AgentLifeCycleListener`](http://javadoc.jetbrains.net/teamcity/openapi/current/jetbrains/buildServer/agent/AgentLifeCycleListener.html) interface and corresponding adapter class [`jetbrains.buildServer.agent.AgentLifeCycleAdapter`](http://javadoc.jetbrains.net/teamcity/openapi/current/jetbrains/buildServer/agent/AgentLifeCycleAdapter.html) which can be used to receive notifications about agent side events, like starting of the build, build finishing and so on. Your listener must be registered in the [`jetbrains.buildServer.util.EventDispatcher`](http://javadoc.jetbrains.net/teamcity/openapi/current/jetbrains/buildServer/util/EventDispatcher%3CAgentLifeCycleListener%3E.html). This service is also defined in the Spring context.

## Build

Each build on the agent is represented by [`jetbrains.buildServer.agent.AgentRunningBuild`](http://javadoc.jetbrains.net/teamcity/openapi/current/jetbrains/buildServer/agent/AgentRunningBuild.html) interface. You can obtain instance of __AgentRunningBuild__ by listening for __buildStarted(AgentRunningBuild)__ event in __AgentLifeCycleListener__.

## Logging to build log

Messages to build log can be sent only when a build is running. Internally agent sends messages to server by packing them into the [`jetbrains.buildServer.messages.BuildMessage1`](http://javadoc.jetbrains.net/teamcity/openapi/current/jetbrains/buildServer/messages/BuildMessage1.html) structures. However instead of creating __BuildMessage1__ structures it is better and easier to use corresponding methods in [`jetbrains.buildServer.agent.BuildProgressLogger`](http://javadoc.jetbrains.net/teamcity/openapi/current/jetbrains/buildServer/agent/BuildProgressLogger.html) which can be obtained from the __AgentRunningBuild__.

If you want to construct your own messages you can use static methods of [`jetbrains.buildServer.messages.DefaultMessagesInfo`](http://javadoc.jetbrains.net/teamcity/openapi/current/jetbrains/buildServer/messages/DefaultMessagesInfo.html) class for that.
