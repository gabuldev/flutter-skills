---
name: flutter-forms
description: Use when building a Flutter form or sign-up screen - field validation, wiring to a Cubit, input masks and pt-BR formatting (CPF, CNPJ, phone, postcode, currency, dates), or a multi-step form. Triggers on "registration screen", "validate this field" or "digits only".
---

You are implementing a form for a Flutter feature.

## Three Patterns by Use Case

Choose based on the skill argument, if one was given, or on the context of the feature:

- `simple` — basic form with minimal validation (settings toggle, quick input)
- `validated` — full validation, field-level errors, submit-to-Cubit flow (login, product creation)
- `multi-step` — form split across tabs or stepper (store config, onboarding)

---

## Form Architecture

Every form follows this structure:

1. **StatefulWidget** — forms need mutable state for controllers and keys
2. **GlobalKey<FormState>** — declared as a field, NEVER recreated in `build()`
3. **TextEditingController per field** — initialized in `initState()` or inline, disposed in `dispose()`
4. **Submit via `_formKey.currentState!.validate()`** — returns true only if all validators pass

```dart
class <Feature>Form extends StatefulWidget {
  final <Entity>? initial; // null = create mode, non-null = edit mode
  const <Feature>Form({super.key, this.initial});

  @override
  State<<Feature>Form> createState() => _<Feature>FormState();
}

class _<Feature>FormState extends State<<Feature>Form> {
  // CORRECT: key is a field, created once
  final _formKey = GlobalKey<FormState>();
  final _nameController = TextEditingController();

  @override
  void initState() {
    super.initState();
    if (widget.initial != null) {
      _nameController.text = widget.initial!.name;
    }
  }

  @override
  void dispose() {
    _nameController.dispose();
    super.dispose();
  }

  // ...
}
```

**NEVER do this:**
```dart
// WRONG: recreates key every build, loses form state
@override
Widget build(BuildContext context) {
  final formKey = GlobalKey<FormState>(); // BUG!
  return Form(key: formKey, child: ...);
}
```

---

## Pattern 1: Simple Form

Best for: quick dialogs, single-field inputs, settings

```dart
class _QuickInputDialogState extends State<QuickInputDialog> {
  final _formKey = GlobalKey<FormState>();
  final _controller = TextEditingController();

  void _handleSubmit() {
    if (_formKey.currentState!.validate()) {
      widget.onSave(_controller.text);
      Navigator.of(context).pop();
    }
  }

  @override
  void dispose() {
    _controller.dispose();
    super.dispose();
  }

  @override
  Widget build(BuildContext context) {
    return AlertDialog(
      title: const Text('Adicionar Item'),
      content: Form(
        key: _formKey,
        child: TextFormField(
          controller: _controller,
          decoration: const InputDecoration(labelText: 'Nome'),
          validator: Validators.required('Digite o nome'),
        ),
      ),
      actions: [
        TextButton(
          onPressed: () => Navigator.pop(context),
          child: const Text('Cancelar'),
        ),
        FilledButton(
          onPressed: _handleSubmit,
          child: const Text('Salvar'),
        ),
      ],
    );
  }
}
```

---

## Pattern 2: Validated Form + Cubit Integration

Best for: login, product creation, user management

**Submission flow:**
```
User fills form -> taps submit -> _formKey.validate()
  -> if valid -> controller.submit(data)
  -> BlocListener shows success/error
```

