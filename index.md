---
title: Home
layout: home
---

# Intro

Curious about the IntelliJ IDE works? [Pablo Baxter](https://github.com/pablobaxter) and I wrote this guide to help you better understand the IDE internals and enable you to ask more insightful questions especially when filing bug reports. Follow the Quickstart below to get started.

## Quickstart

The quickest way to get started is to use the open source versions of the Android plugin and IntelliJ Community Edition maintained by JetBrains. 

## Step 1: Grab the IntelliJ source code

1. Clone the IntelliJ repository.

   ```bash
   git clone https://github.com/JetBrains/intellij-community
   pushd intellij-community
   ```

2. Match the version used in Android Studio (see https://plugins.jetbrains.com/docs/intellij/android-studio-releases-list.html):

   For instance, to checkout the Giraffe version, you would use this command:

   ```bash
   git checkout idea/223.8836.35
   ```

3. To get the Android plugin, clone from this repo.

   ```
   git clone git://git.jetbrains.org/idea/android.git android
   ```

   NOTE: the JetBrains Android plugin is maintained separately from [Google's version](https://android.googlesource.com/platform/tools/base/+log/refs/heads/mirror-goog-studio-main).
   See the section below on how to use the Android plugin from Google.
   
## Step 2: Setup Android Studio

Because Android Studio is a Java process, you can setup config options to be to attach to it.

1. Start Android Studio and go to `Help` -> `Edit Custom VM Options...`.

   <img width="362" alt="image" src="https://github.com/rogerhu/studying-android-studio-internals/assets/326857/99c4e162-9ea8-415c-8426-a06bad74479a">

2. Add `-agentlib:jdwp=transport=dt_socket,server=y,suspend=n,address=5006` to the end of the `studio.vmoptions` file:

   <img width="1725" alt="image" src="https://github.com/rogerhu/studying-android-studio-internals/assets/326857/a1fc27a2-dfb6-4fb3-b815-1d607b390c7e">

   * Note the 5006 address can be any value. Because Gradle uses 5005 by default, we choose 5006 to avoid conflicts.
   * If you want to wait for Gradle to start, set `suspend=y`. This may be useful in case you want to put breakpoints
     against the Gradle source code to see what's happening.

3. Quit and restart Android Studio to load the new `studio.vmoptions` set.

## Step 3: Attach debugger to Android Studio

You'll want to install a copy of IntelliJ to be able to attach to Android Studio. You may need to upgrade to the latest IntelliJ version.

1. Startup IntelliJ.

2. Load the `intellij-community` project.

3. If you see this notification warning, it means you may need to download a newer IntelliJ version:

   <img width="412" alt="image" src="https://github.com/rogerhu/studying-android-studio-internals/assets/326857/f2f7c358-b274-4198-9ee1-c8d40f5f8267">

4. Go to `Run` -> `Edit Configurations...`

5. Click on the `+` icon and `Add New Configuration`. Select `Remote JVM Debug`.

6. Specify the port number that you specified earlier (e.g. `5006`).

   <img width="1032" alt="image" src="https://github.com/rogerhu/studying-android-studio-internals/assets/326857/0798469c-451d-4370-922a-240b9c88ef34">

7. Select the Run Configuration and Attach to the port!

### Attaching to the Gradle Daemon

There are times when you will want to observe the interaction between how the Gradle daemon works.

1. Modify your `~/.gradle/gradle.properties` and add the following:

   ```gradle
   org.gradle.jvmargs=-agentlib:jdwp=transport=dt_socket,server=y,suspend=n,address=5006
   ```

2. `./gradlew --stop`

3. `./gradlew help` to restart the daemon

### Using the Android plugin from Google (optional)

You can also download the Android plugin maintained by Google, which is a separate fork and requires a few more advanced steps that includes downloading multiple repos and invoking Bazel to build a build of the protobuf dependencies. See https://github.com/rogerhu/android-studio-builder#step-by-step-guide-for-mac for instructions about how to get this plugin working in IntelliJ. 

<img width="1562" alt="image" src="https://github.com/rogerhu/studying-android-studio-internals/assets/326857/5f1923ee-0725-46b5-baf2-4d17182ac24a">

Attaching sources to the IntelliJ requires download the corresponding Android Studio. For instance, for Giraffe, we can download the .ZIP file:

https://github.com/JetBrains/intellij-community/releases/tag/idea%2F223.8836.35

And then use the `attach sources` to use the files.

### Testing different IDE versions

Another way of testing IDE versions is to launch them within Android Studio using the Intellij Gradle Plugin. It allows you to launch an IDE within Android Studio, which lets you
attach breakpoints.

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

