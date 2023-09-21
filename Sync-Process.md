---
title: Understanding the Sync Process
layout: default
---

# Overview

The sync process is what enables all the rich features of the IntelliJ IDE, including syntax highlighting, autocomplete, and code navigation. The build system (e.g. Bazel, Gradle) is
responsible for transferring its knowledge of the state of the world into models that can be processed by IntelliJ. The sync and indexing process needs to complete before you can
take advantage of using IntelliJ, so it's important to understand what's happening behind the scenes.

For the Gradle build system, the information about subprojects, build targets, and dependencies are transferred with Gradle's [Tooling API](https://docs.gradle.org/current/userguide/third_party_integration.html#embedding). The Tooling API provides a programmatic way for external applications to get access to information instead of needing to parse information from `.gradle` files. Essentially a client can request models
that have been previously registered with Gradle (e.g. see [this code snippet](https://github.com/square/affected-paths/blob/1dfb915c005730ee59b6155749880eb5d8d7b8bd/tooling/support/src/main/kotlin/com/squareup/tooling/support/SquareToolingPlugin.kt#L38) as an example).

One limitation is that the sync process currently does not take advantage of configuration caching. The sync process is also quite complex and relies a lot on different approaches (e.g. dynamic loading of model builder classes) to create the models. You can begin observing how this process works by reviewing the IDE logs and noticing how the Gradle Tooling API being invoked with an `ijinit.gradle` script:

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

This script loads the [ExtraModelBuilder](https://github.com/JetBrains/intellij-community/blob/53f8ee629ca0aafcf661b6f88c8fbf63d2b7b232/plugins/gradle/tooling-extension-impl/src/com/intellij/gradle/toolingExtension/impl/modelBuilder/ExtraModelBuilder.java#L53-L54), which in turn tries to find classes that extend `ExtraModelBuilder` or `ModelBuilderService`. Attaching a breakpoint to the Gradle daemon shows the different model builders involved, some of which are needed for Kotlin language support.

<img width="1171" alt="image" src="https://github.com/rogerhu/studying-android-studio-internals/assets/326857/bb296d04-9092-4c87-a76a-ef63c37f3c70">

These model builders are only invoked if they're requested by the client. Notice in [this code](https://github.com/JetBrains/intellij-community/blob/01288a51dc574d4ab37ea73a389f00bbfc7d10f5/plugins/gradle/tooling-extension-impl/src/com/intellij/gradle/toolingExtension/impl/modelBuilder/ExtraModelBuilder.java#L62-L64) how the `canBuild()` method is invoked, which primarily checks to see if the model name requested matches
that of what the model builder provides.

```java
for (ModelBuilderService service : modelBuilderServices) {
    if (service.canBuild(modelName)) return true;
}
```
