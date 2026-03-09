# GUI Configuration (`gui`)

## Introduction

The `gui` section within the YAML configuration provides metadata specifically tailored for building user interfaces related to Rainlang strategies. It acts as a bridge between the core technical definition of deployments, tokens, orders, etc., and their presentation to an end-user in a graphical application.

This section dictates how strategies are named, described, and how their configurable parameters (bindings, deposits, token selections) should be rendered as interactive UI elements. It allows for user-friendly labels, descriptions, default values, and predefined options (presets) to simplify the user experience.

## Top-Level `gui` Object

The root of the GUI configuration is the `gui:` key. It contains general information about the strategy and defines reusable field definitions and variations.

```yaml
gui:
  name: Fixed limit
  description: Fixed limit order strategy
  short-description: Buy WETH with USDC on Base.
  field-definitions:
    # ... field definitions ...
  variations:
    # ... variation configurations ...
```

### Fields

* `name`
  * **Required**: Yes
  * **Description**: The primary, human-readable name for the overall strategy presented in the GUI.
* `description`
  * **Required**: Yes
  * **Description**: A more detailed description of the strategy, intended for display in the GUI.
* `short-description`
  * **Required**: No
  * **Description**: An optional, concise description, potentially used in contexts where space is limited (e.g., list views, tooltips).
* `field-definitions`
  * **Required**: No
  * **Description**: A map of reusable field definitions that can be referenced by variations. See the [Field Definitions](#field-definitions) section for details.
* `variations`
  * **Required**: Yes
  * **Description**: A map containing the specific UI configurations for different strategy variations. See the [Variations Map](#variations-map) section for details.

## Field Definitions

The `field-definitions` field under `gui` holds a map where each key is a field identifier and the value defines the complete UI configuration for that field. These definitions are referenced by variations to avoid duplication.

```yaml
gui:
  field-definitions:
    time-per-amount-epoch:
      name: Budget period (in seconds)
      description: The budget is spent over this time period
      show-custom-field: true
      default: 86400
      presets:
        - name: Per day (86400)
          value: 86400
        - name: Per week (604800)
          value: 604800
      validation:
        type: number
        minimum: 60
    amount-per-epoch:
      name: Budget (${order.outputs.0.token.symbol} per period)
      description: The amount to spend each budget period
      default: 0
      validation:
        type: number
        minimum: 0
```

### Field Definition Structure

Each field definition contains the complete configuration for a user-configurable input field. The field identifier (the key in the map) corresponds to the binding name in scenarios.

#### Fields

* `name`
  * **Required**: Yes
  * **Description**: The human-readable label displayed for this input field in the GUI. Can include template variables like `${order.outputs.0.token.symbol}`.
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
  * **Description**: An optional [Validation Object](#validation-object) defining validation rules for the field's value.
* `show-custom-field`
  * **Required**: No
  * **Description**: Controls whether the user is presented with a free-form input field in addition to any defined presets. When set to `false`, the user can only select from presets. Defaults to `true`.

## Variations Map

The `variations` field under `gui` holds a map where each key is a variation identifier and the value defines the specific UI configuration for that variation of the strategy.

```yaml
gui:
  variations:
    standard:
      name: Standard DCA
      description: Deploy a standard dollar-cost averaging order
      fields:
        - time-per-amount-epoch
        - amount-per-epoch
      deposits:
        - token: output
      select-tokens:
        - key: output
          name: Token to Sell
        - key: input
          name: Token to Buy
      networks:
        42161: arbitrum
        14: flare
    special-variant:
      name: Special Variant
      description: A specialized version of the strategy
      fields:
        - time-per-amount-epoch
        - special-field
      deposits:
        - token: special-token
      networks:
        14: special-deployment
```

### Variation Configuration

Each variation defines which fields to show, which tokens can be deposited, and how it maps to different networks and deployments.

#### Fields

* `name`
  * **Required**: Yes
  * **Description**: The name of this variation as it should appear in the GUI.
* `description`
  * **Required**: Yes
  * **Description**: A detailed description for this specific variation.
* `fields`
  * **Required**: Yes
  * **Description**: A list of field identifiers referencing entries in `field-definitions`. The order in the list determines the display order in the GUI.
* `deposits`
  * **Required**: Yes
  * **Description**: A list of [Deposit Items](#deposit-item) that defines the tokens users can deposit into the strategy.
* `select-tokens`
  * **Required**: No
  * **Description**: A list of [Select Token Items](#select-token-item) that defines tokens that need to be selected by the user. Only include tokens that require user selection; hardcoded tokens should be defined directly in the order.
* `networks`
  * **Required**: Yes
  * **Description**: A map where keys are chain IDs (as integers) and values are deployment identifiers (referencing keys in the top-level `deployments` map). This determines which deployment to use for each network when this variation is selected.

## Deposit Item

Each item in the `deposits` list defines a token that can be deposited and optional preset amounts for the UI.

```yaml
deposits:
  - token: output # References one of the order's inputs/outputs
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
  * **Description**: The key referencing a token. This corresponds to a token defined in the order's inputs/outputs.
* `presets`
  * **Required**: No
  * **Description**: An optional list of suggested deposit amounts to display as quick options in the UI. The UI will typically show these alongside the primary deposit input field.
* `validation`
  * **Required**: No
  * **Description**: An optional [Validation Object](#validation-object) defining validation rules for the deposit amount. For deposits, this will always use number validation rules. See the [Validation Object](#validation-object) section for details on available number validation fields like `minimum`, `exclusiveMinimum`, `maximum`, `exclusiveMaximum`, and `multipleOf`.

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
  * **Description**: An optional override for the token's display name specifically within this selection context. If omitted, the GUI would likely use the `label` or `symbol` from the main token definition.
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
