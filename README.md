# Calligraphy

**Calligraphy** is a Python library for flexible function overloading, input validation, and auto-generated documentation — ideal for command-based systems like Discord bots.

The primary focus of Calligraphy is to be **powerful**. Calligraphy is best suited for high-level, low-traffic logic where developer control and clarity matter most.

## Table of Contents

- [Calligraphy](#calligraphy)
  - [Table of Contents](#table-of-contents)
  - [Usage](#usage)
    - [Basic Flow](#basic-flow)
    - [Example](#example)
  - [Commands](#commands)
    - [Properties](#properties)
      - [Signature (**required**)](#signature-required)
      - [Validators (**optional**)](#validators-optional)
      - [Descriptions (**optional**)](#descriptions-optional)
      - [Aliases (**optional**)](#aliases-optional)
      - [Fallback (**optional**)](#fallback-optional)
    - [Shorthand](#shorthand)
  - [Signatures](#signatures)
    - [Type checking](#type-checking)
  - [Settings](#settings)

## Usage

Calligraphy lets you define structured signatures for functions, enabling:

- Overloading — dispatch based on input shape.
- Validation — from type checking and casting to value constraints.
- Inline documentation — for commands, overloads, and arguments.

### Basic Flow

1. **Define a signature** using Calligraphy's syntax
2. **Decorate the function** with `@calligraphy(...)`
3. **Optionally add metadata** like descriptions, validators and fallbacks
4. **Optionally overload commands** by declaring commands with the same name
5. **At runtime** Calligraphy will
   - Parse a raw string input
   - Match it to a valid signature
   - Convert inputs to the correct types
   - Validate them
   - Calls the appropriate function or fallback

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

---

## Commands

Calligraphy commands are regular Python functions, decorated to handle structured input.

### Properties

A command can have several properties, all configured via decorators. These control parsing, validation, documentation, and fallback behavior.

#### Signature (**required**)

Describes the expected structure and types of arguments. See the [Signatures](#signatures) section for full syntax.

```python
@calligraphy.signature('(int) age')
def setAge(*args, **kwargs):
    ...
```

#### Validators (**optional**)
Attach validators to specific arguments. Validators should return a truthy value when satisfied. 
   
Validators can also raise a `Calligraphy.ValidationError` to bubble an error into the fallback handler, as described [here](#fallback-optional). This can be useful to provide more specificity to loggers, end-users, etc.

```python
def validate_percentage(value):
    if value < 0 or value > 100:
        raise Calligraphy.ValidationError("Must be between 0 and 100")
    return True

@calligraphy.validators({
    'price': lambda x: x >= 0,
    'discount': validate_percentage
})
```

> **Note -** Although validation callbacks could theoretically be used to check types, it is strongly recommended to use [signature type checking](#type-checking) to enforce typing, as this will immediately cast arguments into their required types.

#### Descriptions (**optional**)

Provide human-readable descriptions for the command and its arguments. Used for auto-generated documentation.

```python
@calligraphy.descriptions({
    'command': 'Calculates discounted price',
    'arguments': {
        'price': 'Original price',
        'discount': 'Discount percentage (0–100)'
    }
})
```

If no description is provided, it defaults to `"No description."`

#### Aliases (**optional**)
Provide aliases for arguments.

```python
@calligraphy.aliases({
    'verbose': ['v'],
    'quiet': ['silent', 'q', 's']
})
```

#### Fallback (**optional**)

Defines what happens when input matching or validation fails.

| Fallback Mode       | Behavior                                                                 |
|---------------------|--------------------------------------------------------------------------|
| `Fallback.IGNORE`  | Ignores errors silently                                                  |
| `Fallback.RAISE`    | Raises a `Calligraphy.MatchingError` or `Calligraphy.ValidationError`    |
| `Fallback.HANDLE` | Passes errors to a custom handler function for logging or messaging     |

```python
def onFailure(error):
    log.warning(error.message)

@calligraphy.fallback(Fallback.HANDLE, onFailure)
```

If no fallback is specified, the global fallback is used. If no global fallback is defined, `Fallback.IGNORE` is used. Similarly, if `Fallback.HANDLE` is used, but no callback is provided, the global callback is used. If no global callback exists, IGNORE happens.

### Shorthand
It is possible to declare some or all properties of a command with the `@calligraphy` decorator.

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
            validator: lambda x: not x.is_integer()
        }
    }
})

def numbers(*args, **kwargs):
    ...
```

> **Note -** Decorator order does not matter! As long as a signature is provided.

## Signatures
### Type checking
## Settings