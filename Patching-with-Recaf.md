---
title: Patching with Recaf
layout: default
---

## Figuring out where to patch

You’ll want to use grep to find declarations of a class. Let’s for instance say we want to find the `ProjectImportAction` class that gets used in Android Studio to sync the repository:

```bash
pushd "$HOME/Applications/MDX/Electric Eel/Android Studio Preview.app/Contents/plugins/gradle/lib"
grep ProjectImportAction *.jar
```

You may see:

```bash
Binary file gradle-tooling-extension-api.jar matches
```

If you want to be sure, install the `zipgrep` tool (`brew install zipgrep`)

```bash
zipgrep ProjectImportAction gradle-tooling-extension-api.jar
```

This result will show:

```bash
org/jetbrains/plugins/gradle/model/ProjectImportAction$1.class:Binary file (standard input) matches
org/jetbrains/plugins/gradle/model/ProjectImportAction$2.class:Binary file (standard input) matches
org/jetbrains/plugins/gradle/model/ProjectImportAction$3.class:Binary file (standard input) matches
org/jetbrains/plugins/gradle/model/ProjectImportAction$4.class:Binary file (standard input) matches
org/jetbrains/plugins/gradle/model/ProjectImportAction$5.class:Binary file (standard input) matches
org/jetbrains/plugins/gradle/model/ProjectImportAction$6.class:Binary file (standard input) matches
org/jetbrains/plugins/gradle/model/ProjectImportAction$7$1.class:Binary file (standard input) matches
org/jetbrains/plugins/gradle/model/ProjectImportAction$7.class:Binary file (standard input) matches
org/jetbrains/plugins/gradle/model/ProjectImportAction$8$1.class:Binary file (standard input) matches
org/jetbrains/plugins/gradle/model/ProjectImportAction$8$2.class:Binary file (standard input) matches
org/jetbrains/plugins/gradle/model/ProjectImportAction$8.class:Binary file (standard input) matches
org/jetbrains/plugins/gradle/model/ProjectImportAction$AllModels.class:Binary file (standard input) matches
org/jetbrains/plugins/gradle/model/ProjectImportAction$DefaultBuild$DefaultProjectModel.class:Binary file (standard input) matches
org/jetbrains/plugins/gradle/model/ProjectImportAction$DefaultBuild.class:Binary file (standard input) matches
org/jetbrains/plugins/gradle/model/ProjectImportAction$GradleBuildConsumer.class:Binary file (standard input) matches
org/jetbrains/plugins/gradle/model/ProjectImportAction$ModelConverter.class:Binary file (standard input) matches
org/jetbrains/plugins/gradle/model/ProjectImportAction$MyBuildController.class:Binary file (standard input) matches
org/jetbrains/plugins/gradle/model/ProjectImportAction$NoopConverter.class:Binary file (standard input) matches
org/jetbrains/plugins/gradle/model/ProjectImportAction.class:Binary file (standard input) matches
org/jetbrains/plugins/gradle/model/ProjectImportActionWithCustomSerializer$1.c
```

You can also install a GUI viewer to view the JAR file too:

```bash
brew install jadx
```

At the terminal, run `jadx-gui gradle-tooling-extension-api.jar` to confirm the classes are located there.

## Patching

