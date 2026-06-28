<!-- Copyright 2000-2025 JetBrains s.r.o. and contributors. Use of this source code is governed by the Apache 2.0 license. -->

# Starter Core

The core of the Starter framework for writing integration tests for IntelliJ Platform-based IDEs.

## Basics

Each test runs the IDE as a separate process, isolating the test runtime from the IDE runtime.
The IDE is driven by commands, which can be executed in two ways:

1. Write a scenario to a file (a list of plain-text strings in a special format) and pass it to the IDE.
2. Trigger a single command remotely with a JMX call (`Driver` implementation).

In both cases, commands are executed via the 
[`performanceTestingPlugin`](https://github.com/JetBrains/intellij-community/tree/master/plugins/performanceTesting) or its extension points.
Despite its name, this plugin provides the command engine used for integration tests, not only performance tests.

A list of basic out-of-the-box commands that comes with the `performanceTestingPlugin` is available in 
[`generalCommandChain`](https://github.com/JetBrains/intellij-community/blob/master/plugins/performanceTesting/commands-model/src/com/intellij/tools/ide/performanceTesting/commands/generalCommandChain.kt).

## Run with JUnit5

`Starter` is not bound to any test engine and can be run with any of them.
A ready-to-use JUnit5 integration library is also available.
Examples of JUnit5-based tests are available in
[`IdeaJUnit5ExampleTest`](https://github.com/JetBrains/ide-starter-examples/blob/master/intellij.tools.ide.starter.examples/testSrc/com/intellij/ide/starter/examples/junit5/IdeaJUnit5ExampleTest.kt).

## Writing a custom command or performanceTestingPlugin extension

See [createCustomPerformanceCommand.md](documentation/createCustomPerformanceCommand.md).

## How to override/modify default Starter behavior

Any behavior initialized through the `Kodein` DI framework can be modified or extended.
To do so, refer to the
[DI container initialization](https://github.com/JetBrains/intellij-community/blob/master/tools/intellij.tools.ide.starter/src/com/intellij/ide/starter/di/diContainer.kt).

For example, create a custom implementation of `com.intellij.ide.starter.ci.CIServer` and provide it through DI.
Make sure to use the same `Kodein` version specified in the Starter project's `build.gradle`.

Example:

```kotlin
di = DI {
      extend(di)
      bindSingleton<CIServer>(overrides = true) { YourImplementationOfCI() }
}
```

## Freeze/exception collection

Freezes and exceptions are collected by Starter by default and reported as individual test failures on CI.
To enable this machinery, provide
[an implementation of `CIServer`](https://github.com/JetBrains/intellij-community/blob/master/tools/intellij.tools.ide.starter/src/com/intellij/ide/starter/ci/CIServer.kt)
(by default,
[`NoCIServer`](https://github.com/JetBrains/intellij-community/blob/master/tools/intellij.tools.ide.starter/src/com/intellij/ide/starter/ci/NoCIServer.kt)
is used).
For an example implementation, see
[`TeamCityCIServer`](https://github.com/JetBrains/intellij-community/blob/master/tools/intellij.tools.ide.starter/src/com/intellij/ide/starter/ci/teamcity/TeamCityCIServer.kt).
Override `CIServer` in DI (as described in the code snippet above), and reporting of freezes and exceptions will work.

For more detailed customization, the following may be useful.
A test reports errors via
[`ErrorReporter`](https://github.com/JetBrains/intellij-community/blob/master/tools/intellij.tools.ide.starter/src/com/intellij/ide/starter/report/ErrorReporter.kt).
The default implementation is
[`ErrorReporterToCI`](https://github.com/JetBrains/intellij-community/blob/master/tools/intellij.tools.ide.starter/src/com/intellij/ide/starter/report/ErrorReporterToCI.kt).

To customize the header of the error message, provide a custom implementation of
[`FailureDetailsOnCI`](https://github.com/JetBrains/intellij-community/blob/master/tools/intellij.tools.ide.starter/src/com/intellij/ide/starter/report/FailureDetailsOnCI.kt),
which is also registered via DI.

## Debugging the test

> **Tip:** If the `debugger.auto.attach.from.console` registry key is enabled,
> the test can be run under the debugger in IntelliJ IDEA, and attachment happens automatically.

Since the IDE runs as a separate process from the test, the test cannot be debugged directly.
To debug a test, connect remotely to the IDE instance.

General debugging workflow:

1. Create a run configuration for Remote JVM Debug:
  - Debugger mode: Attach to Remote JVM
  - Host: `localhost`
  - Port: `5005`
  - Command line arguments for remote JVM: `-agentlib:jdwp=transport=dt_socket,server=y,suspend=n,address=*:5005`
2. Run the test.
3. The required option will be added automatically.

After seeing the console prompt to connect remotely to port 5005, run the created run configuration.

## Using JUnit5 extensions to modify Starter behavior

For JUnit5, several extensions provide a convenient way to set configuration variables.
A list of extensions is available in the
[config package](https://github.com/JetBrains/intellij-community/tree/master/tools/intellij.tools.ide.starter.junit5/src/com/intellij/ide/starter/junit5/config).

Example:

```kotlin
@ExtendWith(EnableClassFileVerification::class)
@ExtendWith(UseLatestDownloadedIdeBuild::class)
class ClassWithTest {
...
}
```

Environment variables that tweak Starter behavior may also be useful.
They are located in `com.intellij.ide.starter.config.StarterConfigurationStorage`.

## Downloading custom releases

By default, when `useEAP()` or `useRelease()` methods are called,
IDE installers will be downloaded from JetBrains' public hosting. 
If no version is specified, the latest version will be used.
However, a specific version can be specified if needed.

## How to specify another URL for IDE downloading

1. Override the default `IdeDownloader` with `IdeByLinkDownloader`.
   This downloader uses the `downloadURI` field from `IdeInfo`.

```kotlin
init {
  di = DI {
    extend(di)
    bindSingleton<IdeDownloader>(overrides = true) { IdeByLinkDownloader }
  }
}
```

2. Create a custom `IdeInfo` by taking the predefined IntelliJ IDEA Ultimate configuration and overriding only the download URL.

```kotlin
Starter.newContext(
  testName = "custom-ide-download-url",
  testCase = TestCase(
    IdeProductProvider.IU.copy(
      downloadURI = URI("https://example.com/idea-IU-installer.dmg")
    ),
    GitHubProject.fromGithub(
      branchName = "master",
      repoRelativeUrl = "jitpack/gradle-simple.git"
    )
  )
)
```

`IdeProductProvider.IU.copy(downloadURI = URI("https://example.com/idea-IU-installer.dmg"))`
is the key part of this example.
It starts with the standard IntelliJ IDEA Ultimate configuration and changes only the `downloadURI` field,
so Starter will download the IDE from the specified URL.

This example uses `IdeProductProvider.IU` directly, so no additional `IdeInfo.IdeaUltimate` setup is required here.

## Modifying VM Options

There are two ways to modify the VM options.
One is on `IDETestContext`, and the other is on `IDERunContext`.
The first one is used to modify VM options for the whole context that can be reused between runs. The second is used to modify VM options for the current run only.

## Performance testing/Metrics collection

Out of the box, Starter can collect OpenTelemetry metrics using the
[`intellij.tools.ide.metrics.collector.starter`](https://github.com/JetBrains/intellij-community/tree/master/tools/intellij.tools.ide.metrics.collector.starter#readme)
module.

For a more general approach to OpenTelemetry metrics collection (without Starter), see the 
[`intellij.tools.ide.metrics.collector`](https://github.com/JetBrains/intellij-community/tree/master/tools/intellij.tools.ide.metrics.collector#readme)
module.

Unit tests can also be run as benchmark tests via
[`Benchmark.newBenchmark()`](https://github.com/JetBrains/intellij-community/blob/master/tools/intellij.tools.ide.metrics.benchmark/src/com/intellij/tools/ide/metrics/benchmark/Benchmark.java).
See [examples of usages in IntelliJ repo](https://github.com/search?q=repo%3AJetBrains%2Fintellij-community%20Benchmark.newBenchmark&type=code).
  
More details can be found in
[`BenchmarkTestInfo.start()`](https://github.com/JetBrains/intellij-community/blob/master/platform/testFramework/src/com/intellij/testFramework/BenchmarkTestInfo.java),
[`BenchmarkTestInfo.startAsSubtest()`](https://github.com/JetBrains/intellij-community/blob/master/platform/testFramework/src/com/intellij/testFramework/BenchmarkTestInfo.java)
and
[`BenchmarkTestInfoImpl.withMetricsCollector()`](https://github.com/JetBrains/intellij-community/blob/master/tools/intellij.tools.ide.metrics.benchmark/src/com/intellij/tools/ide/metrics/benchmark/BenchmarkTestInfoImpl.java).
