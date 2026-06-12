# Execution Instructions & Strict Rules for Android Development

When generating, reviewing, or modifying Android code, CI/CD pipelines, signing configs, or build environments, you MUST strictly adhere to the following rules. Deviating from these rules is considered a failure.

---

## 1. Project Layout Enforcement (Modern Android)

All Android projects MUST follow a feature-first, multi-module Gradle layout. A typical production app:

```
/
├── app/                              # The main application module
│   ├── build.gradle.kts              # App-level build config
│   ├── proguard-rules.pro
│   └── src/
│       ├── main/AndroidManifest.xml
│       ├── main/kotlin/com/<org>/<app>/
│       │   ├── App.kt                # Application class (@HiltAndroidApp)
│       │   ├── MainActivity.kt       # Single Activity hosting Compose
│       │   ├── ui/theme/             # Material 3 theme
│       │   └── di/                   # Hilt modules (app-level)
│       ├── main/res/
│       ├── test/                     # JVM unit tests
│       └── androidTest/              # Instrumented tests
├── core/                             # Shared infrastructure modules
│   ├── core-ui/                      # Design system, theme, shared Composables
│   ├── core-data/                    # Repositories, DTOs, mappers
│   ├── core-domain/                  # UseCases, domain models
│   ├── core-network/                 # Retrofit, OkHttp, interceptors
│   ├── core-database/                # Room, migrations
│   ├── core-storage/                 # DataStore
│   ├── core-common/                  # Utilities, extensions
│   └── core-testing/                 # Shared test utilities, fakes, fixtures
├── feature/                          # Feature modules (one per major surface)
│   ├── feature-auth/
│   ├── feature-home/
│   ├── feature-profile/
│   └── feature-settings/
├── build-logic/                      # Composite build with convention plugins
│   └── convention/                   # Per-module-type plugins (android-app, android-library, android-feature)
├── gradle/
│   ├── libs.versions.toml            # Version catalog (single source of truth)
│   └── wrapper/
├── .github/workflows/                # CI/CD pipelines
├── Dockerfile                        # Reproducible CI build environment
├── docker-compose.yml                # Local CI parity (optional but recommended)
├── Makefile                          # Standardized commands
├── scripts/                          # Helper scripts
├── docs/
├── keystore/                         # Local debug keystore (gitignored)
├── .gitignore
└── README.md
```

**STRICT RULES:**
- **Multi-module by default** for any app with > 5 screens or > 1 engineer. Single-module is acceptable only for tiny prototypes.
- All modules are **Gradle Version Catalog** consumers (`libs.androidx.compose.material3` etc.). **No hardcoded versions in `build.gradle.kts` files.**
- All modules use the **same** Android `compileSdk`, `minSdk`, `targetSdk`, and JVM target (configured via convention plugins in `build-logic`).
- Application-level code lives in `app/src/main/kotlin/`. NEVER put business logic in `MainActivity` or `Application` subclasses.
- Feature modules MUST be **self-contained**: their public API is the public Composable / UseCase / Repository interface. Internal implementation is hidden.
- Use **Kotlin DSL (`build.gradle.kts`)** exclusively. Groovy `build.gradle` is forbidden for new projects.

---

## 2. Gradle & Dependency Management

- The **`gradle/libs.versions.toml`** file is the **single source of truth** for all dependency versions, plugin versions, and version groups.
- The Gradle Wrapper (`gradle/wrapper/gradle-wrapper.properties`) MUST be committed and pinned to an exact Gradle version (e.g., `8.10.2`).
- The Android Gradle Plugin (AGP) version is pinned in `libs.versions.toml`.
- Use `gradle/libs.versions.toml` `bundles` for grouped dependencies (e.g., `compose = [androidx-activity-compose, androidx-compose-bom, ...]`).
- All modules MUST use the same **Compose BOM** (`androidx.compose:compose-bom`) to align Compose library versions.
- **NEVER** add a transitive dependency directly. Always declare the first-party module.
- Run `./gradlew dependencies` regularly to detect floating versions; pin in CI with `dependencyLocking { lockAllConfigurations() }` and `./gradlew --write-locks` to commit `gradle.lockfile`.
- **Configuration caching** and **build cache** MUST be enabled in `gradle.properties` and CI.
- **Avoid** the `androidx.compose.compiler` Gradle plugin (deprecated since Kotlin 2.0). Use the **Kotlin Compose Compiler plugin** instead.

