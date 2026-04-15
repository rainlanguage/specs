# Builder Configuration (`builder`)

## Introduction

The `builder` section within the YAML configuration provides metadata specifically tailored for building user interfaces related to Rainlang strategies. It acts as a bridge between the core technical definition of deployments, tokens, orders, etc., and their presentation to an end-user in a graphical application.

This section dictates how strategy deployments are named, described, and how their configurable parameters (bindings, deposits, token selections) should be rendered as interactive UI elements. It allows for user-friendly labels, descriptions, default values, and predefined options (presets) to simplify the user experience.

## Top-Level `builder` Object

The root of the builder configuration is the `builder:` key. It contains general information about the strategy or set of deployments described in the file.

```yaml
builder:
  name: Fixed limit
  description: Fixed limit order strategy
  short-description: Buy WETH with USDC on Base.
  deployments:
    # ... deployment configurations ...
```

### Fields

* `name`
  * **Required**: Yes
  * **Description**: The primary, human-readable name for the overall strategy or configuration presented in the builder.
* `description`
  * **Required**: Yes
  * **Description**: A more detailed description of the strategy or configuration, intended for display in the builder.
* `short-description`
  * **Required**: No
  * **Description**: An optional, concise description, potentially used in contexts where space is limited (e.g., list views, tooltips).
