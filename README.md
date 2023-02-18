# Intro

Curious about the IntelliJ IDE works? Here's a guide to help you better understand the IDE internals. You can attach breakpoints to sift through the
complex interaction between code written by Google, JetBrains, and Gradle.

Thanks to [Pablo Baxter](https://github.com/pablobaxter) for figuring this part out.

Follow the Quickstart below to get started. Here are other topics that will be explored:

* History
* JetBrains Runtime (https://github.com/JetBrains/JetBrainsRuntime)
* Sync Process
* How Unit tests are configured
* Writing IntelliJ plugins
* Kotlin frontend compiler

# Quickstart

## Step 1: Grab the IntelliJ source code

1. Clone the IntelliJ repository.

```bash
git clone https://github.com/JetBrains/intellij-community
pushd intellij-community
```

2. Match the version used in Android Studio (see https://plugins.jetbrains.com/docs/intellij/android-studio-releases-list.html):

For instance, to checkout the Electric Eel version, you would use this command:

```bash
git checkout idea/221.6008.13
```

3. To get the Android plugin, clone from this repo.

```
git clone git://git.jetbrains.org/idea/android.git android
```

NOTE: this Android plugin lags behind Google's main repository stored at https://android.googlesource.com/platform/tools/adt/idea/.
The work is a little bit involved to get this work. Ssee https://github.com/rogerhu/android-studio-builder/tree/rogerh/darwin-ee for
instructions about how to get this plugin working in IntelliJ.

## Step 2: Setup Android Studio

Because Android Studio is a Java process, you can setup config options to be to attach to it.

1. Start Android Studio and go to `Help` -> `Edit Custom VM Options...`.

2. Add the following to the end of the `studio.vmoptions` file:

```java
-agentlib:jdwp=transport=dt_socket,server=y,suspend=n,address=5006
```

* Note the 5006 address can be any value. Because Gradle uses 5005 by default, we choose 5006 to avoid conflicts.
* If you want to wait for Gradle to start, set `suspend=y`. This may be useful in case you want to put breakpoints
  against the Gradle source code to see what's happening.

3. Restart Android Studio!

## Step 3: Attach debugger to Android Studio

You'll want to install a copy of IntelliJ to be able to attach to Android Studio.

1. Go to `Run` -> `Edit Configurations...`

2. Click on the `+` icon and `Add New Configuration`. Select `Remote JVM Debug`.

3. Specify the port number that you specified earlier (e.g. `5006`).

4. Select the Run Configuration and Attach to the port!

### Attaching to the Gradle Daemon

There are times when you will want to observe the interaction between how the Gradle daemon works.

1. Modify your `~/.gradle/gradle.properties` and add the following:

```gradle
org.gradle.jvmargs=-agentlib:jdwp=transport=dt_socket,server=y,suspend=n,address=5006
```

2. `./gradlew --stop`

3. `./gradlew help` to restart the daemon
