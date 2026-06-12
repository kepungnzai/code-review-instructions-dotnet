# Agent Skills & Knowledge Base: Android Engineering

You possess expert-level proficiency in the following domains. Use this knowledge to make authoritative recommendations and write idiomatic, production-grade Android code.

---

## 1. Kotlin Language Mastery (Android-Focused)

### 1.1 Core Language
- **Coroutines** are the standard for async. Use `viewModelScope`, `lifecycleScope`, `repeatOnLifecycle`.
- **Flow** for reactive streams: `StateFlow` (state with current value), `SharedFlow` (events), `MutableSharedFlow` / `MutableStateFlow` (private only).
- **Sealed classes/interfaces** for UI state and one-shot events:
  ```kotlin
  sealed interface ProfileUiState {
      data object Loading : ProfileUiState
      data class Success(val user: User) : ProfileUiState
      data class Error(val message: String) : ProfileUiState
  }
  ```
- **Data classes** for value objects. **Value classes** for newtypes (`@JvmInline value class UserId(val value: String)`).
- **Extension functions** for utility and DSLs.
- **Scope functions:** `apply` (configuration), `also` (side effects), `let` (null-safety, scope change), `with` (grouped calls), `run` (computation + result).
- **Destructuring** for data class component access.
- **Property delegates:** `by lazy { ... }`, `by viewModels()`, `Delegates.observable(...)`.

### 1.2 Standard Library Mastery
- `kotlinx.coroutines` and `kotlinx.coroutines.flow` for all async.
- `kotlinx.serialization` for JSON (or `moshi` / `gson` for legacy).
- `kotlinx.datetime` for date/time (or `java.time` from API 26+).
- `kotlinx.collections.immutable` for persistent collections (`persistentListOf`, `persistentMapOf`).
- `Arrow` (optional) for typed errors (`Either<L, R>`) and functional patterns.

### 1.3 Java Interop
- Annotate Java-facing APIs with `@JvmStatic`, `@JvmField`, `@JvmName`, `@JvmOverloads`.
- Use `@file:JvmName("CustomName")` for top-level functions in utility files.
- SAM conversions for Java functional interfaces.

---

## 2. Jetpack Compose

### 2.1 Core APIs
- `setContent { ... }` in `ComponentActivity` (or `ComponentActivity.setContent` with `enableEdgeToEdge()`).
- Composables: stateless by default, hoist state.
- **`collectAsStateWithLifecycle()`** for collecting flows in Composables (lifecycle-aware).
- **`produceState()`** for converting non-Compose state into Compose state.
- **`derivedStateOf { ... }`** for derived state that should only recompose when the result changes.
- **`remember(key1, key2) { ... }`** for caching expensive computations.
- **`rememberSaveable { ... }`** for state that survives config changes and process death.
- **`rememberCoroutineScope()`** to launch coroutines in response to events.

### 2.2 Side Effects
- `LaunchedEffect(key)` — runs a coroutine when keys change.
- `DisposableEffect(key)` — runs setup and teardown code.
- `SideEffect` — runs after every successful recomposition (synchronous).
- `rememberUpdatedState(value)` — capture a value in an effect that doesn't restart when the value changes.
- `SnapshotFlow { ... }` — observe Compose state as a Flow.

### 2.3 Layouts
- `Column`, `Row`, `Box` — basic containers with vertical/horizontal/stacked alignment.
- `LazyColumn`, `LazyRow`, `LazyVerticalGrid` — for lists.
- `ConstraintLayout` (via `androidx.constraintlayout:constraintlayout-compose`) — for complex responsive layouts.
- `SubcomposeLayout` — for custom layout that depends on children's measured sizes.

### 2.4 Material 3
- `androidx.compose.material3.*` — M3 components.
- `MaterialTheme.colorScheme`, `MaterialTheme.typography`, `MaterialTheme.shapes`.
- Dynamic color (Android 12+): `dynamicDarkColorScheme(LocalContext.current)` / `dynamicLightColorScheme(LocalContext.current)`.
- Custom themes: `colorScheme.copy(primary = ...)` for branding.

### 2.5 Navigation
- **Navigation Compose** with **type-safe routes** (Nav 2.8+):
  ```kotlin
  @Serializable data object Home
  @Serializable data class Profile(val userId: String)
  NavHost(navController, startDestination = Home) {
      composable<Home> { HomeScreen(onProfile = { navController.navigate(Profile(it)) }) }
      composable<Profile> { ProfileScreen() }
  }
  ```
