---
name: flutter-theming
description: Use when creating, migrating or auditing a Material 3 theme in a Flutter app - assembling ColorScheme and ThemeData, theming a specific component, replacing a hardcoded colour with a theme token, or reviewing visual consistency. Triggers on any mention of colour, theme, dark mode, or "this looks off-brand" on a Flutter screen.
---

You are working on theming for a Flutter app. The patterns below assume a shared `design_system/` package holding cross-app tokens (`DSColors`, `DSSpacing`, `DSRadius`, `DSTypography`), with each app free to keep its own `lib/theme/` for app-specific overrides.

Always prefer importing a token from the design system over duplicating the value in an app.

---

## Workflow: create

Scaffold a complete theme for the target app, taken
from the request or the skill argument. **If it is not clear which app, ask before writing
anything:** each one has its own theme directory and overwriting the wrong app is silent.

### Theme Directory Structure

```
<app>/lib/shared/theme/
├── app_theme.dart           ← ThemeData factory (light/dark)
├── app_colors.dart          ← App-specific color overrides (imports DSColors)
└── app_text_styles.dart     ← App-specific text style overrides (imports DSTypography)
```

### app_colors.dart

```dart
import 'package:flutter/material.dart';
import 'package:design_system/design_system.dart';

/// App-specific color aliases. Delegates to DSColors for shared tokens
/// and defines overrides only when this app diverges from the design system.
abstract final class AppColors {
  // Re-export shared tokens for convenience
  static const Color primary = DSColors.brandPrimary;
  static const Color secondary = DSColors.brandSecondary;
  static const Color error = DSColors.error;
  static const Color success = DSColors.success;
  static const Color warning = DSColors.warning;

  // App-specific overrides
  static const Color scaffoldBackground = Color(0xFFF8F9FA);
  static const Color cardBackground = DSColors.neutral0;
  static const Color divider = DSColors.neutral200;
}
```

### app_text_styles.dart

```dart
import 'package:flutter/material.dart';
import 'package:design_system/design_system.dart';

/// App-specific text style overrides. Use DSTypography directly when possible.
abstract final class AppTextStyles {
  static TextStyle get pageTitle => DSTypography.headingH4;
  static TextStyle get sectionTitle => DSTypography.headingH5;
  static TextStyle get body => DSTypography.body14Regular;
  static TextStyle get caption => DSTypography.caption;

  // App-specific additions only
  static TextStyle get price => DSTypography.price;
}
```

### app_theme.dart (Material 3)

