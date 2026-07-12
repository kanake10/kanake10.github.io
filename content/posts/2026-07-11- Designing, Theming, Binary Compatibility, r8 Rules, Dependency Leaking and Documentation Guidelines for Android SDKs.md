---
title: "Android SDK: Designing Theming, Binary Compatibility, R8 Rules, Dependency Leaking, and Documentation"
date: 2026-07-11
tags: ["android", "sdk", "kotlin", "jetpack-compose", "gradle", "metalava", "r8"]
categories: ["Android", "SDK"]
draft: false
description: "designing a Material3-independent theming system, keeping the public API binary compatible, validating changes with Metalava, configuring R8 consumer rules and avoiding dependency leaks in a library meant for other developers' apps."
---

Building an Android app and building an Android SDK are very different experiences.

Apps have complete control over their codebase, dependencies and release cycle. SDKs, however, become part of someone else's application. Every public API, dependency and configuration decision can affect developers integrating the library.

This article tries to share some of the lessons learnt during the development of the SDK.

---

## Public APIs Are Contracts

One of the first lessons was understanding that every public declaration becomes part of the SDK's contract.

Consider the following API:

```kotlin
class TranslateClient {
    fun translate(text: String): TranslationResult
}
```

Once developers begin using this method, changing it can become a breaking change:

```kotlin
class TranslateClient {
    fun translate(
        text: String,
        targetLanguage: String
    ): TranslationResult
}
```

Although this appears to be a simple improvement, applications using the previous version will fail to compile after upgrading.

When building an SDK, every public API should be treated as a long-term commitment.

---

## Designing Small Public APIs

Early during development, it was tempting to expose internal classes:

```kotlin
class TranslationRepository
class TranslationApi
class TranslationMapper
class TranslationCache
```

However, every public class increases maintenance costs.

Instead, the Translate SDK exposes only the APIs consumers need:

```kotlin
Translate.initialize(...)
Translate.translate(...)
```

Everything else remains internal.

Keeping the public surface small provides several advantages:

- Easier documentation.
- Fewer breaking changes.
- Better maintainability.
- Simpler onboarding for developers.

---

## Designing Themeable UI Components

The SDK also ships an optional Compose UI module, and this is where a different kind of API design problem showed up: **visual contracts, not just functional ones.**

The first version of the theme leaned directly on Material3:

```kotlin
@Composable
fun TranslateTheme(
    darkTheme: Boolean = isSystemInDarkTheme(),
    content: @Composable () -> Unit
) {
    MaterialTheme(
        colorScheme = if (darkTheme) DarkColorScheme else LightColorScheme,
        typography = Typography,
        content = content
    )
}
```

This works fine as long as every consumer uses Material3 and is happy with the SDK's exact palette. Neither assumption holds for an SDK meant to drop into arbitrary host apps. Some consumers use Material3 with their own brand colors. Some don't use Material3 at all. All of them still needed the components to render sensibly with zero configuration.

**The fix was to stop reading `MaterialTheme` directly inside SDK components and introduce a small token system instead** — the same pattern Material3 itself uses internally, via `CompositionLocal`.

### Semantic tokens instead of a Material dependency

```kotlin
@Immutable
data class TranslateColors(
    val primary: Color,
    val onPrimary: Color,
    val background: Color,
    val onBackground: Color,
    val surface: Color,
    val onSurface: Color,
    val surfaceVariant: Color,
    val onSurfaceVariant: Color,
    val error: Color,
    val onError: Color,
)

@Immutable
data class TranslateTypography(
    val body: TextStyle,
    val title: TextStyle,
    val label: TextStyle,
)
```

### CompositionLocals with safe fallback defaults

The key design constraint: a consumer who forgets to wrap their content in the SDK's theme should still get a sensible result and not a crash.

```kotlin
val LocalTranslateColors = staticCompositionLocalOf { lightTranslateColors() }
val LocalTranslateTypography = staticCompositionLocalOf { defaultTranslateTypography() }

object TranslateThemeTokens {
    val colors: TranslateColors
        @Composable get() = LocalTranslateColors.current

    val typography: TranslateTypography
        @Composable get() = LocalTranslateTypography.current
}
```

