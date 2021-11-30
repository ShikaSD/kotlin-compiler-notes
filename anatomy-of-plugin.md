# Anatomy of a compiler plugin

Compiler plugin is usually contained in an additional .jar file which is loaded by compiler through [`ServiceLoader`](https://docs.oracle.com/javase/7/docs/api/java/util/ServiceLoader.html). 

## Attaching plugins

Kotlin compiler will automatically load .jar files provided in `-Xplugin=` parameter in command line. Gradle provides a few convenience methods to manage plugins via Maven coordinates:

```kotlin
// Gradle dependency, note custom configuration
dependencies {
  kotlinCompilerPluginClasspath("my.plugin:from.maven:1.0.0")
}

// Alternative, modifying CLI args
tasks.withType<KotlinCompile> {
  kotlinOptions {
    freeCompilerArgs += listOf("-Xplugin=path/to/your/jar")
  }
}
```

Compiler plugins frequently require additional configuration available with Gradle plugins, see the section discussing them below.

## Plugin entrypoints

The compiler tries to load two entrypoints through `ServiceLoader`: `CommandLineProcessor` and `ComponentRegistrar`.

`CommandLineProcessor` hooks into processing of command line arguments. It requires providing a `pluginId` specific for the plugin and then a method to handle options (provided as `String`). Processed options are then expected to be included into `CompilerConfiguration`, which is later supplied to `ComponentRegistrar`. 

Specified options can be provided to the plugin with the following template: `-P plugin:${pluginId}:${option}=${value}`.
```kotlin
class ExampleCommandLineProcessor : CommandLineProcessor {
  override val pluginId: String = "dev.example"
  override val pluginOptions: Collection<AbstractCliOption> =
      listOf(
          CliOption(
              name = "enabled",
              valueDescription = "<true|false>", // used for help command
              description = "Whether plugin is enabled",
              required = false
          )
      )

  override fun processOption(option: AbstractCliOption, value: String, configuration: CompilerConfiguration) {
      when (option.optionName) {
          "enabled" -> configuration.put(KEY_ENABLED, value.toBoolean())
      }
  }

  companion object {
      val KEY_ENABLED = CompilerConfigurationKey<Boolean>("example.plugin.enabled")
  }
}

// usage: -P plugin:dev.example:example.plugin.enabled=true
```

`ComponentRegistrar` is the entrypoint to the rest of the compiler plugin. Given the `CompilerConfiguration`, and `Project` instance, you can register all extensions along the compilation process. Most of registation calls come in the form `${Name}Extension.registerExtension(project, instance)`, with each extension that interests you should be called separately.

Here's an example `ComponentRegistrar` registering two generator extensions for old JVM backend and new IR one:
```kotlin
class ExampleComponentRegistrar(): ComponentRegistrar {
  override fun registerProjectComponents(project: MockProject, configuration: CompilerConfiguration) {
      if (configuration[KEY_ENABLED] == false) {
          return
      }

      ExpressionCodegenExtension.registerExtension(
          project,
          ExampleJvmGeneration()
      )

      IrGenerationExtension.registerExtension(
          project,
          ExampleIrGeneration()
      )
  }
}
```
  
Some elements (e.g. additional `CallChecker`) are contributed through compiler DI system instead:
```kotlin
StorageComponentContainerContributor.registerExtension(
  project,
  object : StorageComponentContainerContributor {
    override fun registerModuleComponents(
      container: StorageComponentContainer,
      platform: TargetPlatform,
      moduleDescriptor: ModuleDescriptor
    ) {
      container.useInstance(ExampleCallChecker())
      container.useInstance(ExampleDeclarationChecker())
    }
  }
)
```

To link `CommandLineProcessor` and `ComponentRegistrar` with `ServiceLoader`, compiler needs additional configuration files. The configuration can be done automatically by `AutoService`, annotating abovementioned classes with `@AutoService(CommandLineProcessor::class)` and `@AutoService(ComponentRegistrar::class)` respectively.

It is also possible to provide configuration files manually, they are expected to be in `resources/META-INF/services` folder of a compiler plugin module. The files should be named by fully qualified name of the interface and contain fully qualified name of the implementation they are linking to. See example [here](https://github.com/ShikaSD/kotlin-object-serialization-fix/tree/master/kotlin-plugin/src/main/resources/META-INF/services).

## Gradle plugins

Some compiler plugins require some configuration by the end user. After creating the CLI parameters above, it is often useful to wrap them with Gradle integration. Kotlin provides compiler plugin support API in `org.jetbrains.kotlin:kotlin-gradle-plugin-api` dependency.

`KotlinCompilerPluginSupportPlugin` extends Gradle plugin API to allow configuring some common parts of the compiler plugin. It expects you override several methods to customize this plugin:
```kotlin
/**
 * The same id as in CommandLineProcessor
 */
override fun getCompilerPluginId(): String = "dev.example"

/**
 * List of options for CommandLineProcessor. 
 * They can be resolved from the project or customized in extension.
 */
override fun applyToCompilation(kotlinCompilation: KotlinCompilation<*>): Provider<List<SubpluginOption>> {
  val project = kotlinCompilation.target.project

  return project.objects.listProperty(SubpluginOption::class.java).apply {
    add(
      SubpluginOption(
        key = "enabled",
        value = true
      )
    )
  }
}

/**
 * Maven coordinates of the compiler plugin jar artifact
 */
override fun getPluginArtifact(): SubpluginArtifact =
  SubpluginArtifact(
    groupId = "dev.example",
    artifactId = "example-compiler-plugin",
    version = "1.0.0"
  )

/**
 * Whether compiler is applicable to some sourcesets. 
 * Some could only be available for limited number of platforms, 
 * e.g. JVM or Native, which you can check here.
 */ 
override fun isApplicable(kotlinCompilation: KotlinCompilation<*>): Boolean = true
```

Note that `getPluginArtifact` allows to ship the compiler plugin as a separate JAR. It could be quite useful, but not necessarily required, as the maven artifact coordinates can refer to the same artifact as the Gradle plugin uses. In cases when separate artifact is used, there's no need to add it as a dependency separately, as `KotlinCompilerPluginSupportPlugin` does that by itself.

