<!-- Copyright 2000-2025 JetBrains s.r.o. and contributors. Use of this source code is governed by the Apache 2.0 license. -->

# Integration Tests with Driver

<link-summary>Using the Driver API to call code in a running IDE via JMX for end-to-end testing.</link-summary>

The `com.intellij.driver.client.Driver` API provides a generic interface to call code in a running IntelliJ Platform-based IDE instance, such as service and utility methods.
It connects to a process via [JMX](https://en.wikipedia.org/wiki/Java_Management_Extensions) protocol and creates remote proxies for classes of the running IDE.
The main purpose of this API is to execute IDE actions and observe the state of the process in end-to-end testing.

## Connecting to a Running IDE

Driver uses JMX as the underlying protocol to call IDE code.
To connect to an IDE via Driver, start it with the following VM Options:

```bash
-Dcom.sun.management.jmxremote=true
-Dcom.sun.management.jmxremote.port=7777
-Dcom.sun.management.jmxremote.rmi.port=5000
-Dcom.sun.management.jmxremote.authenticate=false
-Dcom.sun.management.jmxremote.ssl=false
-Djava.rmi.server.hostname=<host-ip>
```

Then, a driver can be created to call the IDE:

```kotlin
val driver = Driver.create(JmxHost(null, null, "<host-ip>:7777"))
assertTrue(driver.isConnected)
println(driver.getProductVersion())
driver.exitApplication()
```

## @Remote API Calls

The main use case for Driver is calling arbitrary services and utilities of the IDE and plugins.

To call any code, create an interface annotated with `@Remote` (`com.intellij.driver.client.Remote`).
It must declare methods with the same name and number of parameters as the actual class in the IDE.
Example:

```kotlin
@Remote("com.intellij.psi.PsiManager")
interface PsiManager {
  fun findFile(file: VirtualFile): PsiFile?
}

@Remote("com.intellij.openapi.vfs.VirtualFile")
interface VirtualFile {
  fun getName(): String
}

@Remote("com.intellij.psi.PsiFile")
interface PsiFile
```

Then it can be used in the following call:

```kotlin
driver.withReadAction {
  // access the Project-level service.
  val psiFile = service<PsiManager>(project).findFile(file)
}
```

Supported types of method parameters and results:
- primitives and their wrappers: `Integer`, `Short`, `Long`, `Double`, `Float`, `Byte`
- `String`
- `@Remote` reference
- Array of primitive values, `String` or `@Remote` references
- Collection of primitive values, `String` or `@Remote` references

To use classes that are not primitives, create the corresponding `@Remote` mapped interface and use it instead of the original types in method signatures.

If a plugin (not the IntelliJ Platform) declares a required service/utility, the plugin identifier must be specified in `Remote.plugin` attribute:

```kotlin
@Remote("org.jetbrains.plugins.gradle.performanceTesting.ImportGradleProjectUtil", 
        plugin = "org.jetbrains.plugins.gradle")
interface ImportGradleProjectUtil {
  fun importProject(project: Project)
}
```

Only `public` methods can be called.
`Private`, `package-private` and `protected` methods are supposed to be changed to `public`.
Mark methods with `org.jetbrains.annotations.VisibleForTesting` to show that they are used from tests.

Service and utility proxies can be acquired on each call.
There is no need to cache them in clients.

Any IDE class may have as many different `@Remote` mapped interfaces as needed — declare another one if the standard SDK does not provide the required method.

Put common platform `@Remote` mappings to `intellij.driver.sdk` module under `com.intellij.driver.sdk` package.

## Invoking UI Actions

There is a shorthand method to trigger actions from tests in Test SDK:

```kotlin
driver.invokeAction("SearchEverywhere")
```

## Contexts and Remote References

Managing references to objects that reside in a separate JVM process is inherently non-trivial.
To prevent memory leaks, Driver uses `java.lang.ref.WeakReference` for call results.

Consider the following example:

```kotlin
val roots = driver.service(ProjectRootManager::class, driver.singleProject()).getContentRoots()
val name = roots[0].getName() // may throw an error
```

In many cases, it throws an exception:
> Weak reference to variable 12 expired. Please use `Driver.withContext { }` for hard variable references.

To use a result later, there must be additional measures to preserve references between calls.
Such measures are called context boundary:

```kotlin
driver.withContext {
  val roots = service<ProjectRootManager>.getContentRoots()
  val name = roots[0].getName() // always OK!

  // results computed inside guaranteed to be alive till the end of the block
}
```

Driver supports many nested context boundaries, and they can be used independently in helper methods, e.g.:

```kotlin
fun Driver.importGradleProject(project: Project? = null) {
  withContext {
    val forProject = project ?: singleProject()
    this.utility(ImportGradleProjectUtil::class).importProject(forProject)
  }
}
```

## UI Testing

Test SDK provides an additional API to simplify simulation of user actions via `com.intellij.driver.sdk.ui.UiRobot`.
Start with calling `driver.ui` to get a `UiRobot` instance, then find UI components with XPath selectors:

```kotlin
driver.ui.welcomeScreen {
  val createNewProjectButton = x("//div[@accessiblename='New Project' and @class='JButton']")
  createNewProjectButton.click()
}
```

Note that `x()` and `xx()` methods do not perform the actual search of a UI component on screen.
The search is done on the first immediate action such as a `click()` or asserts via `should()`:

```kotlin
val header = x("//div[@text='AI Assistant']")
header.shouldBe("AI assistant header not present", visible)
```

To simplify exploration of UIs and make XPath selectors easier to write, use the UI hierarchy web interface.
It can be enabled via a VM option `-Dexpose.ui.hierarchy.url=true`.
The UI hierarchy is then available from a web browser at `http://localhost:<built-in server port>/api/remote-driver/`.

The Page Object pattern allows locators to be reused:

```kotlin
fun Finder.welcomeScreen(action: WelcomeScreenUI.() -> Unit) {
  x("//div[@class='FlatWelcomeFrame']", WelcomeScreenUI::class.java).action()
}

class WelcomeScreenUI(data: ComponentData) : UiComponent(data) {
  private val leftItems = tree("//div[@class='Tree']")

  fun clickProjects() = leftItems.clickPath("Projects")
}
```

The usage can be simplified to:

```kotlin
driver.ui.welcomeScreen {
  clickProjects()
}
```

## Waiting

There are two ways to wait for a condition with a timeout:

1. [Awaitility](https://github.com/awaitility/awaitility) library
2. `should()` methods of UI components

For common IDE states, the Test SDK also provides the following helpers:

```kotlin
// 1. there must be an opened project and all progresses finished
waitForProjectOpen(timeout)

// 2. all progresses must disappear from status bar
waitForIndicators(project, timeout)

// 3. daemon must finish analysis in a file
waitForCodeAnalysis(file)
```

## Bootstrapping IDE for Test

Creating a test requires two main steps:
1. Create an `IDETestContext` using `Starter.newContext()`
2. Start the IDE using `IDETestContext.runIdeWithDriver()`

The simplest test looks like:

```kotlin
class OpenGradleJavaFileTest {
    private lateinit var bgRun: BackgroundRun

    @BeforeEach
    fun startIde() {
        bgRun = Starter.newContext(ideInfo = IdeProductProvider.IU, testName = "testExample") {
            project = GitHubProject.fromGithub(branchName = "master", repoRelativeUrl = "JetBrains/ij-perf-report-aggregator")
        }.runIdeWithDriver()
    }

    @Test
    fun import() {
        bgRun.useDriverAndCloseIde {
            // the test using a driver
        }
    }
}
```

To reuse the IDE between tests and manage the IDE run in `@BeforeAll`/`@AfterAll`:

```kotlin
class OpenGradleJavaFileTest {
    companion object {
        private lateinit var run: BackgroundRun

        @BeforeAll
        @JvmStatic
        fun startIde() {
            run = Starter.newContext(ideInfo = IdeProductProvider.IU, testName = "testExample") {
                project = GitHubProject.fromGithub(branchName = "master", repoRelativeUrl = "JetBrains/ij-perf-report-aggregator")
            }.runIdeWithDriver()
        }

        @AfterAll
        @JvmStatic
        fun closeIde() {
            run.closeIdeAndWait()
        }
    }

    @Test
    fun import() {
        run.driver.withContext {
            //the test goes here
        }
    }
}
```

Tests that follow the convention will work for local IDE runs and for RemDev with client/host where the driver instance will be a driver of client.