Every internal composable now reads `TranslateThemeTokens.colors.x` instead of `MaterialTheme.colorScheme.x`.

### One theme entry point, three ways to use it

```kotlin
@Composable
fun TranslateTheme(
    darkTheme: Boolean = isSystemInDarkTheme(),
    colors: TranslateColors = if (darkTheme) darkTranslateColors() else lightTranslateColors(),
    typography: TranslateTypography = defaultTranslateTypography(),
    content: @Composable () -> Unit
) {
    CompositionLocalProvider(
        LocalTranslateColors provides colors,
        LocalTranslateTypography provides typography,
    ) {
        // Still wrapped in MaterialTheme because Button, OutlinedTextField,
        // and ExposedDropdownMenuBox are Material3 composables that need a
        // ColorScheme in context for things like ripple and default shapes.
        MaterialTheme(
            colorScheme = if (darkTheme) DarkColorScheme else LightColorScheme,
            typography = Typography,
            content = content,
        )
    }
}
```

This single signature ended up covering every case that came up:

- **Default look, zero configuration** — `TranslateTheme { TranslationScreen(...) }`.
- **Match an existing Material3 app theme** — the consumer maps their own `MaterialTheme.colorScheme` values into `TranslateColors` at the call site, instead of the SDK guessing.
- **No Material3 at all / a fully custom design system** — the consumer passes arbitrary `Color` and `TextStyle` values with no Material dependency required for their own theming decisions.
- **Nothing passed at all** — components still resolve sensible defaults via the `CompositionLocal` fallback.

###  mistakes worth naming

Both surfaced only once the theme was actually wired up end-to-end, which is itself a small lesson: theming bugs tend to hide until a real toggle or a real snapshot test exercises them.

**Mismatched fallback schemes.** An earlier draft computed the default `colors` from `isSystemInDarkTheme()` in one place, and separately picked the internal `MaterialTheme` `colorScheme` from `isSystemInDarkTheme()` again, rather than from a single shared flag. In most cases these two calls agree — but the moment a consumer (or a Paparazzi snapshot test) passed `colors` explicitly, the internal Material scheme could still land on the opposite mode, since it wasn't derived from the same source. The fix was routing both from a single `darkTheme: Boolean` parameter, so the token colors and the Material fallback scheme can never disagree.

### Widgets that don't read your tokens automatically

One non-obvious wrinkle: wrapping `Button`, `OutlinedTextField`, and `TextButton` in a theme that provides `TranslateThemeTokens` does **not** automatically make those specific widgets use it. Material3 components resolve their own default colors from the ambient `MaterialTheme.colorScheme`, regardless of what other `CompositionLocal`s are present. Getting them to follow the SDK's own tokens required passing `colors` explicitly at each call site:

```kotlin
Button(
    onClick = onClick,
    colors = ButtonDefaults.buttonColors(
        containerColor = TranslateThemeTokens.colors.primary,
        contentColor = TranslateThemeTokens.colors.onPrimary,
    ),
) { /* ... */ }
```

Without this, a consumer who skips the SDK's theme wrapper entirely would still see some elements correctly falling back to plain black-and-white defaults, while unstyled widget internals — dropdown menu chrome, ripple color, icon buttons — silently inherited whatever Material theme happened to be ambient in their app, or Compose's default purple baseline if there wasn't one. Closing that gap meant explicitly theming each Material primitive individually rather than assuming the wrapper theme would cascade everywhere on its own.

---

## Understanding Binary Compatibility

Source compatibility and binary compatibility are not always the same.

Suppose an application compiles against version 1.0:

```kotlin
client.translate("Hello")
```

If a later SDK version removes or changes that method, applications may encounter runtime errors such as:

```text
java.lang.NoSuchMethodError
```

This happens because the application was compiled against an older version of the API.

Maintaining binary compatibility helps ensure applications continue working after SDK upgrades.

This became one of the reasons we introduced API validation into the Translate SDK.

### Using `@Deprecated` Instead of Removing APIs Outright

