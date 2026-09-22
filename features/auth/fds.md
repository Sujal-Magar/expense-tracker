---
id: auth
title: Authentication and Identity
status: active
version: 1.0.0
owner: security-team
last_updated: 2026-09-22
coverage_target: 95
compliance_relevant: true
dependencies: []
changelog:
  - version: 1.0.0
    date: 2026-09-22
    summary: "Initial specification for FinTrack user authentication, email/password signup & login, Google OAuth SSO, and session management"
---

# Feature Specification: Authentication and Identity

## 1. Overview

The Authentication module provides the secure entry point for the FinTrack application. It enables new user registration (Sign Up), returning user authentication (Sign In), third-party federated identity through Google OAuth SSO, password recovery access, and active session credential management. Because every personal finance record must be bound to an authenticated identity, `auth` serves as the foundational root dependency across the platform.

## 2. Data Model (`AuthSession` & `AuthCredentials`)

### User Account Entity (`UserAuth`)

| Field          | Type          | Required | Description                                           |
| :------------- | :------------ | :------- | :---------------------------------------------------- |
| `id`           | UUID string   | Yes      | Unique identifier (Primary Key)                       |
| `name`         | string        | Yes      | User display name (e.g., "Piyush Kumar")              |
| `email`        | string        | Yes      | Unique, valid email address (normalized to lowercase) |
| `passwordHash` | string        | Yes      | Secure password hash (bcrypt / argon2)                |
| `provider`     | enum          | Yes      | Authentication provider (`"email"` or `"google"`)     |
| `googleId`     | string        | No       | Federated identifier for Google OAuth accounts        |
| `createdAt`    | ISO Timestamp | Yes      | Account creation timestamp                            |
| `updatedAt`    | ISO Timestamp | Yes      | Last account modification timestamp                   |

### Session Payload (`SessionToken`)

| Field          | Type          | Required | Description                                         |
| :------------- | :------------ | :------- | :-------------------------------------------------- |
| `accessToken`  | JWT string    | Yes      | Short-lived bearer token for authenticated requests |
| `refreshToken` | string        | Yes      | Long-lived secure token stored in HTTP-only cookie  |
| `userId`       | UUID string   | Yes      | Reference to `UserAuth.id`                          |
| `expiresAt`    | ISO Timestamp | Yes      | Access token expiration timestamp                   |

## 3. Functional Requirements

### REQ-AUTH-01: Sign Up (Registration Flow)

- Interface renders a two-panel card layout with a sliding emerald-to-dark-green overlay banner:
  - **Left Brand Panel**: Features the FinTrack logo, header text `"Welcome Back!"`, subtext `"Log in to manage your finances."`, and an outline/ghost action button `"SIGN IN"` that toggles to the login panel.
  - **Right Form Panel**: Titled `"Create Account"`, containing:
    - Single Sign-On button with Google icon (`G`).
    - Divider copy: `"or use your email for registration"`.
    - Input fields with icon adornments:
      - **Name**: Text input with user icon.
      - **Email**: Email input with mail icon.
      - **Password**: Password input with lock icon and trailing eye toggle button (show/hide password).
      - **Confirm Password**: Password input with lock icon and trailing eye toggle button.
    - Primary solid teal submit button labeled `"SIGN UP"`.
- On success: Automatically initializes user default profile and redirects user to `/dashboard`.

### REQ-AUTH-02: Sign In (Login Flow)

- Interface presents the inverted two-panel card layout:
  - **Left Form Panel**: Titled `"Sign in to FinTrack"`, containing:
    - Google SSO button (`G`).
    - Divider copy: `"or use your account"`.
    - Input fields:
      - **Email**: Text input with mail icon.
      - **Password**: Password input with lock icon and trailing eye toggle button.
    - Action link: `"Forgot your password?"`.
    - Primary solid teal submit button labeled `"SIGN IN"`.
  - **Right Brand Panel**: Features the FinTrack logo, header text `"Hello, Friend!"`, subtext `"Enter your personal details and start journey with us"`, and an outline/ghost action button `"SIGN UP"` that toggles to the sign-up panel.
- On success: Sets authentication session tokens and navigates user to `/dashboard`.

### REQ-AUTH-03: Google Single Sign-On (OAuth)

- Clicking the Google icon button (`G`) initiates standard OAuth 2.0 flow.
- If the email does not exist, registers a new user; if it exists, issues an authenticated session token.

### REQ-AUTH-04: Password Visibility Toggle

- Trailing eye icon on password and confirmation password fields toggles between obscured (`type="password"`) and plain text (`type="text"`).

### REQ-AUTH-05: Forgot Password Trigger

- Clicking the centered link `"Forgot your password?"` navigates to or opens the password reset request dialog.

## 4. Validation Rules

- `name`: Must be at least 2 characters long.
- `email`: Must be a syntactically valid email string (`name@domain.tld`).
- `password`: Must be at least 8 characters long, containing at least one digit and one special character.
- `confirmPassword`: Must exactly match the value entered in `password`.
- Duplicate registration: Registering with an existing email returns an HTTP 409 conflict error.

## 5. API / Interface Specification

### Authentication Endpoints

| API / Operation Name   | Method | Endpoint                       | Request Body                                   | Success Status / Response                          | Description                                     |
| :--------------------- | :----- | :----------------------------- | :--------------------------------------------- | :------------------------------------------------- | :---------------------------------------------- |
| `register`             | `POST` | `/api/v1/auth/signup`          | `name`, `email`, `password`, `confirmPassword` | `201 Created` (`user`, `accessToken`, `expiresIn`) | Registers new user account and creates session  |
| `login`                | `POST` | `/api/v1/auth/login`           | `email`, `password`                            | `200 OK` (`user`, `accessToken`, `expiresIn`)      | Authenticates existing user credentials         |
| `googleOAuthLogin`     | `POST` | `/api/v1/auth/google`          | `token` (OAuth ID token)                       | `200 OK` (`user`, `accessToken`, `expiresIn`)      | Authenticates or provisions user via Google SSO |
| `logout`               | `POST` | `/api/v1/auth/logout`          | None (reads refresh cookie)                    | `200 OK` (`success: true`)                         | Clears session cookie and invalidates tokens    |
| `requestPasswordReset` | `POST` | `/api/v1/auth/forgot-password` | `email`                                        | `200 OK` (`success: true`, `message`)              | Sends password recovery instructions to email   |

## 6. Acceptance Criteria

- User can switch between Sign In and Sign Up views via the sliding transition panel.
- User can register with valid name, email, and matching passwords and is redirected to `/dashboard`.
- Attempting registration with an existing email displays a 409 Conflict error message.
- Form displays inline validation errors when required fields are missing or password requirements are unmet.
- User can toggle password obscuration using the eye icon in both password fields.
- User can sign in with valid email/password credentials and receive active session cookies.
- User can initiate Google OAuth flow via the Google SSO button.
- Password reset link opens recovery workflow for registered email addresses.
