---
name: flutter-design-system
description: Use when creating or extending a Flutter design system package - design tokens, type scale, spacing, colour palette, or a new reusable component built from Figma tokens. Triggers on a request for a component shared across apps, or for visual standardisation.
---

You are working on a Flutter design system package - a `design_system/` package at the monorepo root, depended on by every app.

## When argument is "setup" or empty

Scaffold the complete design system package structure:

### Package Structure
```
design_system/
├── pubspec.yaml
└── lib/
    ├── design_system.dart              ← Barrel export file
    └── src/
        ├── tokens/
        │   ├── ds_colors.dart          ← All brand colors
        │   ├── ds_typography.dart      ← TextStyle definitions
        │   ├── ds_spacing.dart         ← Spacing constants
        │   ├── ds_radius.dart          ← Border radius constants
        │   └── ds_shadows.dart         ← BoxShadow constants
        ├── theme/
        │   └── ds_theme.dart           ← ThemeData factory
        └── components/
            ├── buttons/
            │   └── ds_button.dart
            ├── inputs/
            │   └── ds_text_field.dart
            └── typography/
                └── ds_text.dart
```

### Design Tokens Pattern (mapped from Figma tokens)

**ds_colors.dart:**
```dart
abstract final class DSColors {
  // Neutral
  static const Color neutral0   = Color(0xFFFFFFFF);
  static const Color neutral100 = Color(0xFFF5F5F5);
  static const Color neutral200 = Color(0xFFE5E5E5);
  static const Color neutral500 = Color(0xFF737373);
  static const Color neutral900 = Color(0xFF181D27);

  // Brand / Primary
  static const Color brandPrimary   = Color(0xFF7839EE);
  static const Color brandSecondary = Color(0xFF5B21B6);

  // Semantic
  static const Color success = Color(0xFF16A34A);
  static const Color warning = Color(0xFFF59E0B);
  static const Color error   = Color(0xFFDC2626);
  static const Color info    = Color(0xFF2563EB);

  // Text (semantic aliases)
  static const Color textPrimary   = neutral900;
  static const Color textSecondary = neutral500;
  static const Color textInverse   = neutral0;
}
```

**ds_spacing.dart:**
```dart
abstract final class DSSpacing {
  static const double xs  = 4.0;
  static const double sm  = 8.0;
  static const double md  = 12.0;
  static const double lg  = 16.0;
  static const double xl  = 24.0;
  static const double xxl = 32.0;
  static const double xxxl = 48.0;
}
```

**ds_radius.dart:**
```dart
abstract final class DSRadius {
  static const double xs  = 4.0;
  static const double sm  = 6.0;
  static const double md  = 8.0;
  static const double lg  = 12.0;
  static const double xl  = 16.0;
  static const double full = 999.0;

  static BorderRadius get xsBorderRadius => BorderRadius.circular(xs);
  static BorderRadius get smBorderRadius => BorderRadius.circular(sm);
  static BorderRadius get mdBorderRadius => BorderRadius.circular(md);
  static BorderRadius get lgBorderRadius => BorderRadius.circular(lg);
}
```

**ds_typography.dart:**
```dart
import 'package:flutter/material.dart';
import '../tokens/ds_colors.dart';

abstract final class DSTypography {
  static const String _fontFamily = 'Roboto';

  // Headings
  static const TextStyle headingH4 = TextStyle(
    fontFamily: _fontFamily,
    fontSize: 28,
    fontWeight: FontWeight.w700,
    color: DSColors.textPrimary,
    height: 1.25,
  );

  static const TextStyle headingH5 = TextStyle(
    fontFamily: _fontFamily,
    fontSize: 22,
    fontWeight: FontWeight.w600,
    color: DSColors.textPrimary,
    height: 1.3,
  );

  static const TextStyle headingH6 = TextStyle(
    fontFamily: _fontFamily,
    fontSize: 18,
    fontWeight: FontWeight.w600,
    color: DSColors.textPrimary,
    height: 1.4,
  );

  // Body
  static const TextStyle body16Regular = TextStyle(
    fontFamily: _fontFamily,
    fontSize: 16,
    fontWeight: FontWeight.w400,
    color: DSColors.textPrimary,
    height: 1.5,
  );

  static const TextStyle body14Regular = TextStyle(
    fontFamily: _fontFamily,
    fontSize: 14,
    fontWeight: FontWeight.w400,
    color: DSColors.textPrimary,
    height: 1.5,
  );

  static const TextStyle body14Medium = TextStyle(
    fontFamily: _fontFamily,
    fontSize: 14,
    fontWeight: FontWeight.w500,
    color: DSColors.textPrimary,
    height: 1.5,
  );

  static const TextStyle body12Regular = TextStyle(
    fontFamily: _fontFamily,
    fontSize: 12,
    fontWeight: FontWeight.w400,
    color: DSColors.textSecondary,
    height: 1.5,
  );

  // Labels / Captions
  static const TextStyle label = TextStyle(
    fontFamily: _fontFamily,
    fontSize: 12,
    fontWeight: FontWeight.w600,
    letterSpacing: 0.4,
    color: DSColors.textSecondary,
  );

  static const TextStyle caption = TextStyle(
    fontFamily: _fontFamily,
    fontSize: 11,
    fontWeight: FontWeight.w400,
    color: DSColors.textSecondary,
  );

  // Price (domain-specific)
  static const TextStyle price = TextStyle(
    fontFamily: _fontFamily,
    fontSize: 16,
    fontWeight: FontWeight.w700,
    color: DSColors.brandPrimary,
  );
}
```

