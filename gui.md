# GUI Configuration (`gui`)

## Introduction

The `gui` section within the YAML configuration provides metadata specifically tailored for building user interfaces related to Rainlang strategies. It acts as a bridge between the core technical definition of deployments, tokens, orders, etc., and their presentation to an end-user in a graphical application.

This section dictates how strategy deployments are named, described, and how their configurable parameters (bindings, deposits, token selections) should be rendered as interactive UI elements. It allows for user-friendly labels, descriptions, default values, and predefined options (presets) to simplify the user experience.

**Note**: While the underlying YAML parser treats all scalar values initially as strings, the application expects these strings to conform to the specified semantic types (e.g., `"true"` or `"false"` for Boolean fields) for successful processing. The types listed below refer to these expected semantic types.

## Top-Level `gui` Object

The root of the GUI configuration is the `gui:` key. It contains general information about the strategy or set of deployments described in the file.

```yaml
gui:
  name: Fixed limit
  description: Fixed limit order strategy
  short-description: Buy WETH with USDC on Base.
  deployments:
    # ... deployment configurations ...
```

### Fields

* `name`
  * **Type**: String
  * **Required**: Yes
  * **Description**: The primary, human-readable name for the overall strategy or configuration presented in the GUI.
* `description`
  * **Type**: String
  * **Required**: Yes
  * **Description**: A more detailed description of the strategy or configuration, intended for display in the GUI.
* `short-description`
  * **Type**: String
  * **Required**: No
  * **Description**: An optional, concise description, potentially used in contexts where space is limited (e.g., list views, tooltips).
* `deployments`
  * **Type**: Map
  * **Required**: Yes
  * **Description**: Contains the specific UI configurations for one or more named deployments. See the [Deployments Map](#deployments-map-deployments) section for details.

## Deployments Map (`deployments`)

The `deployments` field under `gui` holds a map where each key is the name of a deployment (which must correspond to a deployment defined elsewhere in the configuration, e.g., under the top-level `deployments:` key) and the value is an object defining the GUI configuration specific to that deployment.

```yaml
gui:
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
gui:
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
  * **Type**: String
  * **Required**: Yes
  * **Description**: The name of this specific deployment variation as it should appear in the GUI.
* `description`
  * **Type**: String
  * **Required**: Yes
  * **Description**: A detailed description for this specific deployment variation.
* `short-description`
  * **Type**: String
  * **Required**: No
  * **Description**: An optional, concise description for this deployment variation.
* `deposits`
  * **Type**: List of [Deposit Items](#deposit-item)
  * **Required**: Yes
  * **Description**: Defines the tokens users can deposit into the strategy and provides UI hints like presets.
* `fields`
  * **Type**: List of [Field Items](#field-item)
  * **Required**: Yes
  * **Description**: Defines the user-configurable parameters (bindings) for the strategy, including labels, descriptions, presets, and defaults.
* `select-tokens`
  * **Type**: List of [Select Token Items](#select-token-item)
  * **Required**: No
  * **Description**: Defines specific tokens that might be selectable in other parts of the UI for this deployment (e.g., choosing an output token if the strategy supports it).

## Deposit Item

Each item in the `deposits` list defines a token that can be deposited and optional preset amounts for the UI.

```yaml
# Example within a deployment's deposits list:
deposits:
  - token: token1 # Must match a defined token key
    presets:
      - "0"
      - "10"
      - "100"
      - "1000"
      - "10000"
  - token: token2
    # No presets defined for this token
```

### Fields

* `token`
  * **Type**: String
  * **Required**: Yes
  * **Description**: The key referencing a token defined in the top-level `tokens` section. This specifies which token the deposit configuration applies to.
* `presets`
  * **Type**: List of Strings
  * **Required**: No
  * **Description**: An optional list of suggested deposit amounts (as strings) to display as quick options in the UI. The UI will typically show these alongside the primary deposit input field.

## Field Item

Each item in the `fields` list defines a user-configurable input field, mapping it to an underlying Rainlang binding.

```yaml
# Example within a deployment's fields list:
fields:
  - binding: binding-1 # The corresponding binding in the Rainlang source
    name: Field 1 name
    description: Field 1 description
    presets:
      # ... preset items for this field ...
    default: some-default-value
  - binding: binding-2
    name: Field 2 name
    description: Field 2 description
    # No presets for this field
    show-custom-field: true # Explicitly allow custom input
```

### Fields

* `binding`
  * **Type**: String
  * **Required**: Yes
  * **Description**: The name of the binding in the associated Rainlang source code that this field provides the value for.
* `name`
  * **Type**: String
  * **Required**: Yes
  * **Description**: The human-readable label displayed for this input field in the GUI.
* `description`
  * **Type**: String
  * **Required**: No
  * **Description**: An optional, longer description or help text displayed for this field, potentially as a tooltip or helper text.
* `presets`
  * **Type**: List of [Preset Items](#preset-item)
  * **Required**: No
  * **Description**: An optional list of predefined values the user can select for this field.
* `default`
  * **Type**: String
  * **Required**: No
  * **Description**: An optional default value (as a string) to pre-populate the input field with.
* `show-custom-field`
  * **Type**: Boolean
  * **Required**: No
  * **Description**: Controls whether the user is presented with a free-form input field in addition to any defined presets. If `false`, the user might only be able to select from the presets. If `true` or omitted, a custom input field is typically shown.

## Preset Item

Each item in a field's `presets` list defines a single predefined option for that field.

```yaml
# Example within a field's presets list:
presets:
  - name: Preset 1 Name # Optional name for the preset
    value: "0x1234567890abcdef1234567890abcdef12345678"
  - value: "false" # Preset without an explicit name
  - name: Preset 3 Name
    value: "some-string"
```

### Fields

* `name`
  * **Type**: String
  * **Required**: No
  * **Description**: An optional label for the preset option shown in the UI (e.g., in a dropdown or radio button list). If omitted, the `value` itself might be displayed.
* `value`
  * **Type**: String
  * **Required**: Yes
  * **Description**: The actual value (as a string) that will be used for the binding if this preset is selected.

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
  * **Type**: String
  * **Required**: Yes
  * **Description**: The key referencing a token defined in the top-level `tokens` section.
* `name`
  * **Type**: String
  * **Required**: No
  * **Description**: An optional override for the token's display name specifically within this selection context. If omitted, the GUI would likely use the `label` or `symbol` from the main token definition.
* `description`
  * **Type**: String
  * **Required**: No
  * **Description**: An optional description providing context for why this token is selectable here.

