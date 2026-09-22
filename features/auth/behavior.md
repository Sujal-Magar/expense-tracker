# Behavior Specification: Authentication and Identity

## 1. Sliding Panel Navigation & Transitions

- The authentication module presents a unified dual-state card with smooth CSS transitions:
  - **State 1 (Sign Up View)**:
    - Left emerald panel shows welcome message and `"SIGN IN"` button.
    - Right form area displays registration inputs.
    - Clicking `"SIGN IN"` triggers a sliding animation shifting the colored panel to the right, revealing State 2.
  - **State 2 (Sign In View)**:
    - Left form area displays login inputs.
    - Right emerald panel shows invitation copy and `"SIGN UP"` button.
    - Clicking `"SIGN UP"` reverses the sliding animation back to State 1.

## 2. Registration (Sign Up) Workflow

1. User enters their Name, Email, Password, and Confirm Password into the form.
2. Clicking the trailing eye icon toggles mask visibility on password values.
3. User clicks **"SIGN UP"**:
   - If fields are empty or passwords do not match: Form indicates invalid fields inline.
   - If registration succeeds: Authenticates user, establishes session cookie, and navigates immediately to `/dashboard`.
   - If email is already in use: Surfaces an error toast: `"An account with this email already exists."`
4. Alternatively, clicking the **Google icon** opens the third-party OAuth popup modal.

## 3. Login (Sign In) Workflow

1. User inputs their registered Email and Password.
2. User clicks **"SIGN IN"**:
   - Submits credentials for authentication.
   - If invalid: Displays inline error `"Invalid email or password"` and shakes the credential input boxes.
   - If valid: Persists session, sets user state, and redirects to `/dashboard`.
3. Clicking **"Forgot your password?"** directs the user to the account recovery workflow.

## 4. Protected Route & Session Handling

- Unauthenticated requests accessing `/dashboard`, `/transactions`, `/budget`, `/goals`, `/reports`, or `/profile` redirect to the sign-in screen.
- Active session persists in secure storage until explicit logout or token expiry.