**Login form example:**
```dart
class LoginScreen extends StatefulWidget {
  const LoginScreen({super.key});

  @override
  State<LoginScreen> createState() => _LoginScreenState();
}

class _LoginScreenState extends State<LoginScreen> {
  final _formKey = GlobalKey<FormState>();
  final _emailController = TextEditingController();
  final _passwordController = TextEditingController();
  bool _obscurePassword = true;

  @override
  void dispose() {
    _emailController.dispose();
    _passwordController.dispose();
    super.dispose();
  }

  void _handleSubmit() {
    if (_formKey.currentState!.validate()) {
      context.read<AuthController>().login(
            email: _emailController.text.trim(),
            password: _passwordController.text,
          );
    }
  }

  @override
  Widget build(BuildContext context) {
    return BlocListener<AuthController, AuthStatus>(
      listener: (context, status) {
        if (status is AuthSuccess) {
          Navigator.pushReplacementNamed(context, '/home');
        } else if (status is AuthFailure) {
          ScaffoldMessenger.of(context).showSnackBar(
            SnackBar(content: Text(status.message)),
          );
        }
      },
      child: Scaffold(
        body: Form(
          key: _formKey,
          child: Padding(
            padding: const EdgeInsets.all(24),
            child: Column(
              mainAxisAlignment: MainAxisAlignment.center,
              children: [
                TextFormField(
                  controller: _emailController,
                  decoration: const InputDecoration(
                    labelText: 'E-mail',
                    prefixIcon: Icon(Icons.email_outlined),
                  ),
                  keyboardType: TextInputType.emailAddress,
                  validator: Validators.compose([
                    Validators.required('Digite seu e-mail'),
                    Validators.email('E-mail invalido'),
                  ]),
                ),
                const SizedBox(height: 16),
                TextFormField(
                  controller: _passwordController,
                  obscureText: _obscurePassword,
                  decoration: InputDecoration(
                    labelText: 'Senha',
                    prefixIcon: const Icon(Icons.lock_outlined),
                    suffixIcon: IconButton(
                      icon: Icon(_obscurePassword
                          ? Icons.visibility_off
                          : Icons.visibility),
                      onPressed: () =>
                          setState(() => _obscurePassword = !_obscurePassword),
                    ),
                  ),
                  validator: Validators.required('Digite sua senha'),
                ),
                const SizedBox(height: 24),
                BlocBuilder<AuthController, AuthStatus>(
                  builder: (context, status) {
                    final isLoading = status is AuthLoading;
                    return FilledButton(
                      onPressed: isLoading ? null : _handleSubmit,
                      child: isLoading
                          ? const SizedBox(
                              width: 20,
                              height: 20,
                              child: CircularProgressIndicator(strokeWidth: 2),
                            )
                          : const Text('Entrar'),
                    );
                  },
                ),
              ],
            ),
          ),
        ),
      ),
    );
  }
}
```

**Product creation form example (with money input and image picker):**
```dart
class _ProductDialogState extends State<ProductDialog> {
  final _formKey = GlobalKey<FormState>();
  final _nameController = TextEditingController();
  final _codeController = TextEditingController();
  final _descriptionController = TextEditingController();
  late final _priceController = MoneyMaskedTextController(
    initialValue: widget.product?.price ?? 0,
    leftSymbol: 'R\$',
    decimalSeparator: ',',
    thousandSeparator: '.',
    precision: 2,
  );

  Category? _selectedCategory;
  final List<XFile> _newImages = [];
  List<String> _existingImages = [];

  @override
  void initState() {
    super.initState();
    if (widget.product != null) {
      _nameController.text = widget.product!.name;
      _codeController.text = widget.product!.code.toString();
      _descriptionController.text = widget.product!.description;
      _selectedCategory = widget.product!.category;
      _existingImages = List.from(widget.product!.images);
    }
  }

  @override
  void dispose() {
    _nameController.dispose();
    _codeController.dispose();
    _descriptionController.dispose();
    _priceController.dispose();
    super.dispose();
  }

  Future<void> _pickImages() async {
    final picker = ImagePicker();
    final images = await picker.pickMultiImage();
    if (images.isNotEmpty) {
      setState(() => _newImages.addAll(images));
    }
  }

  void _handleSubmit() {
    if (_formKey.currentState!.validate()) {
      final product = Product(
        id: widget.product?.id ?? '',
        name: _nameController.text,
        code: int.tryParse(_codeController.text) ?? 0,
        price: _priceController.numberValue,
        description: _descriptionController.text,
        category: _selectedCategory!,
        images: _existingImages,
      );
      widget.onSave(product, _newImages);
      Navigator.of(context).pop();
    }
  }

  @override
  Widget build(BuildContext context) {
    return AlertDialog(
      title: Text(widget.product == null ? 'Novo Produto' : 'Editar Produto'),
      content: Form(
        key: _formKey,
        child: SingleChildScrollView(
          child: Column(
            mainAxisSize: MainAxisSize.min,
            children: [
              TextFormField(
                controller: _nameController,
                decoration: const InputDecoration(labelText: 'Nome do Produto'),
                validator: Validators.required('Digite o nome do produto'),
              ),
              const SizedBox(height: 16),
              TextFormField(
                controller: _priceController,
                decoration: const InputDecoration(labelText: 'Preco'),
                keyboardType: TextInputType.number,
                validator: Validators.minValue(0.01, 'Preco deve ser maior que zero'),
              ),
              const SizedBox(height: 16),
              DropdownButtonFormField<Category>(
                value: _selectedCategory,
                decoration: const InputDecoration(labelText: 'Categoria'),
                items: widget.categories
                    .map((c) => DropdownMenuItem(value: c, child: Text(c.name)))
                    .toList(),
                onChanged: (value) => setState(() => _selectedCategory = value),
                validator: (value) =>
                    value == null ? 'Selecione uma categoria' : null,
              ),
              const SizedBox(height: 16),
              TextFormField(
                controller: _descriptionController,
                decoration: const InputDecoration(labelText: 'Descricao'),
                maxLines: 3,
              ),
              const SizedBox(height: 16),
              // Image picker section
              Wrap(
                spacing: 8,
                children: [
                  ..._existingImages.map((url) => _ImageTile(
                        imageUrl: url,
                        onRemove: () => setState(
                            () => _existingImages.remove(url)),
                      )),
                  IconButton.filled(
                    onPressed: _pickImages,
                    icon: const Icon(Icons.add_a_photo),
                  ),
                ],
              ),
            ],
          ),
        ),
      ),
      actions: [
        TextButton(
          onPressed: () => Navigator.pop(context),
          child: const Text('Cancelar'),
        ),
        FilledButton(
          onPressed: _handleSubmit,
          child: const Text('Salvar'),
        ),
      ],
    );
  }
}
```

