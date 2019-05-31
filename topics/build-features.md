[//]: # (title: Build Features and Failure Conditions)
[//]: # (auxiliary-id: Build+Features.html)

A [build feature](https://www.jetbrains.com/help/teamcity/adding-build-features.html) allows to modify behavior of a build on the server or on the agent side.
There is no specific workflow associated with a build feature, a plugin implementing a build feature can plug itself into various listeners and somehow affect the build.

An example of a build feature is [XML Report processing](https://github.com/JetBrains/teamcity-xml-tests-reporting) build feature. It detects various XML reports on the agent 
while build is running, parses them and imports results into TeamCity server. 

Note: build failure conditions, such as "Fail build on metric change" or "Fail build on a specific text in build log" are implemented as build features under the hood. 

## Build Feature

To add a custom build feature one should inherit from [`jetbrains.buildServer.serverSide.BuildFeature`](http://javadoc.jetbrains.net/teamcity/openapi/current/jetbrains/buildServer/serverSide/RunType.html). A build runner plugin must provide its own RunType and register it in the [`jetbrains.buildServer.serverSide.RunTypeRegistry`](http://javadoc.jetbrains.net/teamcity/openapi/current/jetbrains/buildServer/serverSide/BuildFeature.html)
and implement abstract methods. The class should be registered as a Spring bean in the plugin spring context.

In general build features are similar to runners and triggers. For instance, they also have a type which should be unique among all of the features and methods 
working with parameters are almost the same.

But there are some methods that may require additional description:
* [`isMultipleFeaturesPerBuildTypeAllowed`](http://javadoc.jetbrains.net/teamcity/openapi/current/jetbrains/buildServer/serverSide/BuildFeature.html#isMultipleFeaturesPerBuildTypeAllowed--) - in most of the cases there can be only one build feature defined per build configuration, 
but if this is not the case then this method should return false
* [`getPlaceToShow`](http://javadoc.jetbrains.net/teamcity/openapi/current/jetbrains/buildServer/serverSide/BuildFeature.html#getPlaceToShow--) - this method determines in which part of administration interface this feature should be configured, 
use [`FAILURE_REASON`](http://javadoc.jetbrains.net/teamcity/openapi/current/jetbrains/buildServer/serverSide/BuildFeature.PlaceToShow.html#FAILURE_REASON) if the build feature is a failure condition and should be configured where all failure conditions reside
* [`isRequiresAgent`](http://javadoc.jetbrains.net/teamcity/openapi/current/jetbrains/buildServer/serverSide/BuildFeature.html#isRequiresAgent--) - some features do not have an agent part and do not require agents to operate, if so, this method should return false
* [`getRequirements`](http://javadoc.jetbrains.net/teamcity/openapi/current/jetbrains/buildServer/serverSide/BuildFeature.html#getRequirements-java.util.Map-) - if build feature has an agent part, with help of this method it is possible to affect agents available to a build where this feature is configured     

## Accessing Build Features via SBuildType

Once a build feature is configured in a build configuration, it is possible to access it with the following methods of [`SBuildType`](http://javadoc.jetbrains.net/teamcity/openapi/current/jetbrains/buildServer/serverSide/SBuildType.html):
* [`getBuildFeatures`](http://javadoc.jetbrains.net/teamcity/openapi/current/jetbrains/buildServer/serverSide/BuildTypeSettings.html#getBuildFeatures--) - this returns the whole set of configured build features (including disabled)
* [`getBuildFeaturesOfType`](http://javadoc.jetbrains.net/teamcity/openapi/current/jetbrains/buildServer/serverSide/BuildTypeSettings.html#getBuildFeaturesOfType-java.lang.String-) - this returns only the build features of specified type (including disabled)
* [`findBuildFeatureById`](http://javadoc.jetbrains.net/teamcity/openapi/current/jetbrains/buildServer/serverSide/BuildTypeSettings.html#findBuildFeatureById-java.lang.String-) - each build feature has an internal ID associated with it, this method can be used to find a feature by such an ID

Note: in API no distinction is made among build features and failure conditions. They are all treated as build features.

Note: build features can be enabled and disabled. Methods above always return all configured build features, does not matter whether they are disabled or not.

### Parameters resolving

Build features can have parameter references in their settings, for instance: ``%some.path%``. But methods like [`SBuildType#getBuildFeatures`](http://javadoc.jetbrains.net/teamcity/openapi/current/jetbrains/buildServer/serverSide/BuildTypeSettings.html#getBuildFeatures--) 
or [`SBuildType#getBuildFeaturesOfType`](http://javadoc.jetbrains.net/teamcity/openapi/current/jetbrains/buildServer/serverSide/BuildTypeSettings.html#getBuildFeaturesOfType-java.lang.String-) 
return build features with unresolved parameters. This is because build features parameters can depend on parameters from agents and in general without an agent 
it is not clear how parameters resolution should be performed.

But it is still possible to retrieve features with all parameters resolved by taking only server side parameters into account via [`SBuildType#getResolvedSettings`](
http://javadoc.jetbrains.net/teamcity/openapi/current/jetbrains/buildServer/serverSide/ResolvedSettings.html).

Note: in this case [`getBuildFeatures`](http://javadoc.jetbrains.net/teamcity/openapi/current/jetbrains/buildServer/serverSide/ResolvedSettings.html#getBuildFeatures--) 
and [`getBuildFeaturesOfType`](http://javadoc.jetbrains.net/teamcity/openapi/current/jetbrains/buildServer/serverSide/ResolvedSettings.html#getBuildFeaturesOfType-java.lang.String-)
will return enabled build features only.

## Accessing Build Features via SBuild

The following method can be used to retrieve build features of a running or finished build:
* [`getBuildFeaturesOfType`](http://javadoc.jetbrains.net/teamcity/openapi/current/jetbrains/buildServer/serverSide/SBuild.html#getBuildFeaturesOfType-java.lang.String-) - this method 
returns build features of specified type with all parameters resolved according to currently known parameters of agent where the build is running; only enabled features are returned

## Accessing Build Features on Agent

On the agent side, build is represented by [`AgentRunningBuild`](http://javadoc.jetbrains.net/teamcity/openapi/current/jetbrains/buildServer/agent/AgentRunningBuild.html) interface.
To access build features the following methods can be used:
* [`getBuildFeatures`](http://javadoc.jetbrains.net/teamcity/openapi/current/jetbrains/buildServer/agent/AgentRunningBuild.html#getBuildFeatures--) 
* [`getBuildFeaturesOfType`](http://javadoc.jetbrains.net/teamcity/openapi/current/jetbrains/buildServer/agent/AgentRunningBuild.html#getBuildFeaturesOfType-java.lang.String-) 

Both of these methods work similarly to methods in [`SBuild`](http://javadoc.jetbrains.net/teamcity/openapi/current/jetbrains/buildServer/serverSide/SBuild.html) - they return features 
with all parameters resolved and they don't return disabled features.


