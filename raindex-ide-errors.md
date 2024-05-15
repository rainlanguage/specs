## Background
@thedavidmeister and I had a conversation about the handling of errors on the "Add Order" page of our app, intially triggered by #640. Our goal is to streamline how errors are reported and handled, ensuring a more intuitive user experience and clearer error visibility.

## Current Issues
1. **Error visibility and context**: Errors are currently shown inline, but it’s not always clear which scenario they belong to, leading to confusion.
2. **Redundant error reporting**: Errors appear even when no scenario is selected, cluttering the interface and reducing usability.
3. **Unclear error states**: Users encounter red squiggles and errors without understanding the context, especially with elided bindings.

## Proposed Changes
These will have depdencies in our other repos that we can split out into separate issues when needed.

### Error Display Mechanism
- **Scenario-based error tabs**: Introduce tabs in the error panel for different scenarios. Users can select a tab to view errors specific to a scenario.
- **Contextual error display**: Inline errors should only be displayed for the selected scenario. If no scenario is selected, no errors should be displayed.

### Error pipeline
We'll have a 3 step pipeline for errors. All errors in the each step must be resolved before the app will show errors in the next step.

1. **Config errors**: Display errors for the front matter config.
2. **Dotrain (composition) errors**: Display only errors that prevent composition.
3. **Onchain parser errors**: These are errors like unknown words, syntax errors, etc.

For example, errors related to elided bindings would need to be resolved for a scenario before any syntax errors would display.

### User Experience Improvements
- **Clear instructions**: Provide clear instructions in the UI for resolving errors, including specifying scenarios and deployments.
- **Dynamic error updates**: Automatically update the error panel when the user changes the scenario or deployment context.
- **Error-free initial state**: Ensure that users do not see any errors when no scenario or deployment is selected.

## Steps to Implement
1. **Focus Dotrain**: Strip back dotrain to only produce composition erorrs
2. **Update error reporting logic**: Adjust the logic to follow the pipeline as above.
3. **Create a new error panel**: Implement tabs for different scenarios and ensure errors are contextually displayed based on the selected scenario.

## Expected Outcomes
- Improved clarity and usability of error reporting on the "Add Order" page.
- Reduction in user confusion due to better contextualization of errors.
- Streamlined process for resolving composition and syntax errors.)