---

## Pattern 3: Multi-Step Form (TabBarView)

Best for: store configuration, onboarding, complex entity creation

Use `AutomaticKeepAliveClientMixin` to preserve form state when switching tabs.

```dart
class StoreConfigScreen extends StatefulWidget {
  const StoreConfigScreen({super.key});

  @override
  State<StoreConfigScreen> createState() => _StoreConfigScreenState();
}

class _StoreConfigScreenState extends State<StoreConfigScreen>
    with SingleTickerProviderStateMixin {
  late final TabController _tabController;

  // One form key per tab — each tab validates independently
  final _infoFormKey = GlobalKey<FormState>();
  final _hoursFormKey = GlobalKey<FormState>();
  final _paymentFormKey = GlobalKey<FormState>();

  final _storeNameController = TextEditingController();
  final _phoneController = TextEditingController();

  @override
  void initState() {
    super.initState();
    _tabController = TabController(length: 3, vsync: this);
  }

  @override
  void dispose() {
    _tabController.dispose();
    _storeNameController.dispose();
    _phoneController.dispose();
    super.dispose();
  }

  bool _validateAllTabs() {
    final infoValid = _infoFormKey.currentState?.validate() ?? false;
    final hoursValid = _hoursFormKey.currentState?.validate() ?? false;
    final paymentValid = _paymentFormKey.currentState?.validate() ?? false;

    if (!infoValid) _tabController.animateTo(0);
    else if (!hoursValid) _tabController.animateTo(1);
    else if (!paymentValid) _tabController.animateTo(2);

    return infoValid && hoursValid && paymentValid;
  }

  void _handleSubmit() {
    if (_validateAllTabs()) {
      context.read<StoreConfigController>().save(/* collected data */);
    }
  }

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(
        title: const Text('Configuracoes da Loja'),
        bottom: TabBar(
          controller: _tabController,
          tabs: const [
            Tab(text: 'Informacoes'),
            Tab(text: 'Horarios'),
            Tab(text: 'Pagamento'),
          ],
        ),
      ),
      body: TabBarView(
        controller: _tabController,
        children: [
          _StoreInfoTab(formKey: _infoFormKey, nameController: _storeNameController),
          _StoreHoursTab(formKey: _hoursFormKey),
          _PaymentConfigTab(formKey: _paymentFormKey),
        ],
      ),
      floatingActionButton: FloatingActionButton.extended(
        onPressed: _handleSubmit,
        label: const Text('Salvar'),
        icon: const Icon(Icons.save),
      ),
    );
  }
}

/// Each tab uses AutomaticKeepAliveClientMixin to preserve state
class _StoreInfoTab extends StatefulWidget {
  final GlobalKey<FormState> formKey;
  final TextEditingController nameController;

  const _StoreInfoTab({required this.formKey, required this.nameController});

  @override
  State<_StoreInfoTab> createState() => _StoreInfoTabState();
}

class _StoreInfoTabState extends State<_StoreInfoTab>
    with AutomaticKeepAliveClientMixin {
  @override
  bool get wantKeepAlive => true;

  @override
  Widget build(BuildContext context) {
    super.build(context); // Required by AutomaticKeepAliveClientMixin
    return Form(
      key: widget.formKey,
      child: ListView(
        padding: const EdgeInsets.all(16),
        children: [
          TextFormField(
            controller: widget.nameController,
            decoration: const InputDecoration(labelText: 'Nome da Loja'),
            validator: Validators.required('Digite o nome da loja'),
          ),
          // ... more fields
        ],
      ),
    );
  }
}
```

