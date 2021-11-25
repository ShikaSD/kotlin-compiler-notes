# 🚧 (WIP) Kotlin compiler plugins 🚧

This repo contains some of my notes on Kotlin compiler plugin development, internal structure of compiler and possible extension points. Majority of these items are taken from [my droidcon 2020 talk](https://www.droidcon.com/2020/10/10/the-magic-of-compiler-extensions/), but I might be slowly extending the content to capture new developments (updates to plugin API or FIR).

If you feel like something is missing, feel free to open an issue with feature request. Contributions are welcome too!

Where to start:
- [[compiler-structure]]: Structured explanation of compiler phases
- 🚧[[plugin]]🚧: Anatomy of the compiler plugin
- 🚧[[tooling]]🚧: Tips for compiler plugin development

Notes in this repo are not intended to be a tutorial but something closer to documentation. However, there are plenty of articles (and talks) on starting with Kotlin compiler plugin development:
- [Writing Your Own Compiler plugin from KotlinConf 2018](https://www.youtube.com/watch?v=w-GMlaziIyo&ab_channel=JetBrainsTV) by Kevin Most (a bit outdated)
- [Writing your second compiler plugin series](https://blog.bnorm.dev/writing-your-second-compiler-plugin-part-1) by Brian Norman
- [Writing Kotlin Parcelize compiler plugin for iOS](https://medium.com/bumble-tech/writing-kotlin-parcelize-compiler-plugin-for-ios-678d81eed27e) by Arkadii Ivanov
- [Fixing serialization of Kotlin objects once and for all](https://medium.com/bumble-tech/fixing-serialization-of-kotlin-objects-once-and-for-all-95886fddba7a) by yours truly

You can also find examples of compiler plugins to start your experiments from (with varied degree of production readiness):
- First-party plugins by JetBrains:
  - [Kotlin serialization](https://github.com/JetBrains/kotlin/tree/master/plugins/kotlin-serialization/kotlin-serialization-compiler)
  - [Parcelize](https://github.com/JetBrains/kotlin/tree/master/plugins/parcelize)
  - [Kotless (compile time reflection)](https://github.com/JetBrains/kotless)
- Third-party:
  - [Jetpack Compose compiler plugin](https://cs.android.com/androidx/platform/frameworks/support/+/androidx-main:compose/compiler/compiler-hosted/src/main/java/androidx/compose/compiler/plugins/kotlin/)
  - [Redacted by Zac Sweers](https://github.com/ZacSweers/redacted-compiler-plugin)
  - [kotlin-power-assert by Brian Norman](https://github.com/bnorm/kotlin-power-assert)
  - [Multiplatform parcelize by Arkadii Ivanov](https://github.com/arkivanov/Essenty/tree/master/parcelable)
  - Your plugin?