**ds_theme.dart:**
```dart
import 'package:flutter/material.dart';
import '../tokens/ds_colors.dart';
import '../tokens/ds_typography.dart';
import '../tokens/ds_radius.dart';

class DSTheme {
  static ThemeData light() => ThemeData(
    useMaterial3: true,
    fontFamily: 'Roboto',
    colorScheme: ColorScheme.fromSeed(
      seedColor: DSColors.brandPrimary,
      brightness: Brightness.light,
    ),
    textTheme: const TextTheme(
      headlineMedium: DSTypography.headingH5,
      titleLarge:     DSTypography.headingH6,
      bodyLarge:      DSTypography.body16Regular,
      bodyMedium:     DSTypography.body14Regular,
      bodySmall:      DSTypography.body12Regular,
      labelSmall:     DSTypography.label,
    ),
    filledButtonTheme: FilledButtonThemeData(
      style: FilledButton.styleFrom(
        backgroundColor: DSColors.brandPrimary,
        foregroundColor: DSColors.neutral0,
        shape: RoundedRectangleBorder(
          borderRadius: DSRadius.mdBorderRadius,
        ),
        minimumSize: const Size(double.infinity, 48),
      ),
    ),
    inputDecorationTheme: InputDecorationTheme(
      border: OutlineInputBorder(borderRadius: DSRadius.mdBorderRadius),
      contentPadding: const EdgeInsets.symmetric(
        horizontal: 16,
        vertical: 12,
      ),
    ),
  );
}
```

**Barrel file (lib/design_system.dart):**
```dart
export 'src/tokens/ds_colors.dart';
export 'src/tokens/ds_typography.dart';
export 'src/tokens/ds_spacing.dart';
export 'src/tokens/ds_radius.dart';
export 'src/tokens/ds_shadows.dart';
export 'src/theme/ds_theme.dart';
export 'src/components/buttons/ds_button.dart';
export 'src/components/inputs/ds_text_field.dart';
export 'src/components/typography/ds_text.dart';
```

## When argument is a component name

Create a new component in the appropriate `components/` subfolder.

**Component pattern (e.g., DSButton):**
```dart
import 'package:flutter/material.dart';
import '../../tokens/ds_colors.dart';
import '../../tokens/ds_spacing.dart';
import '../../tokens/ds_radius.dart';

enum DSButtonVariant { filled, outlined, ghost }

class DSButton extends StatelessWidget {
  final String label;
  final VoidCallback? onPressed;
  final DSButtonVariant variant;
  final bool isLoading;
  final IconData? icon;

  const DSButton({
    super.key,
    required this.label,
    required this.onPressed,
    this.variant = DSButtonVariant.filled,
    this.isLoading = false,
    this.icon,
  });

  @override
  Widget build(BuildContext context) {
    final child = isLoading
        ? const SizedBox(
            width: 20,
            height: 20,
            child: CircularProgressIndicator(strokeWidth: 2),
          )
        : Row(
            mainAxisSize: MainAxisSize.min,
            children: [
              if (icon != null) ...[
                Icon(icon, size: 18),
                const SizedBox(width: DSSpacing.sm),
              ],
              Text(label),
            ],
          );

    return switch (variant) {
      DSButtonVariant.filled => FilledButton(
          onPressed: isLoading ? null : onPressed,
          child: child,
        ),
      DSButtonVariant.outlined => OutlinedButton(
          onPressed: isLoading ? null : onPressed,
          child: child,
        ),
      DSButtonVariant.ghost => TextButton(
          onPressed: isLoading ? null : onPressed,
          child: child,
        ),
    };
  }
}
```

## Instructions

1. Read the existing `design_system/` package first if it exists
2. Never duplicate tokens that already exist — extend or update them
3. All token classes use `abstract final class` (cannot instantiate)
4. All token values are `static const`
5. Component names are prefixed with `DS` (e.g., `DSButton`, `DSTextField`)
6. Export every new public symbol from the barrel file
7. Add the `design_system` dependency to any package that needs it in its `pubspec.yaml`
8. Do NOT use `flutter_screenutil` in the design system package itself — keep it framework-agnostic