---

## Reusable Validators

Create a utility class in `core/shared/validators.dart`. Each validator is a function that returns a `FormFieldValidator<String>` (i.e., `String? Function(String?)`).

```dart
/// core/shared/validators.dart
class Validators {
  Validators._();

  /// Composes multiple validators — runs in order, returns first error
  static FormFieldValidator<String> compose(
      List<FormFieldValidator<String>> validators) {
    return (value) {
      for (final validator in validators) {
        final error = validator(value);
        if (error != null) return error;
      }
      return null;
    };
  }

  /// Field must not be empty
  static FormFieldValidator<String> required(String message) {
    return (value) =>
        (value == null || value.trim().isEmpty) ? message : null;
  }

  /// Valid email format
  static FormFieldValidator<String> email(String message) {
    return (value) {
      if (value == null || value.isEmpty) return null; // use with required()
      final regex = RegExp(r'^[\w\.\-]+@[\w\.\-]+\.\w{2,}$');
      return regex.hasMatch(value) ? null : message;
    };
  }

  /// Minimum character length
  static FormFieldValidator<String> minLength(int min, String message) {
    return (value) =>
        (value != null && value.length < min) ? message : null;
  }

  /// Maximum character length
  static FormFieldValidator<String> maxLength(int max, String message) {
    return (value) =>
        (value != null && value.length > max) ? message : null;
  }

  /// Numeric value only
  static FormFieldValidator<String> numeric(String message) {
    return (value) {
      if (value == null || value.isEmpty) return null;
      return double.tryParse(value) == null ? message : null;
    };
  }

  /// Minimum numeric value (for MoneyMaskedTextController fields)
  static FormFieldValidator<String> minValue(
      double min, String message) {
    return (value) {
      if (value == null || value.isEmpty) return null;
      final cleaned =
          value.replaceAll('R\$', '').replaceAll('.', '').replaceAll(',', '.');
      final number = double.tryParse(cleaned.trim());
      return (number == null || number < min) ? message : null;
    };
  }

  /// Password strength: min 6 chars
  static FormFieldValidator<String> password(String message) {
    return (value) =>
        (value != null && value.length < 6) ? message : null;
  }

  /// Matches another field (password confirmation)
  static FormFieldValidator<String> matches(
      TextEditingController other, String message) {
    return (value) => (value != other.text) ? message : null;
  }
}
```

**Usage with compose:**
```dart
TextFormField(
  validator: Validators.compose([
    Validators.required('Campo obrigatorio'),
    Validators.email('E-mail invalido'),
  ]),
),
```

---

## Field Type Reference

### TextFormField (standard)
```dart
TextFormField(
  controller: _controller,
  decoration: const InputDecoration(labelText: 'Nome'),
  validator: Validators.required('Campo obrigatorio'),
)
```

### DropdownButtonFormField
```dart
DropdownButtonFormField<Category>(
  value: _selectedCategory,
  decoration: const InputDecoration(labelText: 'Categoria'),
  items: categories
      .map((c) => DropdownMenuItem(value: c, child: Text(c.name)))
      .toList(),
  onChanged: (value) => setState(() => _selectedCategory = value),
  validator: (value) => value == null ? 'Selecione uma categoria' : null,
)
```

### Money Input (Brazilian Real)
Uses `flutter_masked_text3` package, which is already a dependency in the project.

```dart
late final _priceController = MoneyMaskedTextController(
  initialValue: widget.product?.price ?? 0,
  leftSymbol: 'R\$',
  decimalSeparator: ',',
  thousandSeparator: '.',
  precision: 2,
);

// In form:
TextFormField(
  controller: _priceController,
  decoration: const InputDecoration(labelText: 'Preco'),
  keyboardType: TextInputType.number,
  validator: Validators.minValue(0.01, 'Preco deve ser maior que zero'),
)

// Reading the value:
final double price = _priceController.numberValue;
```

