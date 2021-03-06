# gradle-apt-plugin

The goal of this plugin is to eventually no longer be needed, being superseded by built-in features.

It originally did a few things to make it easier/safer to use Java annotation processors in a Gradle build.
Those things are now available natively in Gradle, so what's this plugin about?

If you use older versions of Gradle, you can still benefit from those features:
* it ensures the presence of configurations for your compile-time only dependencies (annotations, generally) and annotation processors, consistently across all supported Gradle versions;
* automatically configures the corresponding `JavaCompile` and `GroovyCompile` tasks to make use of these configurations, when the `java` or `groovy` plugin is applied.

With recent versions of Gradle, this plugin will actually only:
* configure `JavaCompile` and `GroovyCompile` tasks' `options.annotationProcessorGeneratedSourcesDirectory` with a _sane_ default value so you can see the generated sources in your IDE and for debugging, and avoid shipping them in your JARs ([see Gradle issue](https://github.com/gradle/gradle/issues/4956));
* add some DSL to configure annotation processors; it is however recommended to directly configure the tasks' `options.compilerArgs`.

Quite ironically, what you'll probably find the most useful here is the part that's only provided as a "best effort",
namely the `net.ltgt.apt-idea` or `net.ltgt.apt-eclipse` plugins that will automatically configures IntelliJ IDEA and Eclipse respectively.

