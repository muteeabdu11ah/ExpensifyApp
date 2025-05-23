# Creating and Using Forms

This document lists specific guidelines for using our Form component and general forms guidelines.

## General Form UI/UX

### Inputs
Any form input needs to be wrapped in [InputWrapper](https://github.com/Expensify/App/blob/029d009731dcd3c44cd1321672b9672ef0d3d7d9/src/components/Form/InputWrapper.js) and passed as `InputComponent` property, additionally, it's necessary to pass an unique `inputID`. All other props of the input can be passed as `InputWrapper` props.

```jsx
<InputWrapper
    // `InputWrapper` required props
    InputComponent={TextInput}
    inputID={INPUT_IDS.UNIQUE_INPUT_ID}
    // `TextInput` specific props
    placeholder="Text input placeholder"
    label="Text input label"
    shouldSaveDraft
/>
```

### Labels, Placeholders, & Hints

Labels are required for each input and should clearly mark the field. Optional text may appear below a field when a hint, suggestion, or context feels necessary. If validation fails on such a field, its error should clearly explain why without relying on the hint. Inline errors should always replace the microcopy hints. Placeholders should not be used as it’s customary for labels to appear inside form fields and animate them above the field when focused.

![hint](https://user-images.githubusercontent.com/22219519/156266779-72deaf42-832c-453c-a5c2-1b2073b8b3b7.png)

Labels and hints are enabled by passing the appropriate props to each input:

```jsx
<InputWrapper
    InputComponent={TextInput}
    label="Value"
    hint="Hint text goes here"
/>
```

### Native Keyboards

We should always set people up for success on native platforms by enabling the best keyboard for the type of input we’re asking them to provide. See [inputMode](https://reactnative.dev/docs/textinput#inputmode) in the React Native documentation.

We have a list of input modes [defined](https://github.com/Expensify/App/blob/9418b870515102631ea2156b5ea253ee05a98ff1/src/CONST.js#L765-L774) and should be used like so:

```jsx
<InputWrapper
    InputComponent={TextInput}
    inputMode={CONST.INPUT_MODE.NUMERIC}
/>
```

We also have [keyboardType](https://github.com/Expensify/App/blob/9418b870515102631ea2156b5ea253ee05a98ff1/src/CONST.js#L760-L763) and should be used for specific use cases when there is no `inputMode` equivalent of the value exist, and should be used like so:

```jsx
<InputWrapper
    InputComponent={TextInput}
    keyboardType={CONST.KEYBOARD_TYPE.ASCII_CAPABLE}
/>
```


### Autofill Behavior

Forms should autofill information whenever possible i.e. they should work with browsers and password managers auto-complete features.

As a best practice, we should avoid asking for information we can get via other means e.g. asking for City, State, and Zip if we can use Google Places to gather information with the least amount of hassle to the user.

Browsers use the name prop to autofill information into the input. Here's a [reference](https://developers.google.com/web/fundamentals/design-and-ux/input/forms#recommended_input_name_and_autocomplete_attribute_values) for available values for the name prop.

```jsx
<InputWrapper
    InputComponent={TextInput}
    name="fname"
/>
```

### Focus and Tab Behavior

All forms should define an order in which the inputs should be filled out, and using tab / shift + tab to navigate through the form should traverse the inputs in that order/reversed order, respectively. In most cases, this can be achieved by composition, i.e. rendering the components in the correct order. If we come across a situation where composition is not enough, we can:

1. Create a local tab index state
2. Assign a tab index to each form input
3. Add an event listener to the page/component we are creating and update the tab index state on tab/shift + tab key press
4. Set focus to the input with that tab index.

Additionally, pressing the enter key on any focused field should submit the form.

Note: This doesn't apply to the multiline fields. To keep the browser behavior consistent, pressing enter on the multiline should not be intercepted. It should follow the default browser behavior (such as adding a newline).

### Modifying User Input on Change

User input that may include optional characters (e.g. (, ), - in a phone number) should never be restricted on input, nor be modified or formatted on blur. This type of input jacking is disconcerting and makes things feel broken.

Instead, we will format and clean the user input internally before using the value (e.g. making an API request where the user will never see this transformation happen). Additionally, users should always be able to copy/paste whatever characters they want into fields.

To give a slightly more detailed example of how this would work with phone numbers, we should:

1. Allow any character to be entered in the field.
2. On blur, strip all non-number characters (with the exception of + if the API accepts it) and validate the result against the E.164 regex pattern we use for a valid phone. This change is internal and the user should not see any changes. This should be done in the validate callback passed as a prop to Form.
3. On submit, repeat validation and submit with the clean value.

### Form Drafts

Form inputs will NOT store draft values by default. This is to avoid accidentally storing any sensitive information like passwords, SSN or bank account information. We need to explicitly tell each form input to save draft values by passing the `shouldSaveDraft` prop to the input. Saving draft values is highly desirable and we should always try to save draft values. This way when a user continues a given flow they can easily pick up right where they left off if they accidentally exited a flow. Inputs with saved draft values [will be cleared when a user logs out](https://github.com/Expensify/App/blob/aa1f0f34eeba5d761657168255a1ae9aebdbd95e/src/libs/actions/SignInRedirect.js#L52) (like most data). Additionally, we should clear draft data once the form is successfully submitted by calling `Onyx.set(ONYXKEY.FORM_ID, null)` in the onSubmit callback passed to Form.

```jsx
<InputWrapper
    shouldSaveDraft
/>
```

## Form Validation and Error handling

### Validate on Blur, on Change and Submit

Each individual form field that requires validation will have its own validate test defined. When the form field loses focus (blur) we will run that validate test and show feedback. A blur on one field will not cause other fields to validate or show errors unless they have already been blurred.

Once a user has “touched” an input, i.e. blurred the input, we will also start validating that input on change when the user goes back to editing it.

All form fields will additionally be validated when the form is submitted. Although we are validating on blur this additional step is necessary to cover edge cases where forms are auto-filled or when a form is submitted by pressing enter (i.e. there will be only a ‘submit’ event and no ‘blur’ event to hook into).

The Form component takes care of validation internally and the only requirement is that we pass a validate callback prop. The validate callback takes in the input values as argument and should return an object with shape `{[inputID]: errorMessage}`.

Here's an example for a form that has two inputs, `routingNumber` and `accountNumber`:

```js
function validate(values) {
    const errors = {};
    if (!values.routingNumber) {
        errors.routingNumber = CONST.ERRORS.ROUTING_NUMBER;
    }
    if (!values.accountNumber) {
        errors.accountNumber = CONST.ERRORS.ACCOUNT_NUMBER;
    }
    return errors;
}
```

When more than one method is used to validate the value, the `addErrorMessage` function from `ErrorUtils` should be used. Here's an example for a form with a field with multiple validators for `firstName` input:

```js
function validate(values) {
        let errors = {};

        if (!ValidationUtils.isValidDisplayName(values.firstName)) {
            errors = ErrorUtils.addErrorMessage(errors, 'firstName', 'personalDetails.error.hasInvalidCharacter');
        }

        if (ValidationUtils.doesContainReservedWord(values.firstName, CONST.DISPLAY_NAME.RESERVED_NAMES)) {
            errors = ErrorUtils.addErrorMessage(errors, 'firstName', 'personalDetails.error.containsReservedWord');
        }

        if (!ValidationUtils.isValidDisplayName(values.lastName)) {
            errors.lastName = 'personalDetails.error.hasInvalidCharacter';
        }

        return errors;
    }
```

For a working example, check [Form story](https://github.com/Expensify/App/blob/aa1f0f34eeba5d761657168255a1ae9aebdbd95e/src/stories/Form.stories.js#L63-L72)

### Character Limits

If a field has a character limit, we should give that field a max limit. This is done by passing the character limit validation in the validate function.

Here's an example for a form that has one input `name`, and has character limit of 100:

```js
function validate(values) {
    const errors = {};
    if (values.name.length > 100) {
        ErrorUtils.addErrorMessage(errors, 'name', translate('common.error.characterLimitExceedCounter', {length: values.name.length, limit: 100}));
    }
    return errors;
}
```

> [!NOTE]
>  We shouldn't place a max limit on a field if the entered value can be formatted. eg: Phone number.
> The phone number can be formatted in different ways.
> 
> - 2109400803
> - +12109400803
> - (210)-940-0803

> [!NOTE]
>  If we want to count number of Unicode code points instead of the number of UTF-16 code units, we should use the spread syntax.
> Example - `[...newCategoryName].length`

### Highlight Fields and Inline Errors

Individual form fields should be highlighted with a red error outline and present supporting inline error text below the field. Error text will be required for all required fields and optional fields that require validation. This will keep our error handling consistent and ensure we put in a good effort to help the user fix the problem by providing more information than less.

![error](https://user-images.githubusercontent.com/22219519/156267035-af40fe93-da27-4e16-bc55-b7cd40b0f1f2.png)

### Multiple Types of Errors for Individual Fields

Individual fields should support multiple messages depending on validation e.g. a date could be badly formatted or outside of an allowable range. We should not only say “Please enter a valid date” and instead always tell the user why something is failing if we can. The Form component supports an infinite number of possible error messages per field and they are displayed simultaneously if multiple validations fail.

### Form Alerts

When any form field fails to validate in addition to the inline error below a field, an error message will also appear inline above the submit button indicating that some fields need to be fixed. A “fix the errors” link will scroll the user to the first input that needs attention and focus on it (putting the cursor at the end of the existing value). By default, on form submit and when tapping the “fix the errors” link we should scroll the user to the first field that needs their attention.

![form-alert](https://user-images.githubusercontent.com/22219519/156267105-861fbe81-32cc-479d-8eff-3760bd0585b1.png)

### Handling Server Errors

Server errors related to form submission should appear in the Form Alert above the submit button. They should not appear in growls or other kinds of alerts. Additionally, as best practice moving forward server errors should never solely do the work that frontend validation can also do. This means that any error that can be validated in the frontend should be validated in the frontend and backend.

Note: This is not meant to suggest that we should avoid validating in the backend if the frontend already validates.

Note: There are edge cases where some server errors will inevitably relate to specific fields in a form with other fields unrelated to that error. We had trouble coming to a consensus on exactly how this edge case should be handled (e.g. show inline error, clear on blur, etc). For now, we will show the server error in the form alert and not inline (so the “fix the errors” link will not be present). In those cases, we will still attempt to inform the user which field needs attention, but not highlight the input or display an error below the input. We will be on the lookout for our first validation in the server that could benefit from being tied to a specific field and try to come up with a unified solution for all errors.

## Form Submission
### Submit Button Disabling

Submit buttons shall not be disabled or blocked from being pressed in most cases. We will allow the user to submit a form and point them in the right direction if anything needs their attention.

The only time we won’t allow a user to press the submit button is when we have submitted the form and are waiting for a response (e.g. from the API). In this case, we will show a loading indicator and additional taps on the submit button will have no effect. This is handled by the Form component and will also ensure that a form cannot be submitted multiple times.

## Using Form

The example below shows how to use [FormProvider](https://github.com/Expensify/App/blob/029d009731dcd3c44cd1321672b9672ef0d3d7d9/src/components/Form/FormProvider.js) and [InputWrapper](https://github.com/Expensify/App/blob/029d009731dcd3c44cd1321672b9672ef0d3d7d9/src/components/Form/InputWrapper.js) in our app. You can also refer to [Form.stories.js](https://github.com/Expensify/App/blob/c5a84e5b4c0b8536eed2214298a565e5237a27ca/src/stories/Form.stories.js) for more examples.

```jsx
function validate(values) {
    const errors = {};
    if (!values.routingNumber) {
        errors.routingNumber = 'Please enter a routing number';
    }
    if (!values.accountNumber) {
        errors.accountNumber = 'Please enter an account number';
    }
    return errors;
}

function onSubmit(values) {
    setTimeout(() => {
        alert(`Form submitted!`);
        FormActions.setIsLoading('TestForm', false);
    }, 1000);
}

<FormProvider
    formID="testForm"
    submitButtonText="Submit"
    validate={this.validate}
    onSubmit={this.onSubmit}
>
    // Wrapping InputWrapper in a View to show that Form inputs can be nested in other components
    <View>
        <InputWrapper
            InputComponent={TextInput}
            label="Routing number"
            inputID={INPUT_IDS.ROUTING_NUMBER}
            maxLength={8}
            shouldSaveDraft
        />
    </View>
    <InputWrapper
        InputComponent={TextInput}
        label="Account number"
        inputID={INPUT_IDS.ACCOUNT_NUMBER}
        containerStyles={[styles.mt4]}
    />
</FormProvider>
```

`FormProvider` also works with inputs nested in a custom component, e.g. [AddressForm](https://github.com/Expensify/App/blob/86579225ff30b21dea507347735259637a2df461/src/pages/ReimbursementAccount/AddressForm.js). The only exception is that the nested component shouldn't be wrapped around any HoC and all inputs in the component needs to be wrapped with `InputWrapper`.

```jsx
const BankAccountForm = () => (
    <>
        <View>
            <InputWrapper
                InputComponent={TextInput}
                label="Routing number"
                inputID={INPUT_IDS.ROUTING_NUMBER}
                maxLength={8}
                shouldSaveDraft
            />
        </View>
        <InputWrapper
            InputComponent={TextInput}
            label="Account number"
            inputID={INPUT_IDS.ACCOUNT_NUMBER}
            containerStyles={[styles.mt4]}
        />
    </>
);

// ...
<FormProvider
    formID="testForm"
    submitButtonText="Submit"
    validate={this.validate}
    onSubmit={this.onSubmit}
>
    <BankAccountForm />
</FormProvider>
```

### Props provided to Form inputs

The following prop is available to form inputs:

- inputID: An unique identifier for the input.
- shouldSaveDraft: Saves a draft of the input value.
- defaultValue: The initial value of the input.
- value: The value to show for the input.
- onValueChange: A callback that is called when the input's value changes.

InputWrapper component will automatically provide the following props to any input with the inputID prop.

- ref: A React ref that must be attached to the input.
- value: The input value.
- errorText: The translated error text that is returned by validate for that specific input.
- onBlur: An onBlur handler that calls validate.
- onTouched: An onTouched handler that marks the input as touched.
- onInputChange: An onChange handler that saves draft values and calls validate for that input (inputA). Passing an inputID as a second param allows inputA to manipulate the input value of the provided inputID (inputB).
- onFocus: An onFocus handler that marks the input as focused.

## Dynamic Form Inputs

It's possible to conditionally render inputs (or more complex components with multiple inputs) inside a form. For example, an IdentityForm might be nested as input for a Form component.
In order for Form to track the nested values properly, each field must have a unique identifier. It's not safe to use an index because adding or removing fields from the child Form component will not update these internal keys. Therefore, we will need to define keys and dynamically access the correlating child form data for validation/submission.

To generate these unique keys, use `Str.guid()`.

An example of this can be seen in the [ACHContractStep](https://github.com/Expensify/App/blob/f2973f88cfc0d36c0dbe285201d3ed5e12f29d87/src/pages/ReimbursementAccount/ACHContractStep.js), where each key is stored in an array in state, and IdentityForms are dynamically rendered based on which keys are present in the array.

### Safe Area Padding

Any `FormProvider.tsx` that has a button at the bottom. If the `<FormProvider>` is inside a `<ScreenWrapper>`, the bottom safe area inset is handled automatically (`includeSafeAreaPaddingBottom` needs to be set to `true`, but its the default).
If you have custom requirements and can't use `<ScreenWrapper includeSafeAreaPaddingBottom={true}>`, you can use the `useSafeAreaPaddings()` hook:

```jsx
const { paddingTop, paddingBottom, safeAreaPaddingBottomStyle } = useSafeAreaPaddings();

<View styles={[safeAreaPaddingBottomStyle, styles.pb5]}>
    <FormProvider>
        {...}
    </FormProvider>
</View>
```

### Handling nested Pickers in Form

In case there's a nested Picker in Form, we should pass the props below to Form, as needed:

#### Enable ScrollContext

Pass the `scrollContextEnabled` prop to enable scrolling up when Picker is pressed, making sure the Picker is always in view and doesn't get covered by virtual keyboards for example.

#### Enable Form to Scroll to the End

Pass the `shouldScrollToEnd` prop to automatically scroll to the bottom when the form is opened. Ensure that the scrolling stops at the appropriate limit so that the button remains visible above the keypad.
