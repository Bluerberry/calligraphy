# Calligraphy

**Calligraphy** is a Python library for flexible function overloading, input validation, and auto-generated documentation — ideal for command-based systems like Discord bots.

The primary focus of Calligraphy is to be **powerful**. It shines in high-level, low-traffic logic where developer control and clarity matter most.

## Table of Contents

- [Calligraphy](#calligraphy)
  - [Table of Contents](#table-of-contents)
  - [Usage](#usage)
    - [Example](#example)
  - [Commands](#commands)
    - [Properties](#properties)
      - [Signature (**required**)](#signature-required)
      - [Constraints (**optional**)](#constraints-optional)
      - [Descriptions (**optional**)](#descriptions-optional)
      - [Aliases (**optional**)](#aliases-optional)
      - [Local settings (**optional**)](#local-settings-optional)
    - [Shorthand](#shorthand)
  - [Signatures](#signatures)
    - [Arguments](#arguments)
    - [Groups](#groups)
    - [Logic](#logic)
    - [Logical precedence](#logical-precedence)
    - [Type annotations](#type-annotations)
      - [Arrays](#arrays)
    - [Custom Types](#custom-types)
    - [Type precedence](#type-precedence)
    - [Flags](#flags)
  - [Global settings](#global-settings)
    - [Options](#options)

## Usage

Calligraphy lets you define structured input signatures for functions, enabling:

- **Overloading** — Dispatch based on input shape.
- **Validation** — Type checking, casting, and constraints.
- **Inline documentation** — For commands, overloads, and arguments.

### Example

```python
@calligraphy('(int) count [--verbose]')
def repeatAction(*args, **kwargs):
    ...
```

When this command is called with a string like `"5 --verbose"`, Calligraphy parses and validates the input, and dispatches to the function with:

```python
count=5, verbose=True
```

## Commands
Calligraphy commands are regular Python functions, decorated to handle structured input. Every command requires a signature, and can be further configured with descriptions, constraints, aliases, and local settings.

A decorated command takes a raw input string as its first argument. This input is parsed and validated, then dispatched to the correct overload, along with any other `*args` and `**kwargs` originally passed into the command.

### Properties
#### Signature (**required**)

Defines the expected structure and types of input for a command. A signature acts as a blueprint for how the raw input string is parsed and mapped into arguments.

```python
@calligraphy.signature('(int) age')
def setAge(*args, **kwargs):
    ...
```

This expects a single integer argument. Input like `"25"` would result in:

```python
age = 25
```

To understand how to write complex or dynamic signatures, see the [Signatures](#signatures) section.

#### Constraints (**optional**)
Attach custom validation logic to arguments. Each constraint is a function that receives the already type-cast value and returns `True` if valid. It can also raise a `Calligraphy.ConstraintError` to report specific issues.

```python
def validate_percentage(value):
    if value < 0 or value > 100:
        raise Calligraphy.ValidationError("Must be between 0 and 100")
    return True

@calligraphy.constraints({
    'price': lambda x: x >= 0,
    'discount': validate_percentage
})
```

> **Note** \
> Constraints are applied _after_ typecasting. Values passed to constraints are already typed.

#### Descriptions (**optional**)
Provide human-readable documentation for the command and its arguments. Useful for auto-generated help systems.

```python
@calligraphy.descriptions({
    'command': 'Calculates discounted price',
    'arguments': {
        'price': 'Original price',
        'discount': 'Discount percentage (0–100)'
    }
})
```

#### Aliases (**optional**)
Allow alternative names for arguments. Aliases must be unique within the context of a command.

```python
@calligraphy.aliases({
    'verbose': ['v'],
    'quiet': ['silent', 'q', 's']
})
```

#### Local settings (**optional**)
Provide local settings that override global behaviour. Local settings are formatted the same as global settings. Read more about global settings [here](#global-settings).

```python
@calligraphy.localSettings({
    fallback: Fallback.RAISE,
    infer_types: True
})
```

### Shorthand
Instead of using multiple decorators, all configuration can be provided inline via the `@calligraphy` shorthand.

```python
@calligraphy('(int) whole | (float) fraction', {
    description: 'A command using shorthand',
    fallback: Fallback.RAISE,
    arguments: {
        'whole': {
            description: 'Integer',
            aliases: ['integer', 'natural']
        },
        'fraction': {
            description: 'Non-integer fraction',
            constraint: lambda x: not x.is_integer()
        }
    }
})

def numbers(*args, **kwargs):
    ...
```

> **Note** \
> Decorator order does not matter — As long as a signature is provided, Calligraphy is happy!

## Signatures
A signature defines the structure, types, and valid patterns of input that a command can accept. It acts like a lightweight grammar for your function’s arguments.

Signatures support:

- Typed arguments — e.g. `(int) count`
- Optional groups — e.g. `[--verbose]`
- Logical branching — e.g. `foo | bar`, `--a / --b`

### Arguments
There are two types of arguments:

- Positional — matched by position (e.g. `foo bar`)
- Keyword — matched by keyword (e.g. `--name=value`)

Keyword arguments are unordered: \
`--foo=1 --bar=2` is equivalent to `--bar=2 --foo=1`

Keyword arguments are also independent of position: \
`foo bar --baz` ≡ `foo --baz bar` ≡ `--baz foo bar`

### Groups
Arguments can be grouped to specify optionality or requirement:

| Syntax  | Behavior                  |
|---------|---------------------------|
| `<...>` | Required Group (implicit) |
| `[...]` | Optional Group            |

For example:

```python
@calligraphy('foo [bar baz]')
```

Accepts `foo` or `foo bar baz` as inputs. 

> **Note** \
> `foo bar` is **not** a valid input, as `bar` and `baz` are grouped.

### Logic
Groups and arguments can be seperated by logical operators. 

- #### OR operator - `/`
  The OR operator is denoted by `/`, where one or both sides of the operator must be satisfied.

  ```python
  @calligraphy('foo / bar / baz')
  ```
  Accepts as input:
  1) foo
  2) bar
  3) baz
  4) foo bar
  5) foo baz
  6) bar baz
  7) foo bar baz

  > **Note**\
  > observe that position is still relevant! `bar foo` would **not** be a valid input.

- #### XOR operator - `|`
  The XOR operator is denoted by `|`. It requires exactly one side of the operator to be satisfied, not both.

  ```python
  @calligraphy('foo | bar | baz')
  ```

  accepts as input:
   1) foo
   2) bar
   3) baz

- #### AND operator - implied
  When no logical operator is present, the AND operator is implied.

  ```python
  @calligraphy('foo bar')
  ```
  requires both `foo` and `bar`.

### Logical precedence
- AND operators take precendence over OR and XOR operators.
  ```python
  'foo faa | bar baz' ≡ '<foo faa> | <bar baz>'
  ```

- OR and XOR operators have the same precedence, and are evaluated left-to-right.
  ```python
  'foo | bar / baz' ≡ '<foo | bar> / baz'
  'foo / bar | baz' ≡ '<foo / bar> | baz'
  ```

### Type annotations
Arguments can have type annotations, denoted using parentheses. 

```python
@calligraphy('(int) number')
```

| Native Type | Description               |
|-------------|---------------------------|
| `bool`      | Boolean value, determined by [settings](#truthy_values-str-defaulttrue-yes) |
| `int`       | Natural number, consisting only of digits        |
| `float`     | Real number, digits and decimals sperated by `.` |
| `str`       | String            |
| `any`       | Inferred to type with highest precedence |

When no type is provided, `any` is used.

#### Arrays
Arguments can be annotated as an array, denoted by square brackets.
```python
@calligraphy('(str[]) names')
```
These arrays collect positional arguments of the correct type until an argument is encountered with a different type. If a numerical value is provided within the brackets, exactly that amount of arguments are collected
```python
@calligraphy('(bool[3]) threeBooleans')
```
> **Warning**\
>  Arrays are greedy! The signature `(any[]) foo (int) bar` will never be satisfied, as all arguments will be collected by `foo`, leaving none for `bar`.
>
> Similarly, the signature `(int[]) foo (any) bar` might be confusing, as `1 2 3 True` matches as expected, but `1 2 3 4` does not.
>
> It is reccomended to place greedy arrays at the end of your signature. `(int) foo (any[]) bar` matches the first input to foo, and the rest to bar, as expected.

### Custom Types
Custom types allow you to define your own input formats with automatic validation and casting. They are especially useful when working with structured string inputs — such as Discord mentions — that need to be transformed into usable Python objects.

```python
class Mention(calligraphy.CustomType):
    symbol = 'mention'

    def infer(value: str) -> bool:
        return re.match(r'^<@(\d+)>$', value) is not None
    def cast(value: str) -> any:
        return int(value[2:-1])  # Strip <@ and >

# Register the custom type globally
calligraphy.globalSettings({ 'custom_types': [Mention] })
```

Now you can use the `mention` type in your signatures just like any native type:

```python
@calligraphy('(mention[]) users')
```

When called with input like 

```
<@12345> <@56789>
```

Calligraphy automatically casts this to

```python
users = [12345, 56789]
```

### Type precedence
When a type is inferred, type precedence is important. First custom types are checked, in order of their declaration (See [settings](#custom_types-customtype-default)). Then native types are checked, in order of precision. `1` is inferred as an `int`, not `float` or `string`. If `1` is defined as a truthy value, it is inferred as a `bool`.

### Flags
Flags are syntactic sugar for implicitly boolean keyword arguments.
```python
'--foo' ≡ '(bool) --foo' 
```
When a flag is provided as input without value, its value is implied to be `True`.
```python
'--foo' ≡ '--foo=true'
```

## Global settings
Calligraphy can be customized to fit your needs, both project-wide _(globally)_, or per command _(locally)_. Local settings always have precedence over global settings.

Setting global settings is easy!

```python
calligraphy.globalSettings({
    # 'option': value
})
```

### Options
- #### `fallback` _(Fallback, default=Fallback.IGNORE)_
  Defines what happens when input matching or validation fails.

  | Fallback Mode     | Behavior                                                                 |
  |-------------------|--------------------------------------------------------------------------|
  | `Fallback.IGNORE` | Ignores errors silently                                                  |
  | `Fallback.RAISE`  | Raises a `Calligraphy.MatchingError` or `Calligraphy.ConstraintError`    |
  | `Fallback.HANDLE` | Passes errors to a custom handler function for logging or messaging. If no handler is provided, `Fallback.IGNORE` is used instead |

- #### `fallback_handler` _(callable, default=None)_
  Callback for `Fallback.HANDLE` fallback mode. Accepts an error as argument. When `fallback` is not `Fallback.HANDLE` this field has no effect.

- #### `custom_types` _(CustomType[], default=[])_
  Declare custom types to be available globally. Custom types always have precendence over native types. Precendence among custom types is determined by the order of `custom_types`, where the first type has the highest precedence.

- #### `truthy_values` _(str[], default=['true', 'yes'])_
  List of strings interpreted as truthy boolean values.

- #### `falsy_values` _(str[], default=['false', 'no'])_
  List of strings interpreted as falsy boolean values.