[//]: # (title: Implement Cloud Support)
[//]: # (auxiliary-id: Implement Cloud Support)

> The described API may be changed in future TeamCity releases.
>
{type="note"}

This page explains how to create a plugin that allows you to run TeamCity agents on a cloud. You can use the open-source [Google Cloud Agents](https://github.com/JetBrains/teamcity-google-agent) plugin as a reference.

> We are looking to improve this documentation and API. Feel free to post comments with questions and suggestions.
>
{type="note"}



## Cloud Structure

We define the following notions:

* **Cloud profile** &mdash; Contains general cloud integration settings (for example, security credentials).
* **Cloud image** &mdash; Allows you to define configuration parameters for an image source (for example, the available amount of RAM).
* **Cloud instance** &mdash; An instance of the running cloud image.

One cloud profile can include multiple cloud images, and each image can have multiple instances.

## Create a Plugin Project

Create [a TeamCity plugin project](getting-started-with-plugin-development.md) with at least one agent-side and one server-side module.

## Add Library References

If you are not using the [Gradle TeamCity plugin](https://github.com/rodm/gradle-teamcity-plugin), add the TeamCity Maven repository to your project:

```xml
<repositories>
   <repository>
      <id>teamcity</id>
      <name>TeamCity</name>
      <url>https://download.jetbrains.com/teamcity-repository/</url>
      <layout>default</layout>
      <snapshots>
         <enabled>true</enabled>
         <updatePolicy>always</updatePolicy>
      </snapshots>
   </repository>
</repositories>
```

Then add the following libraries to your project modules.

**Agent-side module**

```groovy
provided "org.jetbrains.teamcity:cloud-interface:2017.2"
provided "org.jetbrains.teamcity:cloud-shared:2017.2"
provided "org.jetbrains.teamcity.internal:agent:2017.2"
```


```xml
<dependency>
   <groupId>org.jetbrains.teamcity</groupId>
   <artifactId>cloud-interface</artifactId>
   <version>2017.2</version>
   <scope>provided</scope>
</dependency>
<dependency>
   <groupId>org.jetbrains.teamcity</groupId>
   <artifactId>cloud-shared</artifactId>
   <version>2017.2</version>
   <scope>provided</scope>
</dependency>
<dependency>
   <groupId>org.jetbrains.teamcity.internal</groupId>
   <artifactId>agent</artifactId>
   <version>2017.2</version>
   <scope>provided</scope>
</dependency>
```

**Server-side module**


```groovy
provided "org.jetbrains.teamcity:cloud-interface:2017.2"
provided "org.jetbrains.teamcity:cloud-shared:2017.2"
provided "org.jetbrains.teamcity.internal:server:2017.2"
```

```xml
<dependency>
  <groupId>org.jetbrains.teamcity</groupId>
  <artifactId>cloud-interface</artifactId>
  <version>2017.2</version>
  <scope>provided</scope>
</dependency>
<dependency>
  <groupId>org.jetbrains.teamcity</groupId>
  <artifactId>cloud-shared</artifactId>
  <version>2017.2</version>
  <scope>provided</scope>
</dependency>
<dependency>
  <groupId>org.jetbrains.teamcity.internal</groupId>
  <artifactId>server</artifactId>
  <version>2017.2</version>
  <scope>provided</scope>
</dependency>
```

## How to Implement a Plugin for Cloud Support

1. Implement the `jetbrains.buildServer.clouds.CloudClientFactory` interface. This interface exposes the following methods:

    * `getCloudCode` &mdash; Returns the cloud plugin id with maximum length of 5 chars, i.e. "wmw".
    * `getDisplayName` &mdash; Returns a user-friendly name of the plugin .
    * `getEditProfileUrl` &mdash; Returns a context-path to the controller that will provide the cloud profile settings fields for the form.
    * `getInitialParameterValues` &mdash; Returns default values for form fields if any
      getPropertiesProcessor returns form values validator.
    * `canBeAgentOfType` &mdash; Checks whether an agent is running on some instance of this could. If true is returned, TeamCity will create a CloudClient and query it for details.
    * `createNewClient` &mdash; Creates a new `jetbrains.buildServer.clouds.CloudClientEx` cloud client instance from cloud profile parameters. TeamCity creates one client per profile.

2. Register the implemented `CloudClientFactory` class in `jetbrains.buildServer.clouds.CloudRegistrar`. You can get an instance of the CloudRegistrar interface in the constructor of your plugin's [Spring bean](getting-started-with-plugin-development.md).

## Implement the CloudClientEx Interface

The `CloudClientEx` interface is inherited from the base `jetbrains.buildServer.clouds.CloudClient` interface. `CloudClient` exposes only read-only operations, while all action methods are declared in the `CloudClientEx` interface.


> TeamCity expects high performance from methods of your custom `CloudClientEx` implementation. To match these performance expectations, we recommend that your custom implementation has its own thread/thread pool that allows methods to operate asynchronously and update the cloud state.
>
{type="tip"}

### Action Methods

* `startNewInstace(CloudImage image, CloudInstanceUserData tag)` &mdash; Returns a CloudInstance object in the SCHEDULED_TO_START, or STARTING, or RUNNING state. If an instance requires a significant amount of time to start, implement the `ClouldClientEx` interface asynchronously. The `CloudInstanceUserData` parameter contains properties to be set into a build agent that is runs on the virtual machine.

* `terminateInstance(CloudInstance instance)` &mdash; Stops a cloud instance.

* `restartInstance(CloudInstance instance)` &mdash; Restarts a virtual machine.

### Readonly methods


* `isInitialized()` &mdash; Returns whether a plugin is fully initialized. A cloud UI shows 'Loading' while this method returns `false`.

* `getImages()` &mdash; Returns a list of all images registered for the cloud client.

* `getErrorInfo()` &mdash; If any communication error(s) occurred, this method returns information about these errors.

* `canStartNewInstance(CloudImage image)` &mdash; TeamCity calls this method to check whether it is possible to start one mode instance of the provided image. If this method returns `true`, TeamCity schedules the `startNewInstance` method.

### Find methods

* `findImageById` &mdash; TeamCity calls this method to find a running CloudInstance for the registered build agent. This method should work as fast as possible.

## Build Agent mappings

TeamCity maintains mappings between CloudInstances and registered build agents. This mapping is supported by calling:

* `CloudType#canBeAgentOfType` method
* `CloudClient#findInstanceByAgent` method

### Cloud Instance

The `jetbrains.buildServer.clouds.CloudInstance` interface describes an instance that is detected in the cloud. A known cloud instance is a *cloud image*.


An instance has one of the following states:

| State              | Description                                                     |
|--------------------|-----------------------------------------------------------------|
| UNKNOWN            | The implementation fails to determine the status                |
| SCHEDULED_TO_START | Use this status for a newly created instance                    |
| STARTING           | The cloud system has reported a machine as starting             |
| RUNNING            | The cloud system has reported a machine as running (booted)     |
| RESTARTING         | A machine is restarting                                         |
| SCHEDULED_TO_STOP  | Use this status for instances that received the stop request.   |
| STOPPING           | A stop command was sent to the cloud system                     |
| STOPPED            | The cloud system has reported that the machine has been stopped |


### Cloud Image

The cloud image `jetbrains.buildServer.clouds.CloudImage` interface describes a kind of image for virtual machines to start.

### Cloud Profile

The cloud profile `jetbrains.buildServer.clouds.CloudProfile` interface describes settings. Each profile contains the cloud type and holds user-configurable parameters. You do not need to implement this interface.

### Cloud Client

The mandatory cloud client `jetbrains.buildServer.clouds.CloudClientEx` interface is required for CloudImages and CloudInstances. You need to implement this interface in your plugin.

## How to Associate a Build Agent with a Cloud Instance

You can set a dedicated property in the [buildAgent.properties](https://www.jetbrains.com/help/teamcity/configure-agent-installation.html) file of the build agent started inside the cloud. This parameter can be checked on the server to match build agents with CloudInstances.

In TeamCity Amazon EC2 integration, this is done from the build agent plugin. The plugin fetches Amazon Instance metadata (including Amazon EC2 `instanceId`) and publishes the metadata to build agent custom properties. Build agents are matched according to the machine instanceId provided as build agent custom properties.

You can change the `buildAgent.properties` file in a running virtual machine (guest OS) from a host machine to publish properties that will help to match a build agent to a cloud image.

You can also write a custom guest OS service/daemon that will update build agent properties when the machine starts.

## Configuration Parameters

Use `jetbrains.buildServer.clouds.CloudClientParameters` to obtain security credentials and cloud image details. If you need to implement a UI that allows users to set these parameters, use HTML input elements with "prop:" prefixes or TeamCity JSP tags.

### Image Parameters

To store image details, define an HTML element the `jetbrains.buildServer.clouds.CloudImageParameters#SOURCE_IMAGES_JSON` name. This element should store the list of JSON objects with image parameters.

Each image should declare the following parameters:

* `agent_pool_id` &mdash `jetbrains.buildServer.clouds.CloudImageParameters#AGENT_POOL_ID_FIELD`. Stores id of agent pool associated with cloud image.
* `source-id` &mdash; `jetbrains.buildServer.clouds.CloudImageParameters#SOURCE_ID_FIELD`. The unique identifier of cloud image.
* `profileId` &mdash; The cloud profile identifier.

## Plugin Building & Distribution

Cloud support is implemented as a regular [TeamCity plugin](developing-teamcity-plugins.md) that you can publish in the [TeamCity plugins gallery](https://plugins.jetbrains.com/teamcity).