Download the [[Recaf](https://github.com/Col-E/Recaf/releases)](https://github.com/Col-E/Recaf/releases) library. Make sure to download the JAR file that is labeled `-with-dependencies.jar`.

![image](https://github.com/rogerhu/studying-android-studio-internals/assets/326857/059b0cfd-47c5-4a1e-83f6-152c590b4e0a)

Then run Recaf:

```bash
java -jar ~/Downloads/recaf-2.21.13-J8-jar-with-dependencies.jar
```

Before loading any file, we are going to need to make a few tweaks in the `Config`.

Click on the `Decompile` option and choose the `Procyon` disassembler:

![image](https://github.com/rogerhu/studying-android-studio-internals/assets/326857/f35bf777-c6c0-4659-9a2e-1fcb1d79ee86)

Next, uncheck the `Generate missing classes` option. You will want to add libraries manually, especially for projects that depend on many other JAR files. With this option turns on, Recaf seems unable to resolve these missing classes.

![image](https://github.com/rogerhu/studying-android-studio-internals/assets/326857/7eb33359-192d-423a-bed1-dd1f00d809d3)

Exit the Config menu by clicking on the Red icon:

![image](https://github.com/rogerhu/studying-android-studio-internals/assets/326857/8e558815-e4fc-4566-8b03-1881a8e2da8b)

Use the `File` → `Load` to load the JAR file (e.g. `gradle-tooling-extension-api.jar`).

Recaf depends on having all the classes necessary to recompile the JAR file. If you edit a class and hit `Cmd-S` to save it, you will likely see errors at the bottom. If there are `Cannot find symbol` errors and red lines on the import statements for the class file, it means that you may will need to use the `Add library` feature:

![image](https://github.com/rogerhu/studying-android-studio-internals/assets/326857/dc7c78aa-b474-42b6-9dc4-7e8af18aa855)

To find any class references, you will likely need to use the same techniques described in the [[earlier section](https://www.notion.so/Patching-Java-byte-code-911a09142104417b90f4ae55cdc3092a?pvs=21)](https://www.notion.so/Patching-Java-byte-code-911a09142104417b90f4ae55cdc3092a?pvs=21) to find them. It basically is a combination of `grep` and using `zipgrep`/ `jadx-gui` on individual files to confirm these are the right ones.

Once you add a library, you will see the top-left corner turns into a drop-down. Only the JAR labeled `Primary` can be modified. For instance, here is an example of the dependent libraries for the `gradle-tooling-extension-api.jar` that were needed to update the `ProjectImportAction` class. There could be other JAR files depending on which class file you edit.

![image](https://github.com/rogerhu/studying-android-studio-internals/assets/326857/745e9859-c7e6-481a-b7c7-fb00c2ef3f1b)

You can use the `Export workspace` option to save this configuration. The contents will be saved as JSON ([[example](https://square.slack.com/archives/C044SA66QF6/p1665015986611339?thread_ts=1664924516.076049&cid=C044SA66QF6)](https://square.slack.com/archives/C044SA66QF6/p1665015986611339?thread_ts=1664924516.076049&cid=C044SA66QF6)) and can be reloaded again by Recaf.

Once you’ve identified the changes you want to make, you may need to deal with casting issues by the decompiler:

![image](https://github.com/rogerhu/studying-android-studio-internals/assets/326857/1862052f-a841-4a12-9729-fb1182b1bd48)

Most of the issues can be solved by deleting the casting issues in question. The decompiler is not perfect, so using Recaf’s assembler mostly involves removing these errors. For instance, this line can be changed from:

```bash
addFetchedModelActions.addAll((Collection<? extends List<Runnable>>)controller.run((Collection<? extends BuildAction<?>>)buildActions));
```

To:

```bash
addFetchedModelActions.addAll(controller.run(buildActions));
```

The one exception is the ‘cannot find symbol class `AllModels` class, which needs to be referenced as `ProjectImportAction.AllModels` because it’s actually compiled in a different file.

### Adding other Java classes

Recaf doesn’t have a way to add Java classes, but you can do it pretty easily by creating the Java class and compiling it. We can simply create `[SimpleThreadFactor](http://SimpleThreadFactory.java)y.java` in a local editor. Note that we set the package to `org.jetbrains.plugins.gradle.model` so we will need to take care of this case later.

```bash
package org.jetbrains.plugins.gradle.model;

import java.util.concurrent.ThreadFactory;

public final class SimpleThreadFactory implements ThreadFactory {
    public Thread newThread(Runnable r) {
       return new Thread(r, "idea-tooling-model-converter");
   }
 }
```

Then you can create the `.class` file.

```bash
javac SimpleThreadFactory.java
```

The next step is to add it to the JAR we want to patch. Let’s make a backup just in case:

```bash
cp gradle-tooling-extension-api.jar gradle-tooling-extension-api.jar.orig
```

You can then add this `SimpleThreadFactory.class` by relocating it in the right directory:

```bash
mkdir -p org/jetbrains/plugins/gradle/model
mv SimpleThreadFactory.class org/jetbrains/plugins/gradle/model
zip gradle-tooling-extension-api.jar org/jetbrains/plugins/gradle/model/SimpleThreadFactory.class
```

Then go ahead and do your patching work with this new added class!