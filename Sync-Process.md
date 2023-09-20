---
title: Understanding the Sync Process
layout: default
---

The sync process is used to transfer knowledge of your build configurations (e.g. build targets, dependencies, subprojects) in a way that can be used by IntelliJ. Currently these models are transferred with Gradle's [Tooling API](https://docs.gradle.org/current/userguide/third_party_integration.html#embedding) and does not take advantage configuration caching. The sync process is quite complex and relies a lot on different approaches (e.g. dynamic loading of model builder classes) to provide this information.

We can observe how the sync process works by reviewing the IDE logs and noticing how the Gradle Tooling API being invoked with an `ijinit.gradle` script:

```
2023-09-11 08:33:35,293 [  31980]   INFO - #o.j.p.g.s.e.GradleExecutionHelper - Passing command-line args to Gradle Tooling API: 
--init-script /private/var/folders/1v/v3_87mt519v6gvh3yl0vyr480000gn/T/ijmapper.gradle -Didea.sync.active=true -Didea.resolveSourceSetDependencies=true 
-Porg.gradle.kotlin.dsl.provider.cid=23919243869208 --init-script /private/var/folders/1v/v3_87mt519v6gvh3yl0vyr480000gn/T/sync.studio.tooling.gradle 
-Djava.awt.headless=true --continue --stacktrace -Pandroid.injected.build.model.only=true -Pandroid.injected.build.model.only.advanced=true 
-Pandroid.injected.invoked.from.ide=true -Pandroid.injected.build.model.only.versioned=3 
-Pandroid.injected.build.model.disable.src.download=true -Pidea.gradle.do.not.build.tasks=true 
-Dorg.gradle.internal.GradleProjectBuilderOptions=omit_all_tasks -Pkotlin.mpp.enableIntransitiveMetadataConfiguration=true 
--init-script /private/var/folders/1v/v3_87mt519v6gvh3yl0vyr480000gn/T/ijinit.gradle
```

This script loads the [ExtraModelBuilder](https://github.com/JetBrains/intellij-community/blob/53f8ee629ca0aafcf661b6f88c8fbf63d2b7b232/plugins/gradle/tooling-extension-impl/src/com/intellij/gradle/toolingExtension/impl/modelBuilder/ExtraModelBuilder.java#L53-L54), which in turn tries to find classes that extend `ExtraModelBuilder` or `ModelBuilderService`.

The core of this code change lives in [SyncActionRunner.kt](https://cs.android.com/android-studio/platform/tools/adt/idea/+/mirror-goog-studio-main:project-system-gradle-sync/src/com/android/tools/idea/gradle/project/sync/SyncActionRunner.kt;l=222-225?q=syncactionrunner). Currently only Android modules can be fetched in parallel. 

* Why the sync process is necessary
* What is the sync process (mapping information from Gradle build system to IntelliJ data models - projects, modules, facets, content roots, dependencies, use of Gradle tooling API, facets)
* Importing vs. syncing



* Listening to PROJECT_SYSTEM_SYNC_TOPIC and using `ExternalSystemProgressNotificationManager.getInstance().addNotificationListener(gradleSystemListener);` - https://github.com/gradle/gradle-profiler/commit/23bbe61c2455e180d5c761b64e2f9f2516ea35a8

* https://android.googlesource.com/platform/tools/adt/idea/+/107623b47b97536aecd67b86b82f99b8449b0657

* https://android.googlesource.com/platform/tools/adt/idea/+/46125d8bc128093cc94fccdac05e21dae2a3aeb4 -> `With this CL, all Gradle models are serialized to disk when syncing a project using the "new sync" infrastructure. When reopening a project, the models from the cache will be used, instead of performing a full Gradle Sync.`

* https://android.googlesource.com/platform/tools/adt/idea/+/47cbf9a4c69a3334f8d2017deaf61ba852b71dbb -> `Implement basic Kotlin Support for new sync`

* https://android.googlesource.com/platform/tools/adt/idea/+/4d6f053a362d15a8c720f011d784a6bdce07b025 -> `Add Kapt support to new sync`

* https://android.googlesource.com/platform/tools/adt/idea/+/20247d5c7087273fc08b05e4b21fae79458c5576 -> `Enable new sync for non-MPP Kotlin projects`

* https://android.googlesource.com/platform/tools/adt/idea/+/63abed69879f4d7cd5dc7d78cddfeea30b571bd6 -> `Exclude build/ folder directly`

* https://android.googlesource.com/platform/tools/adt/idea/+/caad4eedc5b2d57fab80fdcf235aa78e3e2a8f86 -> `Switch to IDEA sync if buildSrc module exists`

* https://android.googlesource.com/platform/tools/adt/idea/+/48f7db223eed5176ad47cb87c771b1c31c01f831 -> `This change setup build script classpath properly in new sync`

* https://android.googlesource.com/platform/tools/adt/idea/+/48f7db223eed5176ad47cb87c771b1c31c01f831/9dbc4031748df84487082759a68f546429398519 -> `Split IdeAndroidProject from gradle tooling model.`

* https://android.googlesource.com/platform/tools/adt/idea/+/73a0bdad51ab59ba3c7d1012eed0200fea3c21b0 -> `This is to make it possible to restructure the process to be parallelizable with the new experimental Gradle API. This is both to allow introduction of v2 models and to make multi-threaded access to the model cache explicit when using the new experimental Gradle API for parallel sync.`
 
* https://android.googlesource.com/platform/tools/adt/idea/+/223921711b79f38f1cd5ccd32d3cba859084a40c -> `Introduce ActionRunner to abstract execution of build actions and allow an implementation that build actions that request build models in parallel.`

* https://android.googlesource.com/platform/tools/adt/idea/+/259979e60a263243efcdc7d6750c2a4eb39abe1de -> `Prepare variant syncing for parallel sync. Parallel sync relies on ability to run multiple build actions in parallel. However to have multiple build actions to run we need to know which build variant to request for each module upfront and this is not possible without first syncing modules depending on it.`

* https://android.googlesource.com/platform/tools/adt/idea/+/f423d004554d9202f9ed92a2085d854c453e7b97 -> `Split AndroidExtraModelProvider
into two top level classes: a provider, a worker and a utility to run sync actions in parallel.`

* https://android.googlesource.com/platform/tools/adt/idea/+/8afc1e64950c8e95a5ccb93b22c994c5b6dc10c0 - `Move injected sync code to -sync module. This change moves the code which is injected by Android Studio into the Gradle process to a standalone module so that it can target a different JDK/byte code level.`
