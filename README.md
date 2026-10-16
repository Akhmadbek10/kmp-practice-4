# kmp-practice-4
# Kotlin Multiplatform — Practice 4: Answers (Questions 14–25)

**Course:** Mobile Apps (Native and Web)
**Topic:** KMP Project Structure (continuation) — Lecture 10 (Advanced Project Structure) and project configuration
**Student:** Akhmadbek Usmanov SE-17800

> **Sources.** Questions 14–15 and 18–20 are answered from the Lecture 10 slides. Questions 22–25 are based on the official Kotlin Multiplatform documentation page [Choosing a configuration for your Kotlin Multiplatform project](https://kotlinlang.org/docs/multiplatform/multiplatform-project-configuration.html). Hands-on questions (16, 17, 21) show the code and the result observed in Android Studio.

## Contents

**Section 3 — KMP Advanced Project Structure**

- [Question 14 — `dependsOn` vs `dependencies {}`](#question-14)
- [Question 15 — Misusing `dependsOn`](#question-15)
- [Question 16 — Custom source set `jvmAndJsMain` (hands-on)](#question-16)
- [Question 17 — Default hierarchy template caveat (hands-on)](#question-17)
- [Question 18 — Propagation](#question-18)
- [Question 19 — Why `asExecutor()` is unresolved](#question-19)
- [Question 20 — Version alignment](#question-20)
- [Question 21 — Compilations (hands-on)](#question-21)

**Section 4 — Choosing a KMP Project Configuration**

- [Question 22 — Single module vs several modules](#question-22)
- [Question 23 — Umbrella module](#question-23)
- [Question 24 — Monorepo, two repositories, three repositories](#question-24)
- [Question 25 — Design for a 200-engineer company](#question-25)

---

# Section 3 — KMP Advanced Project Structure (Lecture 10)

## Question 14

**Topic:** `dependsOn` is not `dependencies {}`

The claim is wrong because the two are different kinds of relation.

- `dependencies {}` is a Gradle-level dependency: "my code uses this library or module". It is a **packaging/build** relation.
- `dependsOn` is a Kotlin-specific relation between two **source sets**: "A relies on B for its common declarations". It is a **visibility/actualization** relation, not a Gradle dependency.

When `A.dependsOn(B)` is declared, all four of these hold **simultaneously**:

1. A observes B's API, including `internal` declarations.
2. A can provide `actual` implementations for B's `expect` declarations.
3. B compiles to every target A compiles to, plus its own.
4. A inherits all of B's regular dependencies.

So `dependsOn` drives visibility, actuals and target inheritance, not packaging. That is why it is declared outside `dependencies {}`, and why the compiler consults it directly at compile time.

## Question 15

**Topic:** `commonTest.dependsOn(commonMain)` and `dependsOn` on a library

`dependsOn` is reserved for building the source set hierarchy between source sets **inside one project**. Both attempts are therefore mistakes:

- **`commonTest.dependsOn(commonMain)`**: main and test source sets live in **separate trees** (Lecture 10, slide 10). Linking them with `dependsOn` mixes the production and test hierarchies, and by consequence 3 it also changes which targets each one compiles to. Test code does not need `dependsOn` to see production code; Kotlin associates the test compilation with the main compilation automatically.
- **`dependsOn` on a library's own `commonMain`**: `dependsOn` cannot be used for a library or for another Gradle project's source set (slide 4). A library is a published artifact, and Kotlin resolves it down to source set → source set edges by itself.

To depend on a library or another module, use the **`dependencies {}`** block: Maven coordinates (`"group:artifact:version"`) or a project path, with scopes such as `implementation` and `api`. Every source set has its own `dependencies {}` block.

## Question 16

**Topic:** Custom source set `jvmAndJsMain` (hands-on)

`build.gradle.kts` of the KMP module:

```kotlin
plugins {
    kotlin("multiplatform")
}

kotlin {
    jvm()
    js(IR) { browser() }
    linuxX64()

    sourceSets {
        // 1) A new source set, isolated for now
        val jvmAndJsMain by creating { }

        // 2) Wire it into the tree with three dependsOn calls
        jvmAndJsMain.dependsOn(commonMain.get())
        jvmMain.dependsOn(jvmAndJsMain)
        jsMain.dependsOn(jvmAndJsMain)
    }
}
```

Shared function in `src/jvmAndJsMain/kotlin/Greeting.kt`:

```kotlin
fun sharedGreeting(name: String): String =
    "Hello, $name! (shared between JVM and JS)"
```

Expected result:

- Gradle sync finishes without errors and `./gradlew build` succeeds.
- `sharedGreeting` is visible from `jvmMain` and `jsMain`, but **not** from `linuxX64Main`, because linuxX64 is not wired to this source set.
- Inside the function only APIs common to JVM and JS can be used, not platform-specific ones such as `java.*`.

Why three calls: the first connects `jvmAndJsMain` to the tree through `commonMain`. The other two make the existing `jvmMain` and `jsMain` depend on it, which is what actually pulls the JVM and JS target labels onto the new source set (slide 5, BEFORE → AFTER).

<!-- Screenshot (optional): ![Question 16 – sync and build](screenshots/q16.png) -->

## Question 17

**Topic:** Default hierarchy template caveat (hands-on)

Add the two iOS targets:

```kotlin
kotlin {
    jvm()
    js(IR) { browser() }
    linuxX64()
    iosArm64()
    iosSimulatorArm64()
    // sourceSets { ... the three dependsOn calls from Question 16 ... }
}
```

A helper task to list the source sets (add at the end of `build.gradle.kts`):

```kotlin
tasks.register("printSourceSets") {
    doLast {
        println(kotlin.sourceSets.names.sorted())
    }
}
```

Run `./gradlew printSourceSets` and also look under `src/` in the Project view.

**Expected observation:**

- If the manual `dependsOn` calls are temporarily commented out, the default hierarchy template applies and intermediate source sets such as `appleMain`, `iosMain`, `nativeMain` and `linuxMain` are created automatically.
- With the manual `dependsOn` calls in place they **disappear**: `iosArm64Main` and `iosSimulatorArm64Main` attach directly to `commonMain`. The build log warns that the Default Kotlin Hierarchy Template was not applied because explicit `.dependsOn()` edges were configured.

**Explanation (the lecture caveat):** the Kotlin Gradle plugin applies the default hierarchy template only when the project has no manually configured `dependsOn` edges. A single manual `dependsOn` opts the whole project out, so `appleMain` or `iosMain` have to be created and wired by hand if they are needed (slide 12: "check which default source sets disappeared").

Two ways out: remove the manual calls and add the `jvmAndJs` group through the template's own mechanism (`applyDefaultHierarchyTemplate { ... }` with a custom group), or create and wire `iosMain` / `appleMain` manually as well.

<!-- Screenshot (optional): ![Question 17 – printSourceSets output](screenshots/q17.png) -->

## Question 18

**Topic:** Propagation

Declared once in `commonMain.dependencies { implementation("…coroutines-core:1.7.3") }`, the dependency is **automatically propagated** to every source set that depends on `commonMain`, directly or transitively (slide 7). This follows from consequence 4 of `dependsOn`: a source set inherits all regular dependencies of the source set it depends on.

So it does **not** need to be added separately to `androidMain` or `iosMain`: one declaration covers all three targets (android, iosArm64, iosSimulatorArm64).

To scope the same dependency to iOS only, declare it on `iosMain` instead of `commonMain`:

```kotlin
iosMain.dependencies {
    implementation("org.jetbrains.kotlinx:kotlinx-coroutines-core:1.7.3")
}
```

Per the lecture, the source set you pick is itself the scoping decision.

## Question 19

**Topic:** Why `asExecutor()` is unresolved

**The rule (slide 9):** a dependency's source set is compatible only if it covers at least all of your source set's targets.

Here `commonMain` compiles to android, iosArm64 and iosSimulatorArm64. The published `kotlinx-coroutines-core` ships its own source sets (slide 8):

- Library `commonMain`: covers **every** target, so it is compatible.
- Library `concurrentMain`: covers every target **except JS**, so it still covers android and iOS and is compatible.
- Library JVM-only source set: covers only JVM/Android, not the iOS targets, so it is **not compatible**.

The three calls:

- `Dispatchers.Main` resolves from the library's `commonMain`, so it compiles.
- `runBlocking` resolves from the library's `concurrentMain` (as the slide shows), so it compiles.
- `dispatcher.asExecutor()` lives in the JVM-only source set. That source set does not cover iosArm64 and iosSimulatorArm64, so in code shared by all three targets it is unresolved (red). Exactly one of the three calls fails for that reason.

## Question 20

**Topic:** Version alignment

`commonMain` declares coroutines `1.7.3`, while `androidx.navigation-compose:2.7.7` (added in `androidMain`) needs `1.8.0` or newer. `commonMain` is compiled several times, once per target, and **every compilation must agree on one version**. Gradle therefore resolves the conflict automatically: it raises `commonMain` from `1.7.3` to **`1.8.0`** and propagates that version everywhere below it. As a result both `commonMain` and `iosMain` end up on `1.8.0`, even though only `androidMain` asked for it.

Test source sets are aligned on their **own separate tree** (slide 10, "TEST SOURCE SETS (SEPARATE!)"). So upgrading a dependency in `iosTest` only affects the test tree and never changes the versions used by any `*Main` (production) source set.

## Question 21

**Topic:** Compilations (hands-on)

`build.gradle.kts`:

```kotlin
kotlin {
    jvm {
        val mainCompilation = compilations.getByName("main")
        println("JVM main compilation name: ${mainCompilation.name}")
        println("JVM compilations: ${compilations.names}")
    }
}
```

During Gradle sync (or `./gradlew help`) the Build window shows roughly:

```
JVM main compilation name: main
JVM compilations: [main, test]
```

This confirms that a `test` compilation is registered next to `main`: every target automatically gets two compilations, `main` (production sources) and `test` (test sources).

**Difference in one sentence:** a source set is a group of source files and dependencies (what the code consists of), while a compilation is one compiler run for a target that takes those source sets and produces an artifact.

<!-- Screenshot (optional): ![Question 21 – compilations output](screenshots/q21.png) -->

---

# Section 4 — Choosing a KMP Project Configuration

## Question 22

**Topic:** Single module vs several modules

**Single shared module**

- Pros: a simple design with low cognitive load (no need to decide where functionality goes); a great starting point.
- Cons: compilation time grows as the module grows; features cannot be separated, and an app cannot depend only on the features it needs.

**Several shared modules**

- Pros: separation of concerns for shared code; better scalability.
- Cons: more complicated setup, including the umbrella framework; more involved dependency management across modules.

**When to move:** a team starts with one module. When the shared module has grown, compile times hurt, and features need to be separated, it is time to split into feature modules. The cost is a more complex setup (umbrella module); the gain is scalability.

## Question 23

**Topic:** Why per-module iOS frameworks break down, and the umbrella module

**The problem (dependency duplication):** when a KMP module is compiled into a framework, the framework includes **all of its dependencies**. If two or more modules use the same dependency and are exposed to iOS as separate frameworks, the Kotlin/Native compiler **duplicates** that dependency. Consequences:

1. The iOS app size is inflated for no reason.
2. The duplicated dependency's code structure is incompatible with the original, which makes integrating two modules with the same dependency in one app problematic.
3. **Concrete failure mode:** any state passed by different modules through "the same" dependency is not shared, because each framework actually holds its own copy of that dependency. This leads to unexpected behavior and bugs.

**Umbrella module:** an extra module that depends on all the feature modules, and **the only module that generates the iOS framework** (the umbrella framework). Shared dependencies then appear **once** inside that framework, so app size drops and the incompatibilities disappear. The Android app can depend on the umbrella module or directly on feature modules. The umbrella module often holds utility functions and dependency injection setup, and only some modules can be exported into the framework. Known constraint: the iOS app cannot pick only some of the feature modules; it consumes all of them.

## Question 24

**Topic:** Monorepo, two repositories, three repositories

**Monorepo (everything in one repository)**

- Advantage: easy to set up with wizards; iOS developers can easily work with the Kotlin code because everything is in the same repository.
- Disadvantage: iOS developers must set up and configure unfamiliar tools; it often does not work for existing apps already stored in different repositories.
- Versioning: the apps and the shared module have the same version.

**Two repositories: Android + shared | iOS**

- Advantage: existing apps stay in separate repositories while still adding KMP; the Android app uses the shared module as a regular module, with no separate publishing for Android.
- Disadvantage: the shared code sits in the Android repository, so iOS consumes it as a built artifact, and the shared code can start to feel "owned" by Android developers.
- Versioning: Android and iOS apps are versioned separately; the shared module is versioned together with the Android app.

**Three repositories: Android | iOS | shared**

- Advantage: both teams consume the same versioned artifacts (version parity), so shared code is not seen as Android-owned; it fits large projects that already publish versioned internal packages.
- Disadvantage: publishing Android artifacts separately adds complexity for Android developers compared with having the modules inside the Android project.
- Versioning: each project is versioned separately.

**The versioning change between two and three repositories:** with two repositories the shared module is versioned with the Android app and needs no separate Android publication. With three repositories the KMP shared modules must **also be versioned and published for Android/JVM**, either each feature module on its own or only the umbrella module (with the Android app depending on it).

_Note: the documentation does not list pros and cons for the two-repository setup as explicitly as for the others, so those points are drawn from its text._

## Question 25

**Topic:** 200 engineers, five apps, shared logging, networking and analytics

**1) Number of repositories.** Use the "many repositories" configuration:

- One **separate KMP repository per shared library**: logging, networking and analytics, so **3 library repositories**. Each has its own versioning and is published for Android/JVM as well as iOS.
- The Android and iOS apps stay in their own repositories.

**2) Each app consumes only what it needs.**

- Android: each app depends directly on the versioned libraries it needs; nothing else is pulled in.
- iOS: the umbrella framework forces an app to take every module inside it, so each iOS app gets **its own umbrella module** (in an additional repository) that depends only on the libraries that app uses. This is the documented scheme for iOS apps that use different subsets of libraries. It also avoids the duplication problem from Question 23.

**3) Distribution: hybrid (remote by default, local for development).**

- **Local only** does not fit: it requires every team member to have a full environment (Android Studio, Gradle, Xcode), and managing changes becomes difficult as more code is shared and the team grows. With 200 engineers and five apps this does not scale.
- **Remote** (artifact via SwiftPM or CocoaPods) fits a large organization: iOS developers who do not touch shared code never need to learn Kotlin or Gradle, and it matches workflows that already publish versioned internal packages. Its drawbacks are a slower edit loop (publish and version), harder debugging of Kotlin on iOS, and fewer iOS contributions to shared code.
- So use **hybrid**: remote artifacts by default; when developing new functionality, a developer temporarily switches to a **local dependency** to see changes from iOS immediately and debug the Kotlin code. When the work is done, they switch back to remote and publish: first the shared libraries, and only then the apps.

**Summary:** 3 library repositories, one umbrella repository per iOS app (5 in total), Android apps depending directly on the libraries they need, and distribution via remote artifacts complemented by local development.