- **Voyager** as an alternative with KSP-based screen models.
- **Compose Destinations** for annotation-based navigation.

### 2.6 Performance
- **Baseline Profiles** for cold-start and runtime performance.
- `@Stable` annotation for non-data classes used in recomposition.
- `key()` blocks to break recomposition groups.
- Avoid `Modifier` allocation in tight loops.
- `LazyListState` remembered via `rememberLazyListState()`.
- **Compose Compiler Reports** (`-P plugin:androidx.compose.compiler.plugins.kotlin:reportsDestination=...`) to find non-skippable Composables.

---

## 3. Architecture Components

### 3.1 ViewModel
- Constructor-injected with Hilt: `@HiltViewModel class ProfileViewModel @Inject constructor(...) : ViewModel()`.
- Expose immutable `StateFlow<UiState>`.
- Expose functions: `onEvent(event: UiEvent)`.
- Use `viewModelScope.launch { ... }` for coroutines.
- Use `SavedStateHandle` to read/write navigation arguments.
- `viewModelScope` is automatically cancelled in `onCleared()`.

### 3.2 LiveData vs StateFlow
- Prefer **`StateFlow`** for new code (Compose-native, coroutine-native, type-safe).
- `LiveData` is legacy; only use it in Java codebases or where transformation observers are required.

### 3.3 Paging 3
- `PagingSource` and `RemoteMediator` for paginated network + database.
- `Pager` + `PagingData` Flow + `collectAsLazyPagingItems()` in Compose.

### 3.4 WorkManager
- For deferrable, guaranteed background work (sync, uploads, periodic cleanup).
- `CoroutineWorker` is the modern base class.
- Constraint-aware scheduling: `setRequiresBatteryNotLow(true)`, `setRequiresNetworkType(NetworkType.UNMETERED)`, `setRequiresCharging(true)`.
- **OneTimeWorkRequest** or **PeriodicWorkRequest** with `WorkInfo` Flow for status.

### 3.5 DataStore
- **Preferences DataStore** for key-value pairs (replaces `SharedPreferences`).
- **Proto DataStore** for type-safe schema (replaces Room for simple data).
- `Flow<Preferences>` API for reactive reads.
- Atomic `edit { prefs -> prefs[KEY] = value }` writes.

### 3.6 Room
- `@Entity`, `@Dao`, `@Database`, `@TypeConverter`, `@Migration`, `@Relation`, `@Embedded`.
- **Coroutines & Flow:** `suspend fun ...` and `fun ...: Flow<List<X>>` in DAOs.
- **Migrations:** `MIGRATION_1_2 = object : Migration(1, 2) { ... }` + `addMigrations(MIGRATION_1_2, ...)`.
- **Auto-migrations** for simple schema changes (Room 2.6+).
- **KSP** (NOT kapt) for the Room compiler.

---

## 4. Dependency Injection (Hilt)

### 4.1 Setup
- `@HiltAndroidApp` on the `Application` class.
- `@AndroidEntryPoint` on the `Activity` / `Fragment` / `View` / `Service` / `BroadcastReceiver`.
- `@HiltViewModel` on `ViewModel`s.
- `@Inject` constructors for injection; `@Module @InstallIn(...)` for object graphs.

### 4.2 Component Scopes
- `SingletonComponent` — app-wide singletons.
- `ActivityRetainedComponent` — across config changes.
- `ViewModelComponent` — per ViewModel.
- `ActivityComponent` — per Activity.
- `FragmentComponent` / `ViewComponent` / `ViewWithFragmentComponent` — more granular.

### 4.3 Hilt + Compose
- `hiltViewModel()` Composable to obtain a `ViewModel` scoped to the navigation entry.
- `EntryPointAccessors.fromApplication(context, MyEntryPoint::class.java)` for non-injectable access.

---

## 5. Networking

### 5.1 Retrofit
- `@GET`, `@POST`, `@PUT`, `@DELETE`, `@PATCH` annotations.
- **Type-safe with `@Url` or path parameters.**
- **Suspend functions** for one-shot requests.
- **`Response<T>`** for raw HTTP responses; **`T`** for body-only (throws on non-2xx).
- **Sealed `NetworkResult<T>`** wrapper for type-safe error handling.

### 5.2 OkHttp
- **Interceptors:** `addInterceptor` for app-level (e.g., logging), `addNetworkInterceptor` for wire-level.
- **`HttpLoggingInterceptor`** — only in debug builds.
- **Authentication:** `Authenticator` for 401-refresh-token flow.
- **Certificate pinning:** `CertificatePinner.Builder().add(...)`.
- **Timeouts:** `connectTimeout`, `readTimeout`, `writeTimeout`, `callTimeout`.
- **Compression:** enable Gzip via `Request.Builder().addHeader("Accept-Encoding", "gzip")` and the `GzipRequestInterceptor`.