```dart
import 'package:flutter/material.dart';
import 'package:design_system/design_system.dart';
import 'app_colors.dart';

class AppTheme {
  static ThemeData light() {
    final colorScheme = ColorScheme.fromSeed(
      seedColor: AppColors.primary,
      brightness: Brightness.light,
    );

    return ThemeData(
      useMaterial3: true,
      colorScheme: colorScheme,
      fontFamily: 'Roboto',
      scaffoldBackgroundColor: AppColors.scaffoldBackground,

      // --- Component Themes (use *ThemeData suffix) ---
      appBarTheme: AppBarThemeData(
        backgroundColor: colorScheme.surface,
        foregroundColor: colorScheme.onSurface,
        elevation: 0,
        centerTitle: false,
      ),
      cardTheme: CardThemeData(
        elevation: 0,
        shape: RoundedRectangleBorder(
          borderRadius: DSRadius.lgBorderRadius,
          side: BorderSide(color: AppColors.divider),
        ),
        color: AppColors.cardBackground,
      ),
      filledButtonTheme: FilledButtonThemeData(
        style: FilledButton.styleFrom(
          backgroundColor: AppColors.primary,
          foregroundColor: DSColors.neutral0,
          shape: RoundedRectangleBorder(
            borderRadius: DSRadius.mdBorderRadius,
          ),
          minimumSize: const Size(double.infinity, 48),
          textStyle: const TextStyle(
            fontWeight: FontWeight.w600,
            fontSize: 14,
          ),
        ),
      ),
      elevatedButtonTheme: ElevatedButtonThemeData(
        style: ElevatedButton.styleFrom(
          backgroundColor: AppColors.primary,
          foregroundColor: DSColors.neutral0,
          shape: RoundedRectangleBorder(
            borderRadius: DSRadius.mdBorderRadius,
          ),
          minimumSize: const Size(double.infinity, 48),
        ),
      ),
      outlinedButtonTheme: OutlinedButtonThemeData(
        style: OutlinedButton.styleFrom(
          foregroundColor: AppColors.primary,
          side: BorderSide(color: AppColors.primary),
          shape: RoundedRectangleBorder(
            borderRadius: DSRadius.mdBorderRadius,
          ),
          minimumSize: const Size(double.infinity, 48),
        ),
      ),
      textButtonTheme: TextButtonThemeData(
        style: TextButton.styleFrom(
          foregroundColor: AppColors.primary,
        ),
      ),
      inputDecorationTheme: InputDecorationTheme(
        border: OutlineInputBorder(
          borderRadius: DSRadius.mdBorderRadius,
        ),
        contentPadding: const EdgeInsets.symmetric(
          horizontal: DSSpacing.lg,
          vertical: DSSpacing.md,
        ),
      ),
      navigationBarTheme: NavigationBarThemeData(
        indicatorColor: colorScheme.secondaryContainer,
        labelTextStyle: WidgetStateProperty.resolveWith((states) {
          if (states.contains(WidgetState.selected)) {
            return DSTypography.label.copyWith(color: AppColors.primary);
          }
          return DSTypography.label;
        }),
      ),
      dividerTheme: DividerThemeData(
        color: AppColors.divider,
        thickness: 1,
        space: 0,
      ),

      // --- Typography ---
      textTheme: const TextTheme(
        displayLarge: DSTypography.headingH4,
        headlineMedium: DSTypography.headingH5,
        titleLarge: DSTypography.headingH6,
        bodyLarge: DSTypography.body16Regular,
        bodyMedium: DSTypography.body14Regular,
        bodySmall: DSTypography.body12Regular,
        labelSmall: DSTypography.label,
      ),
    );
  }

  static ThemeData dark() {
    final colorScheme = ColorScheme.fromSeed(
      seedColor: AppColors.primary,
      brightness: Brightness.dark,
    );

    return ThemeData(
      useMaterial3: true,
      colorScheme: colorScheme,
      fontFamily: 'Roboto',
      // Mirror the light theme structure but with dark-appropriate values.
      // Override specific component themes as needed.
    );
  }
}
```

### Applying the Theme

In `app_widget.dart`:

```dart
MaterialApp(
  theme: AppTheme.light(),
  darkTheme: AppTheme.dark(),
  themeMode: ThemeMode.system,
  // ...
)
```

---

## Workflow: migrate

Migrate an existing app to Material 3 theming. Read the app's current `app_widget.dart` and any existing theme files first.

### Migration Checklist

1. **Enable Material 3**: Add `useMaterial3: true` to `ThemeData`
2. **Replace `primarySwatch`** with `colorScheme: ColorScheme.fromSeed(seedColor: ...)`
3. **Apply deprecated replacements**:

| Deprecated | Replacement |
|---|---|
| `accentColor` | `colorScheme.secondary` |
| `primaryColor` (in ThemeData) | `colorScheme.primary` |
| `AppBarTheme.color` | `AppBarThemeData.backgroundColor` |
| `AppBarTheme.brightness` | `AppBarThemeData.systemOverlayStyle` |
| `BottomNavigationBar` | `NavigationBar` |
| `FlatButton` | `TextButton` |
| `RaisedButton` | `ElevatedButton` |
| `ButtonTheme` | `TextButtonThemeData` / `ElevatedButtonThemeData` / `OutlinedButtonThemeData` |
| `MaterialStateProperty` | `WidgetStateProperty` |
| `MaterialState` | `WidgetState` |

4. **Normalize component theme class names** (use `*ThemeData` suffix):

| Old Name | Correct Name |
|---|---|
| `CardTheme(...)` constructor in ThemeData | `CardThemeData(...)` |
| `AppBarTheme(...)` | `AppBarThemeData(...)` |
| `IconTheme(...)` in ThemeData | `IconThemeData(...)` |
| `TabBarTheme(...)` | `TabBarThemeData(...)` |
| `DialogTheme(...)` | `DialogThemeData(...)` |
| `BottomSheetTheme(...)` | `BottomSheetThemeData(...)` |
| `TooltipTheme(...)` | `TooltipThemeData(...)` |

5. **Replace hardcoded colors** with `DSColors` or `Theme.of(context).colorScheme` references
6. **Update typography** to Material 3 scale:

