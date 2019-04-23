[//]: # (title: External Storage Implementation Guide)
[//]: # (auxiliary-id: External+Storage+Implementation+Guide.html)

__Since TeamCity 2017.1,__ an API is provided to enable writing TeamCity plugins which can store TeamCity build artifacts in a custom storage. This guide details implementation of support for an external storage system as a TeamCity plugin.

You can use the following plugins from JetBrains as implementation examples:
* [S3 Artifact Storage](https://plugins.jetbrains.com/plugin/9623-s3-artifact-storage)
* [Azure Artifact Storage](https://plugins.jetbrains.com/plugin/9617-azure-artifact-storage)
* [Google Cloud Artifact Storage](https://plugins.jetbrains.com/plugin/9634-google-artifact-storage)


## TeamCity Artifacts Overview

TeamCity provides the following artifacts\-related features:
* Artifacts upload

* Individual artifacts download and browsing of build artifacts in a web browser and via the [REST API](https://www.jetbrains.com/help/teamcity/?rest-api)

* Ability to configure [artifact dependencies](https://www.jetbrains.com/help/teamcity/?artifact-dependencies) between builds and fetching necessary dependencies on the agent

Upload to a TeamCity server is a process of storing data created by a build, so that it is available after a TeamCity agent is disconnected. The data is uploaded from the agent to the server via HTTP multipart requests. Usually the upload process starts when a build finishes on the agent, but it is also possible to initiate the upload while the build is in progress using [service messages](https://confluence.jetbrains.com/display/TCD10/Build+Script+Interaction+with+TeamCity#BuildScriptInteractionwithTeamCity-artPublishingPublishingArtifactswhiletheBuildisStillinProgress). Artifacts are uploaded according to [artifacts paths](https://confluence.jetbrains.com/display/TCD10/Configuring+General+Settings#ConfiguringGeneralSettings-artifactPaths). Uploaded artifacts are also cached on the build agent in case they are requested by another build on this agent via artifact dependencies.

Uploaded data is displayed in the TeamCity web UI as an artifacts tree in the popups or on the Artifacts tab of the build results. It also can be accessed by [http requests](https://confluence.jetbrains.com/display/TCD10/Patterns+For+Accessing+Build+Artifacts) via the [REST API](https://confluence.jetbrains.com/display/TCD10/REST+API#RESTAPI-build_artifacts) and as an [ivy-compliant repository](https://confluence.jetbrains.com/display/TCD10/Artifact+Dependencies#ArtifactDependencies.mdConfiguringArtifactDependenciesUsingAntBuildScript).

TeamCity can also deliver artifacts of one build to another build with the help of [Artifacts dependencies](https://confluence.jetbrains.com/display/TCD10/Artifact+Dependencies). Artifact dependency configuration includes the source build settings (build configuration, version), artifact patterns for matching the source build artifacts, and destination paths on the target agent. Downloaded artifact dependencies are cached on build agents to reduce the download time.

Build artifacts also contain a number of [internal artifacts](https://confluence.jetbrains.com/display/TCD10/Build+Artifact#BuildArtifact-HiddenArtifacts). They include (but not limited to) build logs, build properties, coverage reports, etc. These artifacts are required for TeamCity features to function properly, and unless specified explicitly, they are not removed by clean\-up and not downloaded as dependencies.

Build artifacts can be cleaned up according to [Cleanup Rules](https://confluence.jetbrains.com/display/TCD10/Clean-Up#Clean-Up-ProjectClean-upRules).

## External Artifacts Storage Overview

An implementation of an external storage should be able to upload to, download, and remove artifacts from the external storage.

The external storage plugin should be able to upload artifacts to the storage during a build on agent, send them to the client on a request  and handle the cleanup of the artifacts.

While delegating some of its features to the plugin, TeamCity keeps a part of its internal functions intact. Regardless of external storage settings, internal artifacts (including build logs) are still published to the TeamCity server. Besides, currently the artifacts tree is rendered in the web UI by the TeamCity server itself.

## Implementation

### Settings

The TeamCity artifacts storage is configured on the "Project Settings" page under the dedicated tab. Using the tab, a TeamCity user can choose which storage will be used for builds artifacts. Also, the relevant settings page is displayed there. The choice will be applied to all build configurations in the project and its subprojects.

To get listed on this page, the plugin will provide a spring bean implementing the `jetbrains.buildServer.serverSide.artifacts.ArtifactStorageType` abstract class and should be registered in the `jetbrains.buildServer.serverSide.artifacts.ArtifactStorageTypeRegistry`.

### Publication

Publication is done from the build agent process. The plugin should provide a spring bean implementing the interface `jetbrains.buildServer.agent.ArtifactsPublisher` (for future compatibility we recommend extending the base implementation `jetbrains.buildServer.agent.ArtifactsPublisherBase`). The plugin should publish information about remote artifacts using `jetbrains.buildServer.agent.artifacts.AgentArtifactHelper#publishArtifactList` after a build is finished. This information will be stored in a special index file as a hidden build artifact and can be accessed later on.

### View

If the external artifacts index was created during publication using `jetbrains.buildServer.agent.artifacts.AgentArtifactHelper#publishArtifactList`, it will be used by the TeamCity server when listing build artifacts, e.g. when adding nodes to the artifacts tree.

To access artifact content via HTTP requests, the plugin should provide an implementation of the `jetbrains.buildServer.web.openapi.artifacts.ArtifactDownloadProcessor` interface. To access artifact content for other purposes, it should provide an implementation of the `jetbrains.buildServer.serverSide.artifacts.ArtifactContentProvider` interface.

### Cleanup

For cleanup, the plugin is expected to have a Spring bean implementing `jetbrains.buildServer.serverSide.cleanup.CleanupExtension`. It is recommended to make this bean [PositionAware](http://javadoc.jetbrains.net/teamcity/openapi/10.0/jetbrains/buildServer/util/positioning/PositionAware.html) and place it [first](http://javadoc.jetbrains.net/teamcity/openapi/10.0/jetbrains/buildServer/util/positioning/PositionConstraint.html#first()) to make sure the extension is called before the default TeamCity clean\-up procedures (that will remove builds and data stored on the disk).

The implementation should use `jetbrains.buildServer.serverSide.cleanup.BuildCleanupContext#getErrorReporter` to report errors which occurred during the clean\-up, and `jetbrains.buildServer.serverSide.artifacts.ServerArtifactHelper#removeFromArtifactList` to remove artifacts which were successfully cleaned up from the storage from the artifact list stored in the TeamCity.

 

 

 