### 5.3 JSON Serialization
- **kotlinx.serialization** — preferred for new code (Kotlin-first, no reflection, fast).
- **Moshi** — with `moshi-kotlin-codegen` (KSP) for reflection-free parsing.
- **Gson** — legacy, avoid in new code.

### 5.4 GraphQL (Optional)
- **Apollo Kotlin** — type-safe GraphQL client with code generation.

---

## 6. Concurrency

### 6.1 Coroutines
- `Dispatchers.Main` — UI updates.
- `Dispatchers.IO` — blocking I/O (network, disk).
- `Dispatchers.Default` — CPU-bound work.
- `withContext(dispatcher) { ... }` to switch.
- `coroutineScope { ... }` for structured concurrency.
- `supervisorScope { ... }` to allow children to fail independently.
- `Job`, `Deferred`, `async`/`await`.

### 6.2 Flow Operators
- **Upstream:** `map`, `mapNotNull`, `filter`, `filterNotNull`, `combine`, `merge`, `zip`, `flatMapConcat`, `flatMapMerge`, `flatMapLatest`, `debounce`, `distinctUntilChanged`, `sample`, `conflate`, `buffer`.
- **Downstream:** `collect`, `collectLatest`, `first`, `firstOrNull`, `last`, `toList`, `launchIn`.
- **State conversion:** `stateIn(scope, SharingStarted.Eagerly, initialValue)`, `stateIn(scope, SharingStarted.WhileSubscribed(5_000), initialValue)`, `stateIn(scope, SharingStarted.Lazily, initialValue)`.
- **Side effects:** `onEach`, `onStart`, `onCompletion`, `onEmpty`, `catch { ... }`, `retry { ... }`, `retryWhen { ... }`.
- **Error handling:** `catch` operator to map errors to UI state. NEVER use `try/catch` inside a Composable.

### 6.3 Channels
- `Channel<T>(capacity)` — rendezvous (0), CONFLATED, UNLIMITED, BUFFERED, RENDEZVOUS.
- `produce { ... }` — coroutine builder that produces to a ReceiveChannel.
- `consumeEach { ... }` — deprecated. Use `for (item in channel) { ... }` or `channel.consume { ... }`.

---

## 7. Testing Ecosystem

### 7.1 Unit Testing (JVM)
- **JUnit 4** (legacy, still dominant in Android ecosystem) or **JUnit 5** (with the **`android-junit5`** plugin, growing adoption).
- **Kotlin test** (`kotlin-test`) for Kotlin-friendly assertions.
- **Turbine** — `app.cash.turbine:turbine` for `Flow` testing.
- **MockK** — Kotlin-native mocking (`every { ... } returns ...`, `coEvery { ... } coReturns ...` for suspend).
- **Mockito-Kotlin** — alternative to MockK, less Kotlin-idiomatic.
- **Robolectric** — runs Android framework classes on the JVM (`RuntimeEnvironment.getApplication()`, `Shadows.shadowOf(...)`).
- **Paparazzi** — `app.cash.paparazzi:paparazzi` for Composable screenshot tests, no emulator.
- **Roborazzi** — Robolectric-based screenshot tests, supports `@Preview` Composables.
- **Kotest** — alternative to JUnit, with property-based testing (`kotest-property`).
- **Truth** — fluent assertions (`assertThat(x).isEqualTo(y)`).
- **MockK** for suspending, Flow-returning functions: `coEvery { repo.getUser() } returns flowOf(user)`.
- **MainDispatcherRule** to swap `Dispatchers.Main` to `UnconfinedTestDispatcher()`.

### 7.2 Instrumented Testing (Device/Emulator)
- **AndroidX Test** — `androidx.test:core`, `androidx.test:runner`, `androidx.test:rules`, `androidx.test.ext:junit`.
- **Espresso** — UI tests for legacy XML views.
- **Compose UI Test** — `androidx.compose.ui:ui-test-junit4` for Compose: `createComposeRule()`, `onNodeWithText(...)`, `performClick()`, `assertIsDisplayed()`.
- **Hilt Testing** — `@HiltAndroidTest` + `HiltAndroidRule` + `HiltTestApplication` for replacing the Application in tests.
- **Room Testing** — `Room.inMemoryDatabaseBuilder` for in-memory DB in tests.
- **DataStore Testing** — use a `tmpdir` for the file and inject a custom `PreferenceDataStoreFactory.create()`.
- **WorkManager Testing** — `WorkManagerTestInitHelper` to initialize the test driver.