If you're interested in better IDE support, please vote for those issues to eventually have built-in support:
 * [in Gradle](https://github.com/gradle/gradle/issues/2300) for the `idea` and `eclipse` plugins
 * [in Eclipse Buildship](https://github.com/eclipse/buildship/issues/329)
 * in IntelliJ IDEA: [for annotation processing in the IDE](https://youtrack.jetbrains.com/issue/IDEA-187868),
   and/or simply [`options.annotationProcessorGeneratedSourcesDirectory` (e.g. if delegating build/run actions to Gradle)](https://youtrack.jetbrains.com/issue/IDEA-182577)

## Do without the plugin

<details>
<summary>This only applies if you are using Gradle ≥ 4.3</summary>

### Do without `net.ltgt.apt`

DSL aside, the `net.ltgt.apt` plugin, is equivalent to the following snippet
(unless otherwise noted, all snippets here assume Gradle ≥ 4.9, and only deal with Java projects, not Groovy ones):

<details open>
<summary>Groovy</summary>

```gradle
// Workaround for https://github.com/gradle/gradle/issues/4956
sourceSets.configureEach { sourceSet ->
  tasks.named(sourceSet.compileJavaTaskName).configure {
    options.annotationProcessorGeneratedSourcesDirectory = file("$buildDir/generated/source/apt/${sourceSet.name})
  }
}
```

</details>
<details>
<summary>Kotlin</summary>

```kotlin
// Workaround for https://github.com/gradle/gradle/issues/4956
sourceSets.configureEach {
    tasks.named<JavaCompile>(compileJavaTaskName) {
        options.annotationProcessorGeneratedSourcesDirectory = file("$buildDir/generated/source/apt/${this@configureEach.name}")
    }
}
```

</details>

With Gradle ≤ 4.6, you'll also need to create the `<sourceSet>AnnotationProcessor` configurations and configure the tasks' `options.annotationProcessorPath`
(the following snippet will disable falling back to the compilation classpath to resolve annotation processors though):

```gradle
sourceSets.all {
  def annotationProcessorConfiguration = configurations.create(name == "main" ? "annotationProcessor" : "${name}AnnotationProcessor")
  tasks[compileJavaTaskName].options.annotationProcessorPath = annotationProcessorConfiguration
}
```

Alternatively, you can selectively create configurations and configure tasks as needed:

```gradle
configurations {
  annotationProcessor
}
compileJava {
  options.annotationProcessorPath = configurations.annotationProcessor
}
```

### Do without `net.ltgt.apt-idea`

If you're delegating IDEA build/run actions to Gradle, and/or somehow don't want or need IntelliJ IDEA to do the annotation processing,
the `net.ltgt.apt-idea` plugin is roughly equivalent to the following snippet
(this assumes a somewhat recent version of IntelliJ IDEA that automatically unexcludes source folders inside excluded folders):

<details open>
<summary>Groovy</summary>

```gradle
// Workaround for https://youtrack.jetbrains.com/issue/IDEA-182577
idea {
  module {
    sourceDirs += compileJava.options.annotationProcessorGeneratedSourcesDirectory
    generatedSourceDirs += compileJava.options.annotationProcessorGeneratedSourcesDirectory
    testSourceDirs += compileTestJava.options.annotationProcessorGeneratedSourcesDirectory
    generatedSourceDirs += compileTestJava.options.annotationProcessorGeneratedSourcesDirectory
  }
}
```

</details>
<details>
<summary>Kotlin</summary>

```kotlin
// Workaround for https://youtrack.jetbrains.com/issue/IDEA-182577
idea {
    module {
        tasks.getByName<JavaCompile>("compileJava").options.annotationProcessorGeneratedSourcesDirectory?.also {
            // For some reason, modifying the existing collections doesn't work. We need to copy the values and then assign it back.
            sourceDirs = sourceDirs + it
            generatedSourceDirs = generatedSourceDirs + it
        }
        tasks.getByName<JavaCompile>("compileTestJava").options.annotationProcessorGeneratedSourcesDirectory?.also {
            // For some reason, modifying the existing collections doesn't work. We need to copy the values and then assign it back.
            testSourceDirs = testSourceDirs + it
            generatedSourceDirs = generatedSourceDirs + it
        }
    }
}
```

</details>

If you want IntelliJ IDEA to process annotations, you'll have to do some manual configuration to enable annotation processing,
and add the annotation processors to the project's compilation classpath:

<details open>
<summary>Groovy</summary>

```gradle
// Workaround for https://youtrack.jetbrains.com/issue/IDEA-187868
idea {
  module {
    scopes.PROVIDED.plus += configurations.annotationProcessor
    scopes.TEST.plus += configurations.testAnnotationProcessor
  }
}
```

</details>
<details>
<summary>Kotlin</summary>

```kotlin
// Workaround for https://youtrack.jetbrains.com/issue/IDEA-187868
idea {
    module {
        scopes["PROVIDED"]["plus"].add(configurations.annotationProcessor)
        scopes["TEST"]["plus"].add(configurations.testAnnotationProcessor)
    }
}
```

</details>

If you want to apply this configuration automatically for every project,
you can do this with an init script similar to the following:

```gradle
import org.gradle.util.GradleVersion;

// If running from IntelliJ IDEA (such as when importing the project)
if (Boolean.getBoolean("idea.active")) {
  def HAS_PROCESSOR_GENERATED_SOURCES_DIR = GradleVersion.current() >= GradleVersion.version("4.3")

  allprojects { project ->
    project.apply plugin: 'idea'

    if (HAS_PROCESSOR_GENERATED_SOURCES_DIR) {
      project.plugins.withType(JavaPlugin) {
        project.afterEvaluate {
          project.idea.module {
            def mainGeneratedSources = project.tasks["compileJava"].options.annotationProcessorGeneratedSourcesDirectory
            if (mainGeneratedSources) {
              sourceDirs += mainGeneratedSources
              generatedSourceDirs += mainGeneratedSources
            }
            def testGeneratedSources = project.tasks["compileTestJava"].options.annotationProcessorGeneratedSourcesDirectory
            if (testGeneratedSources) {
              testSourceDirs += testGeneratedSources
              generatedSourceDirs += testGeneratedSources
            }

            // Uncomment if you want to do annotation processing in IntelliJ IDEA:
            // def annotationProcessorConfiguration = configurations.findByName("annotationProcessor")
            // if (annotationProcessorConfiguration) {
            //   scopes.PROVIDED.plus += annotationProcessorConfiguration
            // }
            // def testAnnotationProcessorConfiguration = configurations.findByName("testAnnotationProcessor")
            // if (testAnnotationProcessorConfiguration) {
            //   scopes.TEST.plus += testAnnotationProcessorConfiguration
            // }
          }
        }
      }
    } else {
      // fallback to automatically applying net.ltgt.apt-eclipse whenever net.ltgt.apt is used
      project.plugins.withId("net.ltgt.apt") {
        try {
          project.apply plugin: "net.ltgt.apt-idea"
          // Comment if you want to do annotation processing in IntelliJ IDEA:
          project.plugins.withType(JavaPlugin) {
            project.afterEvaluate {
              project.idea.module.apt.addAptDependencies = false
            }
          }
        } catch (UnknownPluginException) {
          // ignore, in case an older version of net.ltgt.apt is being used
          // that doesn't come with net.ltgt.apt-idea.
        }
      }
    }
  }
}
```

### Do without `net.ltgt.apt-eclipse`

There's no easy workaround at this point, sorry :man_shrugging:

</details>

## Using the plugin

The plugin is published to the Plugin Portal; see instructions there: https://plugins.gradle.org/plugin/net.ltgt.apt

## Configurations

For each `SourceSet`, three configurations are available:

* `<sourceSet>CompileOnly` (Gradle ≥ 2.12 already provides those configurations; note that this plugin doesn't provide a `<sourceSet>CompileClasspath` like Gradle ≥ 2.12, but instead make it directly extend `<sourceSet>Compile`)
* `<sourceSet>AnnotationProcessor`, since version 0.14 (Gradle 4.6 already provides those configurations)
* `<sourceSet>Apt`: those are provided for backwards compatibility with versions of this plugin up to 0.13, and are deprecated since version 0.14. The `<sourceSet>AnnotationProcessor` configurations extend the respective `<sourceSet>Apt` configurations to provide that compatibility.

As a result, the following configurations are available for any Java project:

* `compileOnly`, extends `compile`
* `annotationProcessor` (and `apt`)
* `testCompileOnly`, extends `testCompile`
* `testAnnotationProcessor` (and `testApt`)

The `*Only` configurations are used to specify compile-time only dependencies such as annotations that will be processed by annotation processors. Annotation processors themselves are to be added to the `annotationProcessor` and `testAnnotationProcessor` configurations (or the `apt` and `testApt` configurations in version 0.13 and earlier).

The `*Only` configurations are part of the `classpath` of the `JavaCompile` and `GroovyCompile` tasks, whereas the `apt` and `testApt` configurations are turned into `-processorpath` compiler arguments.
Note that up until version 0.7, if those configurations were empty, an empty processor path (`-processorpath :`) would be passed to `javac`; this was a breaking change compared to the normal behavior of Gradle, as it meant annotation processors wouldn't be looked up in the tasks' `classpath`.
Starting with version 0.8, no `-processorpath` will be passed if the `<sourceSet>Apt` configuration is empty; this is to follow a proposal to add first-class support for annotation processing to Gradle proper, that [has been added in Gradle 4.6](https://github.com/gradle/gradle/pull/3786).

Finally, note that those configurations don't extend each others: `testCompileOnly` doesn't extend `compileOnly`, and `testAnnotationProcessor` doesn't extend `annotationProcessor`; those configurations are only use for their respective `JavaCompile` and `GroovyCompile` tasks.

### Example usage

After applying the plugin following the above instructions, those added configurations can be used when declaring dependencies:

```gradle
dependencies {
  compile("com.google.dagger:dagger:2.18")
  annotationProcessor("com.google.dagger:dagger-compiler:2.18")

  // auto-factory contains both annotations and their processor, neither is needed at runtime
  compileOnly("com.google.auto.factory:auto-factory:1.0-beta6")
  annotationProcessor("com.google.auto.factory:auto-factory:1.0-beta6")

  compileOnly("org.immutables:value-annotations:2.7.1")
  annotationProcessor("org.immutables:value:2.7.1")
}
```

## Groovy support

Starting with version 0.6, the plugin also configures `GroovyCompile` tasks added when the `groovy` plugin is applied.
It does not however configure annotation processing for Groovy sources, only for Java sources used in joint compilation.
To process annotations on Groovy sources, you'll have to configure your `GroovyCompile` tasks; e.g.

<details open>
<summary>Groovy</summary>

```gradle
compileGroovy {
  groovyOptions.javaAnnotationProcessing = true
}
```

</details>
<details>
<summary>Kotlin</summary>

```kotlin
tasks.named<GroovyCompile>("compileGroovy") {
  groovyOptions.isJavaAnnotationProcessing = true
}
```

</details>

## Build cache

Compilation tasks are still [cacheable](https://docs.gradle.org/current/userguide/build_cache.html) with a few caveats:
 * Only one _language_ can be used per source set (i.e. either `src/main/java` or `src/main/groovy` but not both), unless Groovy joint compilation is used (putting Java files in `src/main/groovy`), or tasks are configured to use distinct generated sources destination directories.
 * Groovy compilation tasks are only fully cacheable starting with Gradle 4.3.
   In previous versions, the tasks won't be relocatable and will only be cacheable if files in the annotation processor path do not change (e.g. when using a project dependency and that project is rebuilt, even if the classes come from the build cache).
   This due to a bug/limitation in Gradle preventing the plugin to rely on `options.annotationProcessorPath`, and having no mean to tell Gradle to use classpath normalization; this was fixed in Gradle 4.3.

## Gradle Kotlin DSL

Starting with version 0.15, the plugin provides Kotlin extensions to make configuration easier when using the Gradle Kotlin DSL.
The easiest is to `import net.ltgt.gradle.apt.*` at the top of your `*.gradle.kts` file.
Most APIs are the same as in Groovy, see below for differences.

## Usage with IDEs

IDE configuration is provided on a best-effort basis.

### Eclipse

Starting with version 0.11, applying the `net.ltgt.apt-eclipse` plugin will auto-configure the generated files to enable annotation processing in Eclipse.
In prior versions (until 0.10), that configuration would automatically happen whenever both the `net.ltgt.apt` and `eclipse` were applied (the new `net.ltgt.apt-eclipse` plugin will also automatically apply the `net.ltgt.apt` and `eclipse` plugins).

From version 0.11 onwards, Eclipse annotation processing can be configured through a DSL, as an extension to the Eclipse JDT DSL (presented here with the default values):

<details open>
<summary>Groovy</summary>

```gradle
eclipse {
  jdt {
    apt {
      // whether annotation processing is enabled in Eclipse
      aptEnabled = compileJava.aptOptions.annotationProcessing
      // where Eclipse will output the generated sources; value is interpreted as per project.file()
      genSrcDir = file('.apt_generated')
      // whether annotation processing is enabled in the editor
      reconcileEnabled = true
      // a map of annotation processor options; a null value will pass the argument as -Akey rather than -Akey=value
      processorOptions = compileJava.aptOptions.processorArgs

      file {
        whenMerged { jdtApt ->
          // you can tinker with the JdtApt here
        }

        withProperties { properties ->
          // you can tinker with the Properties here
        }
      }
    }
  }

  factorypath {
    plusConfigurations = [ configurations.annotationProcessor, configurations.testAnnotationProcessor ]
    minusConfigurations = []

    file {
      whenMerged { factorypath ->
        // you can tinker with the Factorypath here
      }

      withXml { node ->
        // you can tinker with the Node here
      }
    }
  }
}
```

</details>
<details>
<summary>Kotlin</summary>

```kotlin
eclipse {
  jdt {
    apt {
      // whether annotation processing is enabled in Eclipse
      isAptEnabled = tasks.getByName<JavaCompile>("compileJava").aptOptions.annotationProcessing
      // where Eclipse will output the generated sources; value is interpreted as per project.file()
      genSrcDir = file(".apt_generated")
      // whether annotation processing is enabled in the editor
      isReconcileEnabled = true
      // a map of annotation processor options; a null value will pass the argument as -Akey rather than -Akey=value
      processorOptions = tasks.getByName<JavaCompile>("compileJava").aptOptions.processorArgs

      file {
        whenMerged {
          // you can tinker with the JdtApt here, you'll need to cast 'this' to JdtApt
        }

        withProperties {
          // you can tinker with the Properties here
        }
      }
    }
  }

  factorypath {
    plusConfigurations = [ configurations.annotationProcessor, configurations.testAnnotationProcessor ]
    minusConfigurations = []

    file {
      whenMerged {
        // you can tinker with the Factorypath here, you'll need to cast 'this' to Factorypath
      }

      withXml {
        // you can tinker with the Node here
      }
    }
  }
}
```

</details>

When using Buildship, you'll have to manually run the `eclipseJdtApt` and `eclipseFactorypath` tasks to generate the Eclipse configuration files, then either run the `eclipseJdt` task or manually enable annotation processing: in the project properties → Java Compiler → Annotation Processing, check `Enable Annotation Processing`. Note that while all those tasks are depended on by the `eclipse` task, that one is incompatible with Buildship, so you have to explicitly run the two or three aforementioned tasks and _not_ run the `eclipse` task.

Note that Eclipse does not distinguish main and test sources, and will process all of them using the same factory path and processor options, and the same generated source directory.

In any case, the `net.ltgt.apt-eclipse` plugin (or simply `eclipse` plugin up until version 0.10) has to be applied to the project.

This can be configured system-wide for all projects using the `net.ltgt.apt` plugin by using an init script similar to the following:
```gradle
allprojects { project ->
  project.plugins.withId("net.ltgt.apt") {
    // automatically apply net.ltgt.apt-eclipse whenever net.ltgt.apt is used
    try {
      project.apply plugin: "net.ltgt.apt-eclipse"
    } catch (UnknownPluginException) {
      // ignore, in case an older version of net.ltgt.apt is being used
      // that doesn't come with net.ltgt.apt-eclipse.
    }
  }
}
```

### IntelliJ IDEA

Starting with version 0.11, applying the `net.ltgt.apt-idea` plugin will auto-configure the generated files to enable annotation processing in IntelliJ IDEA.
In prior versions (until 0.10), that configuration would automatically happen whenever both the `net.ltgt.apt` and `idea` were applied (the new `net.ltgt.apt-idea` plugin will also automatically apply the `net.ltgt.apt` and `idea` plugins).

When using the Gradle integration in IntelliJ IDEA (rather than the `idea` task), it is recommended to delegate the IDE build actions to Gradle itself starting with IDEA 2016.3: https://www.jetbrains.com/help/idea/gradle.html#delegate_build_gradle
Otherwise, you'll have to manually enable annotation processing: in Settings… → Build, Execution, Deployment → Compiler → Annotation Processors, check `Enable annotation processing` and `Obtain processors from project classpath` (you'll have to make sure `idea.module.apt.addAptDependencies` is enabled, starting with version 0.12). To mimic the Gradle behavior and generated files behavior, you can configure the production and test sources directories to `build/generated/source/apt/main` and `build/generated/source/apt/test` respectively and choose to `Store generated sources relative to:` `Module content root`.

Note that starting with IntelliJ IDEA 2016.1, and unless you delegate build actions to Gradle, you'll have to uncheck `Create separate module per source set` when importing the project.

From version 0.12 onwards, IntelliJ IDEA annotation processing can be configured through a DSL, as an extension to the IDEA DSL (presented here with the default values):
```gradle
idea {
  project {
    // experimental: whether annotation processing will be configured in the IDE; only actually used with the 'idea' task.
    configureAnnotationProcessing = true
  }
  module {
    apt {
      // whether generated sources dirs are added as generated sources root
      addGeneratedSourcesDirs = true
      // whether the annotationProcessor/apt and testAnnotationProcessor/testApt dependencies are added as module dependencies
      addAptDependencies = true

      // The following are mostly internal details; you shouldn't ever need to configure them.
      // whether the compileOnly and testCompileOnly dependencies are added as module dependencies
      addCompileOnlyDependencies = false // defaults to true in Gradle < 2.12
      // the dependency scope used for apt and/or compileOnly dependencies (when enabled above)
      mainDependenciesScope = "PROVIDED" // defaults to "COMPILE" in Gradle < 3.4, or when using the Gradle integration in IntelliJ IDEA
    }
  }
}
```

If you always delegate build actions to Gradle, you can thus disable `idea.module.apt.addAptDependencies` system-wide (there's unfortunately no way to detect this when importing the project in IDEA, so the plugin cannot configure itself automatically), by putting the following in an [init script](https://docs.gradle.org/current/userguide/init_scripts.html), e.g. `~/.gradle/init.d/apt-idea.gradle`:
```gradle
allprojects { project ->
  project.plugins.withType(JavaPlugin) {
    project.plugins.withId("net.ltgt.apt-idea") {
      project.afterEvaluate {
        project.idea.module.apt.addAptDependencies = false
      }
    }
  }
}
```

In any case, the `net.ltgt.apt-idea` plugin (or simply `idea` plugin up until version 0.10) has to be applied to the project.

This can be configured system-wide for all projects using the `net.ltgt.apt` plugin by using an init script similar to the following:
```gradle
allprojects { project ->
  project.plugins.withId("net.ltgt.apt") {
    try {
      // automatically apply net.ltgt.apt-idea whenever net.ltgt.apt is used
      project.apply plugin: "net.ltgt.apt-idea"
      // disable addAptDependencies (if you delegate build actions to Gradle)
      project.plugins.withType(JavaPlugin) {
        project.afterEvaluate {
          project.idea.module.apt.addAptDependencies = false
        }
      }
    } catch (UnknownPluginException) {
      // ignore, in case an older version of net.ltgt.apt is being used
      // that doesn't come with net.ltgt.apt-idea.
    }
  }
}
```

## Configuration

Starting with version 0.8, the plugin makes many things configurable by enhancing source sets and tasks.
Some of those enhancements have been added to Gradle proper, sometimes with different names. Starting with version 0.14, you're encouraged to use the built-in Gradle properties, and the equivalent ones added by this plugin are deprecated (and will emit deprecation messages to the console.)

Each source set has a few properties:

* `compileOnlyConfigurationName` (read-only `String`) returning the `<sourceSet>CompileOnly` configuration name (Gradle ≥ 2.12 already provides that property natively, this plugin contributes it for earlier Gradle versions)
* `annotationProcessorConfigurationName` (read-only `String`) returning the `<sourceSet>AnnotationProcessor>` configuration name, starting with version 0.14 (Gradle ≥ 4.6 already provides that property natively, this plugin contributes it for earlier Gradle versions)
* `aptConfigurationName` (read-only `String`) returning the `<sourceSet>Apt` configuration name, deprecated in version 0.14, replaced with `annotationProcessorConfigurationName`. There's no Kotlin extension for this property.
* `annotationProcessorPath`, a `FileCollection` defaulting to the `<sourceSet>AnnotationProcessor` configuration, starting with version 0.14
* `processorpath`, a `FileCollection` defaulting to the `<sourceSet>Apt` configuration, deprecated in version 0.14, replaced with `annotationProcessorPath`. There's no Kotlin extension for this property.

Each source set `output` gains a `generatedSourcesDir` property, a `File` defaulting to `${project.buildDir}/generated/source/apt/${sourceSet.name}`.

Each `JavaCompile` and `GroovyCompile` task gains a couple properties:

* `generatedSourcesDestinationDir`, corresponding to the `-s` compiler argument, i.e. whether (if set) and where to write sources files generated by annotation processors. This property is deprecated starting with version 0.14 when using Gradle ≥ 4.3, please use `options.annotationProcessorGeneratedSourcesDirectory` instead. There's no Kotlin extension for this property.
* `aptOptions` (read-only), itself with 4 properties:
  * `annotationProcessing`, a `boolean` setting whether annotation processing is enabled or not; this maps to the `-proc:none` compiler argument, and defaults to `true` (meaning that argument is not passed in, and annotation processing is enabled)
  * `processorpath`, a `FileCollection` corresponding to the `-processorpath` compiler argument; this property is deprecated starting with version 0.14 when using Gradle ≥ 3.4, please use `options.annotationProcessorPath` instead
  * `processors`, a list of annotation processor class names, mapping to the `-processor` compiler argument
  * `processorArgs`, a map of annotation processor options, each entry mapping to a `-Akey=value` compiler argument

For each source set, the corresponding `JavaCompile` and `GroovyCompile` tasks are configured such that:

* `generatedSourcesDestinationDir` maps to the source set's `output.generatedSourcesDir`
* `aptOptions.processorpath` maps to the source set's `annotationProcessorPath`