| Old (Material 2) | New (Material 3) |
|---|---|
| `headline1` | `displayLarge` |
| `headline2` | `displayMedium` |
| `headline3` | `displaySmall` |
| `headline4` | `headlineMedium` |
| `headline5` | `headlineSmall` |
| `headline6` | `titleLarge` |
| `subtitle1` | `titleMedium` |
| `subtitle2` | `titleSmall` |
| `bodyText1` | `bodyLarge` |
| `bodyText2` | `bodyMedium` |
| `caption` | `bodySmall` |
| `button` | `labelLarge` |
| `overline` | `labelSmall` |

### Button Styling After Migration

```dart
// ElevatedButton with full styling
ElevatedButton(
  style: ElevatedButton.styleFrom(
    backgroundColor: AppColors.primary,
    foregroundColor: Colors.white,
    disabledBackgroundColor: DSColors.neutral200,
    shape: RoundedRectangleBorder(
      borderRadius: DSRadius.mdBorderRadius,
    ),
    padding: const EdgeInsets.symmetric(
      horizontal: DSSpacing.xl,
      vertical: DSSpacing.md,
    ),
  ),
  onPressed: () {},
  child: const Text('Action'),
)

// FilledButton (Material 3 preferred primary action)
FilledButton(
  onPressed: () {},
  child: const Text('Primary Action'),
)

// FilledButton.tonal (secondary emphasis)
FilledButton.tonal(
  onPressed: () {},
  child: const Text('Secondary Action'),
)
```

### State-Dependent Styling

```dart
// Use WidgetStateProperty (NOT MaterialStateProperty) for state-dependent styles
ButtonStyle(
  backgroundColor: WidgetStateProperty.resolveWith((states) {
    if (states.contains(WidgetState.disabled)) return DSColors.neutral200;
    if (states.contains(WidgetState.pressed)) return AppColors.primary.withValues(alpha: 0.8);
    if (states.contains(WidgetState.hovered)) return AppColors.primary.withValues(alpha: 0.9);
    return AppColors.primary;
  }),
  foregroundColor: WidgetStateProperty.resolveWith((states) {
    if (states.contains(WidgetState.disabled)) return DSColors.neutral500;
    return DSColors.neutral0;
  }),
  overlayColor: WidgetStateProperty.resolveWith((states) {
    if (states.contains(WidgetState.pressed)) return Colors.white.withValues(alpha: 0.12);
    if (states.contains(WidgetState.hovered)) return Colors.white.withValues(alpha: 0.08);
    return null;
  }),
)

// Shorthand for simple cases
WidgetStateProperty.all(DSColors.neutral0)
```

---

## Workflow: audit

Audit theme consistency across apps and report issues.

### Audit Steps

1. Read every app's root widget (`app_widget.dart` / `main.dart`)
2. Read all files under `*/lib/theme/`, `*/lib/shared/theme/`
3. Search for hardcoded `Color(0x...)` values in widget files (outside theme files)
4. Search for deprecated APIs: `accentColor`, `primarySwatch`, `FlatButton`, `RaisedButton`, `MaterialState`
5. Check that all apps import from `design_system` instead of duplicating token values
6. Verify `useMaterial3: true` is set in all apps

### Report Format

Produce a markdown report with:

```
## Theme Audit

### Material 3 Status
- <app>: [enabled/disabled]   (one line per app)

### Deprecated API Usage
- [file:line] — description of deprecated usage and fix

### Hardcoded Colors (should use DSColors/AppColors)
- [file:line] — Color(0xFF...) → suggested token

### Token Duplication
- [file] duplicates [token] already in design_system

### Recommendations
1. ...
```

---

## Instructions

1. Always read the app's existing `app_widget.dart` and theme files before making changes
2. Import shared tokens from `design_system` — never duplicate `DSColors`, `DSSpacing`, `DSRadius`, or `DSTypography` values
3. Use `abstract final class` for all token/color/style classes (cannot be instantiated)
4. All token values must be `static const`
5. Use `ColorScheme.fromSeed()` — never construct `ColorScheme()` manually unless overriding specific slots
6. Use `*ThemeData` suffix for component themes (`CardThemeData`, `AppBarThemeData`, etc.)
7. Use `WidgetStateProperty` and `WidgetState` — never `MaterialStateProperty` or `MaterialState`
8. Use `.withValues(alpha: 0.5)` — never `.withOpacity(0.5)` (deprecated)
9. Prefer `FilledButton` for primary actions over `ElevatedButton` (Material 3 convention)
10. Always set `useMaterial3: true`
11. When the app already has an app-local `<App>DesignTokens`-style class, migrate its values into the `design_system` package or into `app_colors.dart` / `app_text_styles.dart` — do not leave duplicate token files