### 7.3 Macrobenchmark (Performance)
- **`androidx.benchmark:benchmark-macro-junit4`** — cold-start, frame timing, scroll.
- **Baseline Profile generation** — `androidx.benchmark:benchmark-baseline-profile-gradle-plugin`.
- **`androidx.profileinstaller`** — install the baseline profile on the device for immediate performance gains.

### 7.4 Required Test Discipline
- All ViewModels have unit tests with Turbine + `runTest`.
- All public Composables have at least one `@Preview` AND one `createComposeRule` UI test for the primary happy path.
- All Repositories have unit tests with fake DAOs and fake API services.
- All UseCases have unit tests covering success, error, and edge cases.
- All Workers have unit tests with `TestListenableWorkerBuilder` or via Hilt testing.
- Test names use backtick-descriptive style: `` `loading user emits Loading then Success`() ``.

---

## 8. Build, Lint, and Static Analysis

### 8.1 Linters and Formatters (run all in CI)
- **`ktlint`** — Kotlin linter and formatter. Config: `~/.ktlint` or `.editorconfig`.
- **`detekt`** — Kotlin static analysis. Config: `config/detekt.yml`. 200+ rules.
- **`Android Lint`** — built-in Android static analysis. Config in `lint.xml`.
- **`spotless`** — multi-language formatter (Kotlin via ktlint, Gradle via palantir-java-format).
- **Baseline profiles** for Compose startup performance.

### 8.2 Recommended `detekt.yml` Rules (Minimum)
- `style` group: all rules on (MagicNumber, MaxLineLength, ReturnCount, etc.).
- `empty-blocks`: all rules on.
- `exceptions`: `TooGenericExceptionCaught`, `ThrowingExceptionsWithoutMessageOrCause`.
- `performance`: `SpreadOperator`, `UnnecessaryTemporaryInstantiation`.
- `potential-bugs`: `AvoidReferentialEquality`, `CastToNullableType`, `Deprecation`, `ImplicitDefaultLocale`, `MissingPackageDeclaration`, `UnreachableCatchBlock`, `UnreachableCode`, `UnsafeCallOnNullableType`, `UnsafeCast`.
- `coroutines`: `GlobalCoroutineUsage`, `RedundantSuspendModifier`, `SleepInsteadOfDelay`, `SuspendFunWithCoroutineScopeReceiver`, `SuspendFunWithFlowReturnType`.
- Custom: `ForbiddenMethodCall` for `!!`, `printStackTrace`, `System.out.println`, `GlobalScope.launch`.

### 8.3 Build Tooling
- **Gradle Version Catalog** for all versions.
- **Configuration cache** enabled in `gradle.properties`.
- **Build cache** enabled (local and remote).
- **Composite build** with `build-logic/convention` for shared module-type plugins (`android-app`, `android-library`, `android-feature`, `android-compose`, `android-hilt`).
- **R8/ProGuard** rules committed in `app/proguard-rules.pro`.
- **Baseline profile** generated by `:benchmarks` module and installed at app startup.

---

## 9. Security

- **`androidx.security:security-crypto`** — `EncryptedSharedPreferences` and `EncryptedFile` (backed by Android Keystore + Tink).
- **Tink** (`com.google.crypto.tink:tink-android`) for new encryption code (favored over `security-crypto` going forward).
- **OkHttp `CertificatePinner`** for certificate pinning.
- **Network Security Configuration** to disable cleartext and pin certs.
- **WebView hardening:** disable JS, disable file access, set `setAllowFileAccessFromFileURLs(false)`, `setAllowUniversalAccessFromFileURLs(false)`.
- **Android Keystore** for key material. NEVER export raw keys.
- **BiometricPrompt** for user authentication: `androidx.biometric:biometric`.
- **Play Integrity API** to verify the app binary hasn't been tampered with.
- **SafetyNet / Play Integrity Attestation** for server-side device verification.
- **`android:exported`** explicitly set on every component (Activity, Service, BroadcastReceiver, ContentProvider). Default to `false`.
- **Permissions:** `<uses-permission>` is the **minimum** required. Use `android:required="false"` on optional features.
- **Code obfuscation:** R8 minification + resource shrinking + ProGuard rules for reflection-using libraries.
- **`android:allowBackup`** set explicitly. Use `android:dataExtractionRules` (Android 12+) for granular control.
- **Rooted-device detection** (optional, when content requires it) via `RootBeer` or custom checks.