Binary compatibility doesn't mean an API can never change — it means changes need a transition path. In such a case, we use the `@Deprecated` annotation to inform consumers of the new implementation while keeping the old one working, rather than removing it outright and breaking anyone still compiled against it.

`@Deprecated` supports a `replaceWith` argument, which powers Android Studio's automatic "replace with..." quick fix so consumers can migrate with one click instead of digging through docs. It also supports a `level`, which controls how strongly the deprecation is enforced: a `WARNING` level simply flags the old API without breaking builds, an `ERROR` level stops new source code from using it while still preserving the compiled symbol for older consumers, and a `HIDDEN` level removes it from autocomplete and further compilation entirely while keeping it present in the binary. Moving an API through these levels over a few minor releases — rather than deleting it in one step — gives consumers time to migrate before it's finally removed in a major version bump.

This staged approach is also what keeps `metalavaCheckCompatibility` passing during a deprecation cycle: the signature file still lists the old method, now annotated `@Deprecated`, so Metalava doesn't treat it as a breaking removal until it's actually deleted from the API surface.

---

## Using Metalava for API Validation

To track changes to the public API surface, the Translate SDK uses Metalava.

Metalava generates a signature file representing all public APIs:

```text
package com.kanake.translate {

  public final class TranslateClient {
    method public TranslationResult translate(String text);
  }

}
```

This file acts as a snapshot of the SDK's public contract.

Generating the API file:

```bash
./gradlew metalavaGenerateSignature
```

Checking for compatibility:

```bash
./gradlew metalavaCheckCompatibility
```

If a public method is removed or changed, Metalava reports the issue before the SDK is released.

For example:

```text
error: Removed method:
    TranslationResult translate(String text)
```

This helps prevent accidental breaking changes.

---

## Automating API Checks in CI

Running API checks locally is useful, but developers can forget to run them.

To solve this, API validation was integrated into GitHub Actions:

```yaml
name: API Validation

on:
  pull_request:
  push:
    branches:
      - main

jobs:
  api-check:
    runs-on: ubuntu-latest

    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-java@v4
        with:
          distribution: temurin
          java-version: 21

      - uses: gradle/actions/setup-gradle@v4

      - name: Run API checks
        run: ./gradlew metalavaCheckCompatibility
```

Now every pull request automatically validates the SDK's public API before merging.

---

## R8 and ProGuard Considerations

Most Android applications enable code shrinking using R8 nowadays.

Without proper configuration, important classes or metadata may be removed.

For example:

```kotlin
@Serializable
data class TranslationRequest(
    val text: String
)
```

If required metadata is removed, runtime failures can occur.

The Translate SDK ships with consumer ProGuard rules:

```text
-keep class com.kanake.translate.** { *; }
-keepattributes *Annotation*
```

These rules help applications safely shrink and obfuscate their code while preserving the SDK's required components.

Providing consumer rules reduces integration issues for developers.

---

## Avoid Leaking Dependencies

Another important lesson was avoiding implementation details in public APIs.

For example:

```kotlin
fun translate(): retrofit2.Response<TranslationResponse>
```

This forces every consumer to depend on Retrofit.

Instead:

```kotlin
fun translate(): TranslationResult
```

The networking implementation remains internal.

This allows the SDK to change:

- Retrofit to Ktor.
- Moshi to Kotlin Serialization.
- Networking implementations.
- Caching strategies.

Applications using the SDK remain unaffected.

The same principle applies to the UI module's theming design above: consumers depend on `TranslateColors` and `TranslateTypography`, not on Material3's `ColorScheme` shape directly, so the SDK is free to change its internal Material dependency without breaking anyone's theme mapping.

---

## Versioning Matters

Not every release should be treated the same.

A simple approach is:

- Patch version: bug fixes.
- Minor version: new APIs.
- Major version: breaking changes.

For example:

```text
1.0.0
1.1.0
2.0.0
```

Clear versioning helps developers understand the impact of upgrading.

---

## Documentation Is Part of the API

Even a well-designed SDK can be difficult to adopt without documentation.

Important documentation includes:

- Quick start guides.
- Integration examples.
- API references.
- Migration guides.
- ProGuard instructions.
- Release notes.

Good documentation improves the developer experience and reduces support requests.

---