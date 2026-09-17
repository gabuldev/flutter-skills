---
name: flutter-accessibility
description: Use when implementing or auditing accessibility on a Flutter screen — the Semantics widget, screen-reader labels (TalkBack/VoiceOver), colour contrast, minimum tap-target size, system font scaling, or semantics in a web build. Also triggers on "make this accessible", "a11y", or a request for an accessibility review.
---

You are implementing or auditing accessibility (a11y) for a Flutter app.

## Context

Accessibility pressure differs by surface, and it changes what you prioritise:

- **Customer-facing web** — the highest bar. Screen-reader semantics are off by
  default on Flutter web (see section 5), the audience is unknown, and in many
  jurisdictions this is a legal requirement, not a nicety.
- **Internal/admin web** — keyboard navigation and contrast matter most; staff
  use these all day.
- **Mobile** — platform screen readers (TalkBack/VoiceOver) mostly work if you
  give widgets labels and keep tap targets large enough.

Choose based on the skill argument, if one was given, or on the screen/task at hand:
- `audit` — Review existing code for a11y violations and produce a report with fixes
- `implement` — Add accessibility annotations and fixes to existing widgets
- `web-setup` — Enable Flutter web semantics and configure for screen readers
- If no mode specified, default to `implement`

---

## 1. Semantic Annotations

Wrap custom widgets with `Semantics` to expose meaning to assistive technology:

```dart
// Label interactive elements
Semantics(
  label: 'Adicionar ao pedido',
  button: true,
  child: CustomAddButton(...),
)

// Group related elements into a single announcement
MergeSemantics(
  child: Row(
    children: [
      Icon(Icons.star),
      Text('4.5'),
    ],
  ),
)

// Hide decorative elements from screen readers
ExcludeSemantics(
  child: DecorativeBackground(),
)
```

### When to use each

| Widget | Use case |
|---|---|
| `Semantics(label:)` | Custom widgets that lack built-in semantics (raw GestureDetector, Container with onTap) |
| `Semantics(button: true)` | Custom buttons not using Material `ElevatedButton`/`TextButton` |
| `Semantics(header: true)` | Section headers (category names in a catalog) |
| `Semantics(image: true, label:)` | Product/content images |
| `MergeSemantics` | Icon + text pairs, price + currency pairs |
| `ExcludeSemantics` | Purely decorative images, background gradients, dividers |

Material widgets (`ElevatedButton`, `TextField`, `Checkbox`) already have built-in semantics. Do NOT double-wrap them unless overriding the default label.

---

## 2. Color Contrast

Follow WCAG 2.1 minimum contrast ratios:

| Element | Minimum ratio |
|---|---|
| Normal text (< 18sp) | **4.5:1** |
| Large text (>= 18sp or >= 14sp bold) | **3.0:1** |
| UI components & graphical objects | **3.0:1** |

### Verification

- Use the **Accessibility Scanner** on Android (Settings > Accessibility)
- Run **Lighthouse** audit in Chrome DevTools for the web build
- Use online contrast checkers (WebAIM Contrast Checker) with hex values from `ds_colors.dart`

### Violations that show up again and again

- Light gray text on white backgrounds for secondary info (descriptions, prices in light theme)
- Disabled button text that falls below 3.0:1
- Placeholder text in search fields

When fixing, update colors in `design_system/` tokens so the fix propagates everywhere.

---

## 3. Tap Targets

All interactive elements must be at least **48x48 logical pixels**:

```dart
// BAD - icon button too small
GestureDetector(
  onTap: onAdd,
  child: Icon(Icons.add, size: 24),
)

// GOOD - ensure minimum hit area
IconButton(
  onPressed: onAdd,
  icon: Icon(Icons.add, size: 24),
  // IconButton already enforces 48x48 by default
)

// GOOD - for custom widgets, use constraints
GestureDetector(
  onTap: onAdd,
  child: ConstrainedBox(
    constraints: const BoxConstraints(minWidth: 48, minHeight: 48),
    child: Center(child: Icon(Icons.add, size: 24)),
  ),
)
```