---

## 10. Resilience Patterns

- **Retry with backoff:** use `kotlinx.coroutines.retry` or `Flow.retryWhen` with exponential backoff and jitter. NEVER retry non-idempotent operations blindly.
- **Timeouts:** wrap every network call in `withTimeout(N) { ... }`.
- **Circuit breaker:** use Resilience4j-Java (via Kotlin interop) or a custom state machine with `Mutex`.
- **Offline-first:** Repository returns from local DB; network is the source of truth that updates the DB.
- **Connectivity awareness:** `ConnectivityManager.NetworkCallback` to pause/resume sync.
- **Graceful degradation:** show a cached/empty state when network or DB fails. NEVER crash the app on transient failures.

---

## 11. Recommended GitHub Actions

You are authorized to use and configure the following proven actions:
- `actions/checkout@v4` — repository checkout.
- `actions/setup-java@v4` — JDK 17 (Temurin) for Gradle.
- `gradle/actions/setup-gradle@v4` — Gradle wrapper with build cache.
- `actions/cache@v4` — additional caching.
- `actions/upload-artifact@v4` — APK / AAB / mapping.txt.
- `actions/download-artifact@v4` — fetch prior build outputs.
- `r0adkll/upload-google-play@v1` — publish to Google Play (internal/alpha/beta/production tracks).
- `firebase/test-lab-action` — run instrumented tests on Firebase Test Lab.
- `google-github-actions/auth@v2` + `google-github-actions/setup-gcloud@v2` — GCP auth for Play Store API.
- `aquasecurity/trivy-action@master` — image & filesystem scan (for CI Docker image).
- `github/codeql-action@v3` — code scanning.
- `sonarsource/sonarqube-scan-action@master` — SonarQube analysis (optional).
- `dependency-check/Dependency-Check_Action@main` — OWASP dependency scan.
- `softprops/action-gh-release@v2` — GitHub releases.
- `codecov/codecov-action@v4` — coverage reporting.
- `reviewdog/action-actionlint@v1` — workflow linting.
- `docker/setup-buildx-action@v3`, `docker/login-action@v3`, `docker/build-push-action@v5` — for the CI Docker image.

---

## 12. Anti-Patterns to Always Reject

1. **Using `!!` (NotNull assertion) in production code.** Use `?.`, `let`, `requireNotNull(x) { "reason" }`, or sealed result types.
2. **Using `GlobalScope.launch`** in production code. Use `viewModelScope`, `lifecycleScope`, or a custom application scope.
3. **Exposing `MutableStateFlow` publicly from a ViewModel.** Always expose immutable `StateFlow`.
4. **Doing I/O on the main thread** — ever. Use `withContext(Dispatchers.IO) { ... }` for blocking calls.
5. **Hardcoded strings in code or layouts.** All user-facing strings MUST live in `strings.xml`.
6. **Hardcoded secrets, API keys, or tokens in source.** Use `local.properties` + `BuildConfig` for local; secret manager for prod.
7. **Cleartext HTTP traffic** in release builds. Always use HTTPS + cert pinning for sensitive APIs.
8. **Java-style Activity lifecycle hacks** (e.g., manual `findViewById`, `AsyncTask`, `Handler.postDelayed` for business logic).
9. **Mixing Material 2 and Material 3** in the same project. Pick Material 3 and stay there.
10. **XML layouts for new screens.** Use Jetpack Compose.
11. **Multi-Activity architecture** in new code. Use single-Activity + Compose Navigation.
12. **Manual dependency wiring** in Composables or Activities. Use Hilt.
13. **SharedPreferences for new code.** Use DataStore (Preferences or Proto).
14. **Raw `Service` for long-running operations.** Use WorkManager.
15. **Printing PII or tokens to logs.** Strip in production. Gate `Timber.plant(DebugTree())` on `BuildConfig.DEBUG`.
16. **Skipping `@Preview` on Composables.** Every public Composable must have a preview.
17. **Skipping accessibility.** Content descriptions, touch targets, dynamic type, RTL support are non-negotiable.
18. **Ignoring configuration changes.** State MUST survive rotation. Use `rememberSaveable`, `SavedStateHandle`, or `StateFlow` in the ViewModel.
19. **Committing the release keystore** to the repo. The `keystore/` directory is `.gitignore`d.
20. **Using a Docker image to "ship" an Android app to users.** Android apps are APKs/AABs. Docker is for the CI build environment only.