---

## 3. Coding Standards (Non-Negotiable)

### 3.1 Naming
- **Packages:** lowercase, dot-separated, no underscores (`com.example.feature.profile`).
- **Classes, objects, interfaces, type aliases:** `UpperCamelCase`. Acronyms ≤ 2 letters stay uppercase (`HttpClient`, `RgbColor`); longer acronyms are words (`XmlParser`, not `XMLParser`).
- **Functions, properties, parameters, local variables:** `lowerCotlinCase` is **not** a word — it's `lowerCamelCase`.
- **Constants (`const val`):** `SCREAMING_SNAKE_CASE`.
- **Composable functions:** `UpperCamelCase` (PascalCase) — Composables are types of UI elements.
- **Composable that returns a `Unit` and is a screen root:** suffix `Screen` (e.g., `ProfileScreen`, `HomeScreen`).
- **Composable that renders a single UI element:** prefix or descriptive name (`UserAvatar`, `LoadingIndicator`).
- **ViewModels:** suffix `ViewModel` (`ProfileViewModel`).
- **UseCases / Interactors:** verb-noun (`GetUserUseCase`, `SyncDataUseCase`).
- **Repositories:** noun (`UserRepository`, `AuthRepository`).

### 3.2 Functions & Methods
- Functions should do ONE thing. If the name contains "and", split it.
- Use **expression bodies** for one-liners: `fun isAdmin(): Boolean = role == Role.ADMIN`.
- Use **trailing lambdas** for DSLs and Composables.
- Top-level / extension functions for stateless utilities. Class methods only when state is required.
- Max function length: ~50 lines. Beyond that, refactor.
- **No `!!` operator** in production code. Use safe call `?.`, `let`, `requireNotNull(x) { "reason" }`, or sealed-result types.

### 3.3 Kotlin Idioms
- Prefer `val` over `var`. `var` is only justified when the value genuinely mutates over time.
- Prefer **immutable collections** (`List`, `Map`, `Set`) returned from `val`. Use `MutableList` only internally.
- Use **data classes** for value objects. Use `copy()` for non-destructive updates.
- Use **sealed classes/interfaces** for restricted hierarchies and exhaustive `when` expressions.
- Use **value classes (inline classes)** for newtypes: `@JvmInline value class UserId(val value: String)`.
- Use **type aliases** for complex generic types: `typealias EventHandler = (Event) -> Unit`.
- Use **scope functions** judiciously: `apply` for configuration, `also` for side effects, `let` for null-safety, `with` for grouped calls, `run` for computation. Don't chain more than 2 of them.
- Use **property delegates** (`by lazy`, `observable`, `viewModels()`) where they reduce boilerplate.
- Use **extension functions** instead of utility classes.
- **Use `Result<T>` or a sealed `Outcome<T>` for error handling, NOT exceptions**, in business logic. Exceptions only at framework boundaries.

### 3.4 Coroutines & Concurrency
- Structured concurrency ALWAYS. NEVER use `GlobalScope`.
- Use `viewModelScope` for ViewModel-scope work, `lifecycleScope` for UI-related work.
- Use `repeatOnLifecycle(Lifecycle.State.STARTED)` to collect flows safely from Composables and Fragments.
- Use `withContext(Dispatchers.IO) { ... }` for blocking I/O; use `Dispatchers.Default` for CPU-bound work; use `Dispatchers.Main` only for UI updates.
- Use **Mutex / Mutex.withLock** for shared mutable state. NEVER use `synchronized` for cross-thread.
- Use `Channel` / `Flow` for async event streams. NEVER pass coroutine references between classes.
- Cancellation MUST propagate. `Job.cancel()` must reach child coroutines. NEVER catch `CancellationException` and swallow it.
- `Flow` operators: `map`, `filter`, `combine`, `stateIn`, `shareIn`. Use `shareIn(scope, SharingStarted.WhileSubscribed(), 5_000)` for shared cold flows.
- `StateFlow` for UI state (always has a current value). `SharedFlow` for events. `MutableStateFlow` only inside ViewModels.