* `deployments`
  * **Required**: Yes
  * **Description**: A map containing the specific UI configurations for one or more named deployments. See the [Deployments Map](#deployments-map-deployments) section for details.

## Deployments Map (`deployments`)

The `deployments` field under `builder` holds a map where each key is the name of a deployment (which must correspond to a deployment defined elsewhere in the configuration, e.g., under the top-level `deployments:` key) and the value is an object defining the builder configuration specific to that deployment.

```yaml
builder:
  # ... name, description ...
  deployments:
    <deployment-key-1>:
      # ... configuration for deployment 1 ...
    <deployment-key-2>:
      # ... configuration for deployment 2 ...
    # ... etc ...
```

## Deployment Configuration (`<deployment-key>`)

Each entry within the `deployments` map defines the specific UI elements and text for a single deployment.

```yaml
builder:
  # ...
  deployments:
    some-deployment: # This key must match a defined deployment
      name: Buy WETH with USDC on Base.
      description: Buy WETH with USDC for fixed price on Base network.
      short-description: Buy WETH with USDC on Base.
      deposits:
        # ... deposit items ...
      fields:
        # ... field items ...
      select-tokens:
        # ... select token items ...
```

### Fields

* `name`
  * **Required**: Yes
  * **Description**: The name of this specific deployment variation as it should appear in the builder.
* `description`
  * **Required**: Yes
  * **Description**: A detailed description for this specific deployment variation.
* `short-description`
  * **Required**: No
  * **Description**: An optional, concise description for this deployment variation.
* `deposits`
  * **Required**: Yes
  * **Description**: A list of [Deposit Items](#deposit-item) that defines the tokens users can deposit into the strategy and provides UI hints like presets.
* `fields`
  * **Required**: Yes
  * **Description**: A list of [Field Items](#field-item) that defines the user-configurable parameters (bindings) for the strategy, including labels, descriptions, presets, and defaults.
* `select-tokens`
  * **Required**: No
  * **Description**: A list of [Select Token Items](#select-token-item) that defines specific tokens that might be selectable in other parts of the UI for this deployment (e.g., choosing an output token if the strategy supports it).

## Deposit Item

Each item in the `deposits` list defines a token that can be deposited and optional preset amounts for the UI.

```yaml
# Example within a deployment's deposits list:
deposits:
  - token: token1 # Must match a defined token key
    presets:
      - 0
      - 10
      - 100
      - 1000
      - 10000
    validation:
      minimum: 0
      maximum: 100000
      multipleOf: 1 # Ensures the value is a whole number if desired
  - token: token2
    # No presets defined for this token
    validation:
      minimum: 0.001 # Example for a token with high precision
```

### Fields

* `token`
  * **Required**: Yes
  * **Description**: The key referencing a token defined in the top-level `tokens` section. This specifies which token the deposit configuration applies to.
* `presets`
  * **Required**: No
  * **Description**: An optional list of suggested deposit amounts to display as quick options in the UI. The UI will typically show these alongside the primary deposit input field.
* `validation`
  * **Required**: No
  * **Description**: An optional [Validation Object](#validation-object) defining validation rules for the deposit amount. For deposits, this will always use number validation rules. See the [Validation Object](#validation-object) section for details on available number validation fields like `minimum`, `exclusiveMinimum`, `maximum`, `exclusiveMaximum`, and `multipleOf`.

## Field Item

Each item in the `fields` list defines a user-configurable input field, mapping it to an underlying Rainlang binding.

```yaml
# Example within a deployment's fields list:
fields:
  - binding: binding-1 # The corresponding binding in the Rainlang source
    name: Price
    description: The price for the order
    presets:
      # ... preset items for this field ...
    default: 100.50
    validation:
      type: number
      minimum: 0.01
      multipleOf: 0.01
  - binding: binding-2
    name: Slippage Tolerance
    description: Maximum allowed slippage in percentage
    validation:
      type: number
      minimum: 0
      maximum: 100
  - binding: user-provided-note
    name: Order Note
    description: A custom note for the order
    validation:
      type: string
      maxLength: 140
    show-custom-field: true # Explicitly allow custom input
```

### Fields

* `binding`
  * **Required**: Yes
  * **Description**: The name of the binding in the associated Rainlang source code that this field provides the value for.
* `name`
  * **Required**: Yes
  * **Description**: The human-readable label displayed for this input field in the builder.
* `description`
  * **Required**: No
  * **Description**: An optional, longer description or help text displayed for this field, potentially as a tooltip or helper text.
* `presets`
  * **Required**: No
  * **Description**: An optional list of [Preset Items](#preset-item) containing predefined values the user can select for this field.
* `default`
  * **Required**: No
  * **Description**: An optional default value to pre-populate the input field with.
* `validation`
  * **Required**: No
  * **Description**: An optional [Validation Object](#validation-object) defining validation rules for the field's value. This includes specifying the semantic type (e.g., "number", "string", "boolean") and type-specific rules like `minimum`/`exclusiveMinimum`/`maximum`/`exclusiveMaximum` for numbers, or `minLength`/`maxLength` for strings. See the [Validation Object](#validation-object) section for details.
* `show-custom-field`
  * **Required**: No
  * **Description**: Controls whether the user is presented with a free-form input field in addition to any defined presets. When set to a falsy value (e.g., `false`), the user might only be able to select from the presets. When set to a truthy value or omitted, a custom input field is typically shown.

## Preset Item

Each item in a field's `presets` list defines a single predefined option for that field.

```yaml
# Example within a field's presets list:
presets:
  - name: Preset 1 Name # Optional name for the preset
    value: 0x1234567890abcdef1234567890abcdef12345678
  - value: false # Preset without an explicit name
  - name: Preset 3 Name
    value: some-string
```

### Fields

* `name`
  * **Required**: No
  * **Description**: An optional label for the preset option shown in the UI (e.g., in a dropdown or radio button list). If omitted, the `value` itself might be displayed.
* `value`
  * **Required**: Yes
  * **Description**: The actual value that will be used for the binding if this preset is selected.

## Select Token Item

Each item in the optional `select-tokens` list defines a token that might be presented for selection elsewhere in the UI for this deployment.

```yaml
# Example within a deployment's select-tokens list:
select-tokens:
  - key: token1 # Must match a defined token key
    name: Optional Display Name for Token 1
    description: Optional description for selection context
  - key: token2
    # Uses default token name/symbol from token definition
```

### Fields

* `key`
  * **Required**: Yes
  * **Description**: The key referencing a token defined in the top-level `tokens` section.
* `name`
  * **Required**: No
  * **Description**: An optional override for the token's display name specifically within this selection context. If omitted, the builder would likely use the `label` or `symbol` from the main token definition.
* `description`
  * **Required**: No
  * **Description**: An optional description providing context for why this token is selectable here.

## Validation Object

The `validation` object is used within `Deposit Item` and `Field Item` to specify rules that the user's input must adhere to. The available validation rules depend on the nature of the validation type.

```yaml
# Example for a number field validation:
validation:
  type: number
  minimum: 0
  maximum: 10000
  multipleOf: 0.01

# Example for a string field validation:
validation:
  type: string
  minLength: 1
  maxLength: 255

# Example for a deposit validation (implicitly number):
validation:
  minimum: 0.000001
  maximum: 1000

# Example for a boolean field validation:
validation:
  type: boolean
```

### Common Fields

* `type`
  * **Required**: Yes (for `Field Item`)
  * **Description**: Specifies the expected semantic type of the input. Must be either `number`, `string`, or `boolean`.
  * **Note**: For `Deposit Item`, the type is implicitly `number` and this field can be omitted.

### Number Validation Fields

These fields apply when `type` is `"number"`.

* `minimum`
  * **Required**: No
  * **Description**: The minimum allowed value (inclusive). Input must be greater than or equal to this value.
* `exclusiveMinimum`
  * **Required**: No
  * **Description**: The minimum allowed value (exclusive). Input must be greater than this value.
* `maximum`
  * **Required**: No
  * **Description**: The maximum allowed value (inclusive). Input must be less than or equal to this value.
* `exclusiveMaximum`
  * **Required**: No
  * **Description**: The maximum allowed value (exclusive). Input must be less than this value.
* `multipleOf`
  * **Required**: No
  * **Description**: The input value must be a multiple of this number. For example, if `multipleOf` is `0.01`, then `1.02` is valid, but `1.025` is not.

### String Validation Fields

These fields apply when `type` is `"string"`.

* `minLength`
  * **Required**: No
  * **Description**: The minimum allowed length of the string (inclusive). The string length must be greater than or equal to this value.
* `maxLength`
  * **Required**: No
  * **Description**: The maximum allowed length of the string (inclusive). The string length must be less than or equal to this value.

### Boolean Validation Fields

These fields apply when `type` is `"boolean"`.

* No additional validation fields are available for boolean type. Field value must be supplied as a boolean when using this type.