### Password Field with Obscure Toggle
```dart
bool _obscurePassword = true;

TextFormField(
  controller: _passwordController,
  obscureText: _obscurePassword,
  decoration: InputDecoration(
    labelText: 'Senha',
    prefixIcon: const Icon(Icons.lock_outlined),
    suffixIcon: IconButton(
      icon: Icon(
          _obscurePassword ? Icons.visibility_off : Icons.visibility),
      onPressed: () =>
          setState(() => _obscurePassword = !_obscurePassword),
    ),
  ),
  validator: Validators.compose([
    Validators.required('Digite sua senha'),
    Validators.password('Senha deve ter no minimo 6 caracteres'),
  ]),
)
```

### Password Confirmation
```dart
TextFormField(
  controller: _confirmPasswordController,
  obscureText: true,
  decoration: const InputDecoration(labelText: 'Confirmar Senha'),
  validator: Validators.compose([
    Validators.required('Confirme sua senha'),
    Validators.matches(_passwordController, 'As senhas nao coincidem'),
  ]),
)
```

### Date Picker Field
```dart
DateTime? _selectedDate;

FormField<DateTime>(
  initialValue: _selectedDate,
  validator: (value) => value == null ? 'Selecione uma data' : null,
  builder: (field) {
    return InkWell(
      onTap: () async {
        final picked = await showDatePicker(
          context: context,
          initialDate: _selectedDate ?? DateTime.now(),
          firstDate: DateTime(2020),
          lastDate: DateTime(2030),
        );
        if (picked != null) {
          setState(() => _selectedDate = picked);
          field.didChange(picked);
        }
      },
      child: InputDecorator(
        decoration: InputDecoration(
          labelText: 'Data',
          errorText: field.errorText,
        ),
        child: Text(
          _selectedDate != null
              ? '${_selectedDate!.day}/${_selectedDate!.month}/${_selectedDate!.year}'
              : 'Selecione',
        ),
      ),
    );
  },
)
```

### Image Picker (Product Photos)
```dart
final List<XFile> _newImages = [];
List<String> _existingImages = [];

Future<void> _pickImages() async {
  if (_existingImages.length + _newImages.length >= 5) {
    ScaffoldMessenger.of(context).showSnackBar(
      const SnackBar(content: Text('Maximo de 5 imagens permitido.')),
    );
    return;
  }
  final picker = ImagePicker();
  final images = await picker.pickMultiImage();
  if (images.isNotEmpty) {
    setState(() {
      final totalAllowed = 5 - _existingImages.length - _newImages.length;
      _newImages.addAll(images.take(totalAllowed));
    });
  }
}
```

---

## Form + Cubit Integration Rules

1. **Form collects data, controller processes it** — the form widget builds the entity/DTO, then calls `controller.submit(data)`
2. **Use `BlocListener` for side effects** — navigation, snackbar, dialog. Never navigate inside `BlocBuilder`
3. **Use `BlocBuilder` only for the submit button** — to show loading spinner and disable the button
4. **Controller emits Loading before async work, Success or Failure after** — see `/flutter-state` skill
5. **In dialogs**: call `widget.onSave(entity)` callback instead of using a controller directly, since the parent screen owns the controller

```dart
// Screen owns the controller and reacts to state
BlocListener<MenuController, MenuStatus>(
  listener: (context, status) {
    if (status is MenuSuccess) {
      ScaffoldMessenger.of(context).showSnackBar(
        const SnackBar(content: Text('Produto salvo com sucesso!')),
      );
    } else if (status is MenuFailure) {
      ScaffoldMessenger.of(context).showSnackBar(
        SnackBar(content: Text(status.message)),
      );
    }
  },
  child: /* form widget */,
),
```

---

## Instructions

1. Read the existing screen/dialog if one already exists in the module
2. Choose the right pattern (`simple`, `validated`, `multi-step`) based on the form's complexity
3. If a `Validators` class does not yet exist in `core/shared/`, create one
4. Use `MoneyMaskedTextController` from `flutter_masked_text3` for any price/currency fields
5. Always dispose all controllers in `dispose()`
6. Never recreate `GlobalKey<FormState>` inside `build()` — always declare as a class field
7. For forms inside `TabBarView`, use `AutomaticKeepAliveClientMixin`
8. Connect form submission to the module's controller following the `/flutter-state` skill patterns
9. Use Portuguese strings for user-facing validation messages (e.g., "Campo obrigatorio")