### 3.5 Jetpack Compose
- **State hoisting:** Composables should be stateless. Pass state in, emit events up. Hoist state to the nearest common ancestor (usually the ViewModel).
- **Side effects:** use `LaunchedEffect(key)`, `DisposableEffect`, `SideEffect`, `rememberCoroutineScope()`. NEVER use `LaunchedEffect(Unit)` for one-shot init; use `remember` or the ViewModel.
- **Recomposition:** prefer `@Stable` annotated classes. Use `remember` to cache expensive computations. Use `derivedStateOf` for derived state.
- **Modifiers:** pass `Modifier` as the **first optional parameter** to all Composables, named `modifier`. NEVER hardcode `Modifier.padding(16.dp)` inside a reusable Composable.
- **Previews:** every public Composable MUST have a `@Preview` Composable. Use `PreviewParameterProvider` for state variants.
- **Performance:** avoid passing unstable lambdas. Use `remember` for callbacks that capture large state. Use `key()` to break recomposition groups.
- **Lists:** use `LazyColumn` / `LazyRow` / `LazyVerticalGrid`. Provide stable `key`s.
- **Navigation:** use **Navigation Compose** (type-safe routes in 2.8+) or **Voyager**. Each screen is a destination with a Composable + ViewModel.
- **Material 3:** use `androidx.compose.material3.*`. NEVER mix Material 2 in new code.

### 3.6 Architecture
- **Layered architecture:**
  - **UI layer** (`feature-*` modules): Composables + ViewModels. UDF with `StateFlow`.
  - **Domain layer** (`core-domain`): UseCases, domain models, repository interfaces. Pure Kotlin (no Android dependencies).
  - **Data layer** (`core-data`, `core-network`, `core-database`): Repository implementations, DTOs, mappers, Retrofit services, Room DAOs, DataStore.
- **ViewModel** MUST expose immutable `StateFlow<UiState>` and a function `onEvent(event: UiEvent)`. NEVER expose `MutableStateFlow` publicly.
- **Repository** is the single source of truth. It mediates between network, database, and in-memory caches.
- **UseCase** is a single-purpose business operation (`operator fun invoke(...): Flow<Resource<T>>` or `suspend fun ...: Result<T>`).
- **Hilt** is the default DI framework. Every `@Inject` constructor; every Hilt module is `@Module @InstallIn(SingletonComponent::class)` (or the appropriate scope).
- **WorkManager** for deferrable, guaranteed background work. NEVER use `Service` for long-running operations in new code.
- **DataStore** (Preferences or Proto) for all key-value storage. **NEVER** use `SharedPreferences` for new code.

### 3.7 Resources & Strings
- ALL user-facing strings MUST live in `res/values/strings.xml` (and locale-specific variants in `res/values-<locale>/`). **NEVER** hardcode strings in code or layouts.
- All dimensions in `res/values/dimens.xml`. Use `Material 3` token system where possible (`MaterialTheme.spacing.large`).
- All colors in `res/values/colors.xml` (semantic names: `colorPrimary`, `colorOnPrimary`). NEVER reference raw hex in code.
- All drawables go in `res/drawable-*dpi/`. Use **vector drawables** (`VectorDrawable`) over PNGs. Use Material icons.
- All animations use `MotionLayout` (XML) or `animate*AsState` (Compose). NEVER use `ObjectAnimator` directly.

---

## 4. Testing Standards (Mandatory)

### 4.1 The Test Pyramid
1. **Unit tests** (80% of tests): Pure JVM tests under `src/test/`. Fast, hermetic, no Android framework.
2. **Integration tests** (15%): Instrumented tests under `src/androidTest/`. Run on emulator/device.
3. **UI tests** (mandatory for user flows): Compose tests with `androidx.compose.ui.test` or Espresso for XML.
4. **Screenshot / visual tests** (optional but recommended): `Paparazzi` (JVM, fast) or `Roborazzi` (Robolectric-based, more accurate).
5. **Property-based tests** for parsers/serializers: Use `kotest-property`.

### 4.2 Test File Conventions
- Unit tests live in `src/test/kotlin/.../<Name>Test.kt` (`UserRepositoryTest`, `ProfileViewModelTest`).
- Instrumented tests live in `src/androidTest/kotlin/.../<Name>Test.kt` (suffix `Test`).
- Compose UI tests in `src/androidTest/.../<Name>ScreenTest.kt` (suffix `ScreenTest`).
- One test class per production class. One test method per behavior.

