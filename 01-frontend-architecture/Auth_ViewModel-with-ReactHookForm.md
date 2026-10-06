# React Hook Form in `AuthViewModel` & `SignInPage`

This guide explains how **React Hook Form** works in this project, breaking down every word, concept, and line of logic in [`AuthViewModel.ts`](file:///Users/shraddhadongol/invoice-app/src/view-models/auth/AuthViewModel.ts) and its usage in [`SignInPage.tsx`](file:///Users/shraddhadongol/invoice-app/src/views/pages/SignInPage.tsx).

---

## 1. What is React Hook Form (RHF)?

In standard React, handling forms often requires managing state for each input (`useState`), tracking input changes (`onChange`), managing blur events, and tracking errors manually. This causes:
- Excessive re-renders on every keystroke.
- Lots of boilerplate code.

**React Hook Form** solves this by:
- Using **uncontrolled components** under the hood (accessing DOM inputs via `ref`).
- Only re-rendering when necessary (e.g., when validation errors change or form submission state changes).
- Providing built-in validation, error tracking, and submission handling.

---

## 2. Line-by-Line Breakdown: [`AuthViewModel.ts`](file:///Users/shraddhadongol/invoice-app/src/view-models/auth/AuthViewModel.ts)

### A. Imports
```typescript
import { useLoginMutation } from '../../slice/authApiSlice';
import { useForm } from 'react-hook-form';
```
- **`useLoginMutation`**: An RTK Query hook that executes an asynchronous `POST` request to log the user in.
- **`useForm`**: The primary custom hook from `react-hook-form` used to create and configure form instances.

---

### B. TypeScript Interface
```typescript
interface LoginFormData {
  email: string;
  password: string;
}
```
- Defines the shape of the form's data values.
- Ensures TypeScript knows that our form strictly contains two fields: `email` (string) and `password` (string).

---

### C. Initializing the Form Hook
```typescript
const form = useForm<LoginFormData>({
  defaultValues: {
    email: '',
    password: '',
  },
  mode: 'onBlur',
});
```

Here is what each word and option means:

1. **`useForm<LoginFormData>(...)`**:
   - Passes the `LoginFormData` type parameter to `useForm`. This gives you auto-complete and strict type checking when registering fields or reading errors.
2. **`defaultValues`**:
   - An object defining the initial values of each form field when the component mounts.
   - Initializing these prevents React's `"uncontrolled to controlled input"` console warnings.
3. **`mode: 'onBlur'`**:
   - Controls **when** validation is triggered.
   - By default (`mode: 'onSubmit'`), validation only runs when the user clicks submit.
   - `'onBlur'` means validation runs whenever the user unfocuses (tabs or clicks away from) an input field. Other modes include `'onChange'` (every keystroke), `'onSubmit'` (on submit only), and `'all'` (both blur and change).

---

### D. Custom Validation Logic
```typescript
const validateLogin = (email: string, password: string): string | null => {
  if (!email.trim()) {
    return "Email is required";
  }

  const emailRegex = /^[^\s@]+@[^\s@]+\.[^\s@]+$/;
  if (!emailRegex.test(email)) {
    return "Please enter a valid email";
  }

  if (!password.trim()) {
    return "Password is required";
  }

  if (password.length < 6) {
    return "Password must be at least 6 characters";
  }

  return null;
};
```
- A custom validation function checking:
  - If `email` is empty after stripping whitespaces (`!email.trim()`).
  - If `email` matches a regular expression pattern for valid emails (`emailRegex.test(...)`).
  - If `password` is empty or shorter than 6 characters.
- Returns an error message string or `null` if valid.

---

### E. Submission Handler Logic
```typescript
const handleLogin = async (data: LoginFormData) => {
  const validationError = validateLogin(data.email, data.password);

  if (validationError) {
    throw new Error(validationError);
  }

  try {
    const response = await login({ email: data.email, password: data.password }).unwrap();
  } catch (error) {
    console.error("Login failed:", error);
    throw error;
  }

  console.log("Login successful");
};
```
- **`data: LoginFormData`**: React Hook Form automatically passes the collected, validated form values (`{ email, password }`) to this function upon successful submission.
- **`login(...).unwrap()`**: An RTK Query helper that extracts the raw payload or throws the rejected error if the network request fails.

---

### F. Return Object
```typescript
return {
  form,
  onSubmit: form.handleSubmit(handleLogin),
  isLoading
};
```

- **`form`**: The entire form control object returned by `useForm`, which contains methods like `register`, `watch`, `setValue`, and properties like `formState`.
- **`form.handleSubmit(handleLogin)`**:
  - `handleSubmit` is a higher-order function provided by React Hook Form.
  - It intercepts the native browser submit event (`e.preventDefault()` is handled internally).
  - It runs all validations defined on registered inputs.
  - If validation **passes**, it calls `handleLogin(data)`.
  - If validation **fails**, it stops submission and populates `formState.errors` without calling `handleLogin`.
- **`isLoading`**: Indicates whether the RTK Query login mutation is currently in-flight.

---

## 3. How the View Consumes RHF: [`SignInPage.tsx`](file:///Users/shraddhadongol/invoice-app/src/views/pages/SignInPage.tsx)

### A. Destructuring Form Properties
```typescript
const { form, onSubmit, isLoading } = AuthViewModel();
const { register, formState: { errors } } = form;
```

1. **`register`**:
   - A function that connects an `<input>` element to React Hook Form.
   - It injects event handlers (`onChange`, `onBlur`), a `name`, and a `ref` into the input.
2. **`formState`**:
   - Contains information about the state of the form (e.g., `isSubmitting`, `isDirty`, `isValid`, `errors`).
3. **`errors`**:
   - An object holding active validation errors keyed by field name (`errors.email`, `errors.password`).

---

### B. Binding the Form Submit
```tsx
<form className="space-y-5" onSubmit={onSubmit}>
```
- Passes `onSubmit` (which is `form.handleSubmit(handleLogin)`) to the HTML `<form>` tag.

---

### C. Connecting Inputs with `register` & Displaying Errors
```tsx
<InputField
  id="email"
  label="Email"
  type="email"
  placeholder="you@example.com"
  showRequiredIndicator={true}
  {...register('email', {
    required: 'Email is required',
  })}
  error={errors.email?.message}
/>
```

Here is what happens under the hood:

1. **`register('email', { required: 'Email is required' })`**:
   - **First argument (`'email'`)**: Registers the field under the key `"email"` in form state.
   - **Second argument (Validation rules)**: Declares validation rules. Here, `required: 'Email is required'` means if the field is empty, React Hook Form will flag it as an error with the message `"Email is required"`.
   - **Spread operator (`{...register(...)}`)**: `register` returns an object with `{ name: 'email', onChange, onBlur, ref }`. Spreading this onto `<InputField />` wires the input directly to RHF.
2. **`error={errors.email?.message}`**:
   - Reads the error message for `email` from `formState.errors`.
   - If an error exists (e.g., due to blur or submit without input), the error message is displayed below the field.

---

## 4. Summary Flow Diagram

```text
User fills input ──► On Blur / Typing
                          │
                          ▼
                 React Hook Form validates
                          │
          ┌───────────────┴───────────────┐
          │                               │
       Invalid                          Valid
          │                               │
          ▼                               ▼
`errors[field].message`               No errors
shown in UI UI updates                User clicks "Sign In"
                                          │
                                          ▼
                               `form.handleSubmit()`
                                          │
                                          ▼
                                Calls `handleLogin(data)`
                                          │
                                          ▼
                                RTK Query `login()` called
```
