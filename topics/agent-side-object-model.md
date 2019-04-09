[//]: # (title: Agent-side Object Model)
[//]: # (auxiliary-id: Agent-side+Object+Model.html)



On the agent side agent is represented by `jetbrains.buildServer.agent.BuildAgent` interface. BuildAgent is available as a Spring bean and can be obtained by autowiring.

Build agent configuration can be read from the `jetbrains.buildServer.agent.BuildAgentConfiguration`, it can be obtained from the __BuildAgent#getConfiguration()__ method.

## Agent side events

There is `jetbrains.buildServer.agent.AgentLifeCycleListener` interface and corresponding adapter class `jetbrains.buildServer.agent.AgentLifeCycleAdapter` which can be used to receive notifications about agent side events, like starting of the build, build finishing and so on. Your listener must be registered in the `jetbrains.buildServer.util.EventDispatcher<AgentLifeCycleListener>`. This service is also defined in the Spring context.

## Build

Each build on the agent is represented by `jetbrains.buildServer.agent.AgentRunningBuild` interface. You can obtain instance of __AgentRunningBuild__ by listening for __buildStarted(AgentRunningBuild)__ event in __AgentLifeCycleListener__.

## Logging to build log

Messages to build log can be sent only when a build is running. Internally agent sends messages to server by packing them into the `jetbrains.buildServer.messages.BuildMessage1` structures. However instead of creating __BuildMessage1__ structures it is better and easier to use corresponding methods in `jetbrains.buildServer.agent.BuildProgressLogger` which can be obtained from the __AgentRunningBuild__.

If you want to construct your own messages you can use static methods of `jetbrains.buildServer.messages.DefaultMessagesInfo` class for that.