### 4.3 Required Test Patterns
- **Given-When-Then** structure with comments or descriptive names.
- **Fakes over mocks** for repositories and data sources. Use `mockk` only at the boundary of the unit under test (e.g., a `Worker` that talks to a `Retrofit` interface).
- **Turbine** for testing `Flow` emissions: `userFlow.test { assertEquals(Loading, awaitItem()) }`.
- **MainDispatcherRule** to swap `Dispatchers.Main` to a test dispatcher in unit tests.
- **Hilt testing:** `@HiltAndroidTest` + `HiltAndroidRule` for instrumented tests; `HiltTestApplication` for Robolectric.
- **Test fixtures:** shared builders and fakes in `core-testing`.
- **Paparazzi** for screenshot tests of Composables without an emulator: `@Test fun profileScreenLight() = paparazzi.snapshot { ProfileScreen(...) }`.
- **Macrobenchmark** for cold-start, frame timing, and scroll performance: `@RunWith(AndroidJUnit4::class) class ColdStartBenchmark`.
- **Baseline profile generation** for hot-path performance: `:benchmarks` module + `androidx.profileinstaller`.

```kotlin
// Example ViewModel test with Turbine + coroutines test
class ProfileViewModelTest {
    @get:Rule val mainDispatcherRule = MainDispatcherRule()
    @Test
    fun `loading user emits Loading then Success`() = runTest {
        val viewModel = ProfileViewModel(fakeGetUserUseCase)
        viewModel.state.test {
            assertEquals(ProfileUiState.Loading, awaitItem())
            viewModel.onEvent(ProfileEvent.Load("u1"))
            assertEquals(ProfileUiState.Success(fakeUser), awaitItem())
            cancelAndIgnoreRemainingEvents()
        }
    }
}
```

### 4.4 Coverage Requirements
- **CI MUST enforce ≥ 80% line coverage** on changed files (Jacoco for unit tests; configured in `app/build.gradle.kts`).
- Coverage of new code must not decrease the overall project coverage.
- 100% coverage is NOT the goal — meaningful assertions are. Test behavior, not getters.

### 4.5 UI Test Discipline
- Every public Composable MUST have at least one preview and one UI test for the primary happy path.
- Use `createComposeRule()` for Compose UI tests; `composeTestRule.setContent { ... }` to host the Composable.
- Use `onNodeWithText`, `onNodeWithTag`, `onNodeWithContentDescription` for semantic lookups. NEVER use `printToLog` to debug.
- Test the **primary user flow** of every feature module: launch → interact → assert end state.
- Use **Robolectric** (`@RunWith(RobolectricTestRunner::class)`) for fast JVM tests of code that depends on Android framework classes (Resources, Context).

