---
title: Running an IDE with an IDE
layout: default
---

### Running an IDE with an IDE

JetBrains provides a [Toolbox app](https://www.jetbrains.com/toolbox-app/) that can allow you to experiment with different Android Studio and IntelliJ versions. Another way
to experiment with different versions of IDEs is to launch them within Android Studio as well! You can accomplish this goal by using the Intellij Gradle Plugin. It allows you to launch an IDE within
Android Studio, which will automate the downloading of these versions for you.

1. Add the IntelliJ Gradle plugin to your `build.gradle.kts` file (see [source](https://github.com/rogerhu/intellij-gradle-plugin-demo/blob/master/build.gradle.kts#L3-L12)) as an example):

   ```gradle

   plugins {
      kotlin("jvm") version "1.7.0-Beta"
      id("org.jetbrains.intellij") version "1.14.0"
   }

   intellij {
      // See https://jb.gg/android-studio-releases-list.xml
      version.set("2022.3.1.18") // Giraffe
      type.set("AI")
      downloadSources.set(true)
   }
   ```

2. Create a `runIde` task:

   <img width="796" alt="image" src="https://github.com/rogerhu/studying-android-studio-internals/assets/326857/c9aa8dfd-2819-4794-ab5b-4fa621a4591d">

   <img width="1037" alt="image" src="https://github.com/rogerhu/studying-android-studio-internals/assets/326857/0a834457-f5e1-4de3-96b2-5b6ff72232cc">

3. Set the runIde task to include the jvmArgs:

   ```groovy
   tasks.named('runIde') {
     jvmArgs = ['-agentlib:jdwp=transport=dt_socket,server=y,suspend=n,address=5006']
   }
   ```

   ```kotlin
   tasks.named<RunIdeTask>("runIde").configure {
     jvmArgs = listOf("-agentlib:jdwp=transport=dt_socket,server=y,suspend=n,address=5006")
   }
   ```

Jump to step 3 described in the [quickstart](/#step-3-attach-debugger) to attach breakpoints!