### Spacing between targets

Adjacent tap targets must have enough spacing to avoid accidental taps. Minimum **8px** gap between interactive elements.

---

## 4. Screen Reader Support

### Labels for images (product photos, avatars, thumbnails)

```dart
// Product image in a catalog
Semantics(
  image: true,
  label: 'Foto do produto: ${product.name}',
  child: CachedNetworkImage(
    imageUrl: product.imageUrl,
    // ...
  ),
)

// Decorative images (backgrounds, patterns)
ExcludeSemantics(
  child: Image.asset('assets/background_pattern.png'),
)
```

### Labels for icon buttons

```dart
IconButton(
  icon: Icon(Icons.shopping_cart),
  tooltip: 'Ver carrinho', // tooltip doubles as semantic label
  onPressed: () => openCart(),
)
```

### Price announcements

```dart
// Announce price clearly
MergeSemantics(
  child: Row(
    children: [
      Text('R\$'),
      Text('29,90'),
    ],
  ),
)
// Screen reader announces: "R$ 29,90"
```

---

## 5. Web Accessibility (any web build)

Flutter web **disables semantics by default** for performance. For customer-facing apps this is unacceptable.

### Enable semantics in web builds

In the app's `web/index.html`, add before the Flutter loader script:

```html
<script>
  // Enable accessibility for Flutter web
  window.flutterConfiguration = {
    canvasKitBaseUrl: '/canvaskit/',
  };
</script>
```

In the app's `main.dart`:

```dart
import 'package:flutter/semantics.dart';

void main() {
  // Force enable semantics for web accessibility
  WidgetsFlutterBinding.ensureInitialized();
  SemanticsBinding.instance.ensureSemantics();

  runApp(const MyApp());
}
```

### Alternative: enable via URL parameter

For testing, append `?semantics=true` to the web URL to activate the semantics tree without code changes.

### Web-specific considerations

- Ensure the HTML `<title>` tag is descriptive (e.g., "Product Catalog — Acme")
- Add `lang="pt-BR"` to the `<html>` tag in `index.html`
- Flutter web renders to canvas, so ALL accessibility depends on the Semantics tree being correct

---

## 6. Font Scaling

Support dynamic text sizing for users who increase system font size:

```dart
// BAD - breaks with large fonts
SizedBox(
  height: 48,
  child: Text('Categoria', style: TextStyle(fontSize: 16)),
)

// GOOD - adapt to text scale
ConstrainedBox(
  constraints: const BoxConstraints(minHeight: 48),
  child: Text('Categoria', style: TextStyle(fontSize: 16)),
)
```

### Guidelines

- Use `minHeight` / `minWidth` instead of fixed `height` / `width` for containers holding text
- Test with system font scale at **2.0x** (Settings > Display > Font size)
- Ensure text does not overflow or get clipped — use `Flexible`, `Expanded`, or `overflow: TextOverflow.ellipsis` with `maxLines`
- Product names and descriptions must remain readable at large scales

### Limiting scale when necessary

Only limit scale factor for truly constrained UI (e.g., small price badges):

```dart
MediaQuery.withClampedTextScaling(
  minScaleFactor: 1.0,
  maxScaleFactor: 1.5,
  child: PriceBadge(...),
)
```

---

## 7. Focus Management (Web)

Keyboard navigation is essential for web accessibility:

```dart
// Ensure custom interactive widgets can receive focus
Focus(
  child: GestureDetector(
    onTap: onTap,
    child: MenuItemCard(...),
  ),
)

// Set logical focus traversal order
FocusTraversalGroup(
  policy: OrderedTraversalPolicy(),
  child: Column(
    children: [
      FocusTraversalOrder(
        order: NumericFocusOrder(1),
        child: SearchBar(),
      ),
      FocusTraversalOrder(
        order: NumericFocusOrder(2),
        child: CategoryFilter(),
      ),
      FocusTraversalOrder(
        order: NumericFocusOrder(3),
        child: ProductGrid(),
      ),
    ],
  ),
)
```