### 4.6 Lint, Format, and Static Analysis
- CI MUST run `./gradlew lint` and `./gradlew lintRelease` — fail on any error.
- CI MUST run `./gradlew detekt` (with the team's `config/detekt.yml`) — fail on any error.
- CI MUST run `./gradlew ktlintCheck` — fail on any error.
- CI MUST run `./gradlew test` and `./gradlew testReleaseUnitTest` — all tests must pass.
- CI MUST run `./gradlew connectedAndroidTest` on a Firebase Test Lab device or emulator.
- CI MUST fail on any new `!!` (NotNull assertion) in production code: `detekt.yml` rule `ForbiddenMethodCall { value = "!!" }`.

---

## 5. GitHub Actions CI/CD Pipeline (Required)

You MUST generate a `.github/workflows/` directory with the following workflow files:

### 5.1 `ci.yml` — Continuous Integration
Triggered on `push` to any branch and on `pull_request` to `main`. Steps:
1. Checkout code (`actions/checkout@v4`).
2. Set up JDK 17 via `actions/setup-java@v4` with `distribution: temurin`.
3. **Run inside the project Dockerfile** (`docker build` then `docker run`) OR directly on `ubuntu-latest` for parity.
4. Cache Gradle: `gradle/actions/setup-gradle@v4`.
5. Validate formatting: `./gradlew ktlintCheck detekt`.
6. Lint: `./gradlew lint lintRelease`.
7. Compile: `./gradlew assembleDebug testDebugUnitTest`.
8. Unit tests with coverage: `./gradlew jacocoTestReport` (Jacoco).
9. Upload coverage to Codecov.
10. Build matrix: API 26, API 31, API 34 (current, min, latest).
11. Run instrumented tests on Firebase Test Lab (`firebase/test-lab-action`).

### 5.2 `build-and-distribute.yml` — Build & Distribute
Triggered on `push` to `main` and on version tags (e.g., `v*.*.*`). Steps:
1. Checkout code.
2. Set up JDK 17.
3. **Decrypt the release keystore** from a GitHub Actions secret (`base64 -d` or `gpg`).
4. Build **signed AAB**: `./gradlew bundleRelease`.
5. Build **signed APK** (for direct distribution / QA): `./gradlew assembleRelease`.
6. Upload artifacts to the Play Store via the **Google Play Publisher Gradle plugin** OR `r0adkll/upload-google-play@v1` action.
7. Upload APK artifacts to GitHub Releases.
8. Tag with semantic version.

### 5.3 `codeql.yml`
GitHub CodeQL static analysis on every push and weekly schedule.

### 5.4 `dependency-review.yml` (recommended)
On PR, run `actions/dependency-review-action@v4` to detect vulnerable or unmaintained dependencies.

### 5.5 `release.yml` (optional)
Generate release notes, create GitHub Release with APK + AAB + mapping.txt (ProGuard/R8 mapping for crash deobfuscation).

---

## 6. Dockerfile (CI Build Environment, NOT for the App Itself)

Android apps DO NOT run in Docker — they run on a device/emulator. The Dockerfile standardizes the **CI build environment** so that local and CI builds are bit-for-bit identical.

```dockerfile
# syntax=docker/dockerfile:1.7

FROM eclipse-temurin:17-jdk-jammy AS base
ENV ANDROID_HOME=/opt/android-sdk \
    ANDROID_SDK_ROOT=/opt/android-sdk \
    GRADLE_USER_HOME=/gradle-cache
RUN apt-get update && apt-get install -y --no-install-recommends \
        unzip wget git curl ca-certificates libstdc++6 libgcc-s1 \
    && rm -rf /var/lib/apt/lists/*

# Install Android command-line tools
RUN mkdir -p $ANDROID_HOME/cmdline-tools \
    && cd $ANDROID_HOME/cmdline-tools \
    && wget -q https://dl.google.com/android/repository/commandlinetools-linux-11076708_latest.zip -O cmdline-tools.zip \
    && unzip -q cmdline-tools.zip && rm cmdline-tools.zip \
    && mv cmdline-tools latest

ENV PATH=$PATH:$ANDROID_HOME/cmdline-tools/latest/bin:$ANDROID_HOME/platform-tools

# Install required SDK platforms, build-tools, and NDK
RUN yes | sdkmanager --licenses \
    && sdkmanager "platforms;android-34" "build-tools;34.0.0" "platform-tools" \
    && sdkmanager "ndk;26.1.10909125" "cmake;3.22.1"

# Install Gradle (pinned)
FROM base AS gradle
ARG GRADLE_VERSION=8.10.2
RUN wget -q https://services.gradle.org/distributions/gradle-${GRADLE_VERSION}-bin.zip -O /tmp/gradle.zip \
    && unzip -q /tmp/gradle.zip -d /opt && rm /tmp/gradle.zip \
    && ln -s /opt/gradle-${GRADLE_VERSION}/bin/gradle /usr/local/bin/gradle

FROM gradle AS final
WORKDIR /workspace
CMD ["./gradlew", "assembleDebug"]
```

**STRICT RULES:**
- The Dockerfile standardizes the **CI build environment** (JDK, Android SDK, build-tools, NDK). The Android app itself is NOT a Docker image.
- Pin the JDK version (`eclipse-temurin:17-jdk-jammy`) and Gradle version in the Dockerfile.
- Pin Android SDK components by version (`platforms;android-34`, `build-tools;34.0.0`).
- Accept all SDK licenses via `yes | sdkmanager --licenses` (CI only).
- Use a multi-stage build to cache SDK install from the build layer.
- Provide a `docker-compose.yml` that mounts the project directory and Gradle cache for local CI parity.
- For emulator-based instrumented tests, use `budtmo/docker-android:emulator_*` as a secondary image (or run on Firebase Test Lab directly).
- **NEVER** use a Docker image to ship the Android app to users. The app is delivered as a signed APK/AAB.

You MUST also generate a `.dockerignore` file at the project root excluding at minimum: `.gradle/`, `build/`, `app/build/`, `**/build/`, `.git/`, `local.properties`, `keystore/release.jks`.

---

## 7. Standardized Commands (Makefile)

You MUST generate a `Makefile` at the project root:

```makefile
.PHONY: help build build-debug build-release test test-unit test-ui lint format check clean docker-build docker-run

GRADLE        := ./gradlew
DOCKER        := docker
APP_NAME      := app
DOCKER_IMAGE  := android-ci:local

help: ## Show this help
	@grep -E '^[a-zA-Z_-]+:.*?## .*$$' $(MAKEFILE_LIST) | awk 'BEGIN {FS = ":.*?## "}; {printf "\033[36m%-20s\033[0m %s\n", $$1, $$2}'

build: ## Build debug + release variants
	$(GRADLE) assembleDebug assembleRelease

build-debug: ## Build debug APK
	$(GRADLE) assembleDebug

build-release: ## Build release AAB
	$(GRADLE) bundleRelease

test: ## Run unit and instrumented tests
	$(GRADLE) test connectedAndroidTest

test-unit: ## Run JVM unit tests with coverage
	$(GRADLE) testDebugUnitTest jacocoTestReport

test-ui: ## Run Compose/Espresso UI tests on a connected device
	$(GRADLE) connectedDebugAndroidTest

lint: ## Run Android Lint + detekt + ktlint
	$(GRADLE) lint lintRelease detekt ktlintCheck

format: ## Auto-format with ktlint and Spotless
	$(GRADLE) ktlintFormat spotlessApply

check: ## Run all quality gates
	$(GRADLE) check

clean: ## Clean build outputs
	$(GRADLE) clean
	rm -rf */build .gradle

docker-build: ## Build the CI Docker image
	$(DOCKER) build -t $(DOCKER_IMAGE) .

docker-run: ## Run the CI Docker image interactively
	$(DOCKER) run --rm -it -v "$(PWD):/workspace" -v "$(HOME)/.gradle:/gradle-cache" $(DOCKER_IMAGE) bash

verify-versions: ## Print versions of all build tools
	$(GRADLE) --version
	java -version
```

---

## 8. Configuration & Secrets

- All configuration MUST be loaded from `local.properties` (gitignored) for local dev, environment variables, or the `gradle.properties` keys.
- Use **BuildConfig fields** for compile-time config: `buildConfigField("String", "API_BASE_URL", "\"https://api.example.com\"")`.
- Use `gradle.properties` for **per-developer** config (only that developer's overrides) and `defaultConfig` in `build.gradle.kts` for project-wide defaults.
- **NEVER** hardcode API keys, secrets, or credentials. Use `local.properties` + Gradle's `Properties().load(...)` for local-only values.
- For production secrets: use **Google Cloud Secret Manager**, **AWS Secrets Manager**, or the **Android Keystore** + **Play Integrity API**. In CI, pass via GitHub Actions secrets and read in `build.gradle.kts`.
- The `local.properties` file MUST be in `.gitignore` and MUST NOT be committed.
- Use **`gradle.properties`** for build-wide toggles: `android.useAndroidX=true`, `kotlin.code.style=official`, `org.gradle.parallel=true`, `org.gradle.caching=true`, `org.gradle.configuration-cache=true`, `org.gradle.jvmargs=-Xmx4g -XX:+UseParallelGC`.
- Use **`gradle/libs.versions.toml`** for all dependency versions.

---

## 9. Security Standards

- **Network Security Configuration** (`network_security_config.xml`): disable cleartext traffic (`cleartextTrafficPermitted="false"`) for production builds. Allow cleartext only on `debug` overrides.
- **TLS:** all HTTP traffic MUST be HTTPS. Pin certificates via OkHttp `CertificatePinner` for sensitive APIs.
- **Encrypted storage:** use **`androidx.security:security-crypto`** for `EncryptedSharedPreferences` and `EncryptedFile`. Use **DataStore + Tink** for new code.
- **Android Keystore** for key material. NEVER hardcode keys.
- **WebView:** `setJavaScriptEnabled(false)` by default. Enable only on trusted, internal pages. NEVER load arbitrary user-supplied URLs.
- **Deep links / Intent filters:** declare `android:exported` explicitly. Default to `false`. Verify intent data with `Uri` parsing and signature checks.
- **Permissions:** request **runtime permissions** via `ActivityResultContracts.RequestPermission` ONLY when the feature is used. NEVER request all permissions at app launch.
- **Logging:** strip PII and tokens from logs in production. Use `BuildConfig.DEBUG` to gate `Timber.plant(DebugTree())`.
- **R8/ProGuard:** enabled in release builds (`isMinifyEnabled = true`, `isShrinkResources = true`). Use **R8** (the default since AGP 7.0), not legacy ProGuard.
- **Backup rules:** declare `android:fullBackupContent` and `android:dataExtractionRules` to control what is backed up to Google Drive.
- **Network timeouts:** OkHttp `connectTimeout`, `readTimeout`, `writeTimeout` MUST all be set. Use `callTimeout` as an overall budget.
- **Input validation:** validate ALL user input server-side AND client-side. Use **`org.owasp.encoder`** for HTML escaping in WebViews.

---

## 10. Performance Standards

- **Cold start time** MUST be < 1 second on mid-range devices. Use **Baseline Profiles** to optimize.
- **APK/AAB size budget:** < 20 MB for a typical app. Use **APK Analyzer** and **`bundle.tool`** to investigate bloat.
- **Frame rate:** 60 fps minimum (120 fps on supported devices). Use **Macrobenchmark** to enforce.
- **Memory:** profile with **Android Studio Profiler** (Memory, CPU, Network). Watch for memory leaks via LeakCanary (debug builds only).
- **Image loading:** use **Coil** (Compose-native) or **Glide**. Cache aggressively. Use WebP / AVIF over PNG.
- **Lazy loading:** use `LazyColumn` / `LazyRow` for lists. Paginate network results (Paging 3).
- **Background work:** use **WorkManager** for deferrable work. NEVER use `Thread` or raw `Service`.
- **Battery:** respect `PowerManager.isPowerSaveMode()` and `JobScheduler` constraints. Defer non-critical work on low battery.
- **Cold start optimization:** use **App Startup** library for one-time init. Defer non-critical library init until after first frame.

---

## 11. Accessibility Standards

- **Content descriptions:** every meaningful interactive View/Composable MUST have `contentDescription` / `Modifier.semantics { contentDescription = ... }`.
- **Touch targets:** minimum 48dp x 48dp. Use `Modifier.minimumInteractiveComponentSize()`.
- **Dynamic type:** use `MaterialTheme.typography` (which scales with the user's font preference). NEVER hardcode `Text(fontSize = 14.sp)`.
- **TalkBack:** test the full app with TalkBack enabled. Every interactive element must be announced correctly.
- **Color contrast:** verify text contrast ratios ≥ 4.5:1 (normal) / 3:1 (large text). Use the [WebAIM Contrast Checker](https://webaim.org/resources/contrastchecker/).
- **Reduce motion:** respect `Settings.Global.ANIMATOR_DURATION_SCALE` via `MotionDurationScale` and provide reduced-motion alternatives.
- **RTL support:** declare `android:supportsRtl="true"`. Use `start`/`end` instead of `left`/`right` in layouts and modifiers.
- **Color-blind support:** don't rely on color alone to convey state. Use shape, text, and iconography together.

---

## 12. Dependency Hygiene

- All dependencies live in **`gradle/libs.versions.toml`**. No hardcoded versions elsewhere.
- Run `./gradlew :app:dependencies --refresh-dependencies` and review for floating versions.
- Run **`dependency-check-gradle`** or **OWASP Dependency-Check** in CI to detect known CVEs.
- **Renovate** or **Dependabot** MUST be enabled to keep dependencies and GitHub Actions up to date.
- Reject any dependency that has not had a release in the last 18 months OR is marked as unmaintained.
- Pin the **Android Gradle Plugin**, **Kotlin**, **Compose BOM**, and all **Hilt**, **Retrofit**, **Room** versions.
- For AndroidX libraries, always prefer the BOM (`androidx.compose:compose-bom`, `androidx.room:room-bom`, etc.) over pinned versions.

---

## 13. Commit & Branching Standards

- Branch naming: `<type>/<ticket-id>-<short-description>` (e.g., `feat/PROJ-123-add-user-registration`).
- Commit messages follow **Conventional Commits**:
  - `feat:`, `fix:`, `chore:`, `docs:`, `style:`, `refactor:`, `perf:`, `test:`, `build:`, `ci:`
  - Example: `feat(profile): add Hilt-bound ProfileViewModel`
- Squash-merge feature branches to `main` with a single conventional commit.
- Every PR MUST reference a ticket/issue ID.
- `main` branch is protected: requires PR, 1+ review, all CI checks green, no direct pushes.
- Every PR MUST pass `./gradlew check` (lint + detekt + ktlint + test) before merge.

---

## 14. Release & Signing

- Use **Play App Signing** (Google manages the upload key) and keep the **upload key** in GitHub Actions secrets.
- Generate the upload keystore ONCE and store its base64 in a GitHub Actions secret: `KEYSTORE_BASE64`, `KEYSTORE_PASSWORD`, `KEY_ALIAS`, `KEY_PASSWORD`.
- NEVER commit the release keystore to the repo. The `keystore/` directory is `.gitignore`d (except for the local `debug.keystore`).
- Use the `signingConfigs` block in `app/build.gradle.kts`:
  - `debug` uses the default Android debug keystore.
  - `release` reads from `local.properties` for local builds, from `System.getenv()` in CI.
- Bump `versionCode` and `versionName` on every release. Use `versionCode` as a monotonic integer; `versionName` is human-readable (semver: `MAJOR.MINOR.PATCH`).
- Maintain a `CHANGELOG.md` (or release notes per version in the Play Console).
- Crash deobfuscation: upload `mapping.txt` (from `app/build/outputs/mapping/release/`) to **Crashlytics** / **Sentry** / **Play Console** with every release so that production stack traces are deobfuscated.

---

## 15. Pre-Commit Checklist (Self-Audit)

Before declaring work complete, the agent MUST verify:

- [ ] `./gradlew ktlintCheck` produces no errors
- [ ] `./gradlew detekt` produces no errors
- [ ] `./gradlew lint lintRelease` produces no errors
- [ ] `./gradlew test` passes (all unit tests)
- [ ] `./gradlew testReleaseUnitTest` passes
- [ ] `./gradlew connectedAndroidTest` passes on a connected emulator/device
- [ ] `./gradlew jacocoTestReport` shows ≥ 80% coverage on changed files
- [ ] `./gradlew assembleDebug` succeeds
- [ ] `./gradlew bundleRelease` succeeds (signed AAB builds)
- [ ] No new `!!` (NotNull assertion) in production code
- [ ] No new `GlobalScope.launch` in production code
- [ ] No hardcoded strings in code or layouts (all in `strings.xml`)
- [ ] No hardcoded secrets, tokens, or credentials in the diff
- [ ] All new Composables have `@Preview` annotations
- [ ] All new ViewModels have unit tests with Turbine
- [ ] All new public APIs in `core-*` modules are documented (KDoc)
- [ ] `AndroidManifest.xml` declares all permissions explicitly
- [ ] `network_security_config.xml` disables cleartext for release
- [ ] ProGuard/R8 rules updated for any reflection-using library
- [ ] Baseline profile regenerated if hot paths changed
- [ ] `CHANGELOG.md` updated for user-facing changes
- [ ] `README.md` updated if setup steps, env vars, or build commands changed
- [ ] `gradle/libs.versions.toml` updated with any new dependency
- [ ] CI workflow files validate locally with `actionlint`
- [ ] Dockerfile builds successfully: `docker build -t android-ci:local .`
- [ ] `local.properties` is in `.gitignore`
- [ ] Keystore directory is in `.gitignore`

---

## Output Format for the Agent

When generating a new Android app, the agent MUST output, at minimum:

1. The full directory tree of the generated multi-module project.
2. The complete contents of every file created (with `path/to/file.ext` headers).
3. A `README.md` containing: project description, prerequisites (JDK, Android Studio, SDK), build commands (`./gradlew ...`, `make ...`), module map, and a "How to release" section.
4. A summary of decisions made and the trade-offs (e.g., why Hilt over Koin, why Compose over XML, why single-Activity over multi-Fragment).

When reviewing or modifying existing Android code, the agent MUST output:

1. A unified diff of every changed file.
2. A categorized list of issues: `BLOCKING`, `WARNING`, `OPTIMIZATION`, `NIT`.
3. For each issue: file path, line number, the rule violated (with link to this document or a referenced Android Developers doc), and a suggested fix in code.
4. A final verdict: `[APPROVED]`, `[APPROVED WITH WARNINGS]`, or `[REJECTED — CHANGES REQUIRED]`.
