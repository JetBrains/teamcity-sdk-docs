[//]: # (title: Custom Build Trigger)
[//]: # (auxiliary-id: Custom+Build+Trigger.html)

An example of a trigger plugin can be found in [Url Build Trigger](https://plugins.jetbrains.com/plugin/9074-url-build-trigger).



## Build Trigger Service



Build trigger is a service whose purpose is to trigger builds (add builds to the queue). Build trigger must extend `jetbrains.buildServer.buildTriggers.BuildTriggerService` abstract class. Build trigger service is uniquely identified by trigger name (see __getName__ method). There is no need to register BuildTriggerService, instead plugin should provide a class extending the `jetbrains.buildServer.buildTriggers.BuildTriggerService` defined as a Spring bean.



## Build Trigger Settings



Build trigger settings is an object containing build trigger parameters specified by a user via the web UI. Build trigger settings are represented by `jetbrains.buildServer.buildTriggers.BuildTriggerDescriptor` class. The settings are contained within a map of string parameters. More than one instance of `jetbrains.buildServer.buildTriggers.BuildTriggerDescriptor` corresponding to the same trigger service can be added to the build configuration or a template. Instances of the `jetbrains.buildServer.buildTriggers.BuildTriggerDescriptor` with the same trigger name and parameters are considered equal. With help of `jetbrains.buildServer.buildTriggers.BuildTriggerService` trigger service can allow or disallow multiple trigger settings per build configuration.



Each trigger service can provide an URL to a jsp or custom controller which will show the trigger web UI. The approach is similar to the one used for VCS roots and build runners.



## Triggering Policy



Build trigger service must return a policy (`jetbrains.buildServer.buildTriggers.BuildTriggeringPolicy`) which will be used to add builds to the queue. Currently only one policy is available: `jetbrains.buildServer.buildTriggers.PolledBuildTrigger`. More policies can be added in the future.



Trigger returning `jetbrains.buildServer.buildTriggers.PolledBuildTrigger` policy will be polled by the server with regular intervals. Trigger will receive `jetbrains.buildServer.buildTriggers.PolledTriggerContext` object which contains all information necessary to make a decision whether a build must be triggered or not. Trigger can use `jetbrains.buildServer.serverSide.SBuildType` method to add builds to the queue. Note that `jetbrains.buildServer.buildTriggers.PolledTriggerContext` also provides access to the custom data storage. This storage can be used for build trigger state associated with a build configuration and trigger settings. Custom storage will be automatically persisted and restored upon server restart.