### Keyboard shortcuts for the web build

- **Tab** — move between interactive elements
- **Enter/Space** — activate buttons
- **Escape** — close modals/bottom sheets

Ensure all modals trap focus (Material `showDialog` and `showModalBottomSheet` do this automatically).

---

## 8. Accessible Images

Every meaningful image needs descriptive text:

```dart
// Product image
Semantics(
  image: true,
  label: product.name, // "Pizza Margherita"
  child: ProductImage(url: product.imageUrl),
)

// Category icon
Semantics(
  image: true,
  label: 'Categoria: ${category.name}',
  child: CategoryIcon(icon: category.icon),
)

// Restaurant logo
Semantics(
  image: true,
  label: 'Logo do restaurante ${restaurant.name}',
  child: RestaurantLogo(url: restaurant.logoUrl),
)

// Decorative divider — exclude
ExcludeSemantics(
  child: Image.asset('assets/divider_ornament.png'),
)
```

---

## 9. Accessible Forms

Forms (search, checkout, sign-up) must announce labels and errors:

```dart
TextFormField(
  decoration: InputDecoration(
    labelText: 'Nome completo', // announced by screen reader
    hintText: 'Digite seu nome',
    errorText: hasError ? 'Nome e obrigatorio' : null, // announced on error
  ),
  validator: (value) {
    if (value == null || value.isEmpty) {
      return 'Nome e obrigatorio'; // announced when validation fails
    }
    return null;
  },
)
```

### Guidelines

- Always use `labelText` on `InputDecoration`, not just `hintText` (hints disappear on focus)
- Error messages must be descriptive: say what is wrong, not just "campo invalido"
- After form submission with errors, move focus to the first field with an error
- Group related fields with `MergeSemantics` or `Semantics(label:)` when context is needed

---

## 10. Audit Workflow

### When mode is `audit`

1. **Scan widgets** for missing `Semantics` on:
   - Images (`Image`, `CachedNetworkImage`, `SvgPicture`)
   - Custom buttons (`GestureDetector`, `InkWell` without `Semantics`)
   - Icon-only buttons without `tooltip`
   - Decorative elements that should use `ExcludeSemantics`

2. **Check tap target sizes** — find interactive widgets with constrained sizes below 48x48

3. **Review color usage** — cross-reference text colors against background colors in the design system tokens

4. **Check web semantics** — verify `SemanticsBinding.instance.ensureSemantics()` is called in every web build

5. **Check font scaling** — find fixed `height`/`width` on containers holding text

6. **Produce report** in this format:

```
## Accessibility Audit: [app_name]

### Critical (must fix)
- [ ] Issue description — file:line

### Important (should fix)
- [ ] Issue description — file:line

### Minor (nice to have)
- [ ] Issue description — file:line
```

### Manual testing checklist

| Platform | Tool | How to test |
|---|---|---|
| Android | TalkBack | Settings > Accessibility > TalkBack. Navigate the app with swipe gestures. |
| iOS | VoiceOver | Settings > Accessibility > VoiceOver. Swipe to navigate. |
| macOS | VoiceOver | Cmd+F5. Use arrow keys to navigate web apps in Safari/Chrome. |
| Web | Lighthouse | Chrome DevTools > Lighthouse > Accessibility. Target score >= 90. |
| Web | axe DevTools | Chrome extension. Finds WCAG violations in rendered page. |
| Web | Tab navigation | Press Tab through the entire checkout/browse flow without using a mouse. |

### Lighthouse audit for the web build

```bash
# Run Lighthouse CI from command line
npx lighthouse http://localhost:PORT --only-categories=accessibility --output=json --output-path=./a11y-report.json
```

Target: **Lighthouse accessibility score >= 90** for the web build.
