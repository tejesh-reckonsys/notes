---
date: "2025-05-25T18:19:05+05:30"
draft: "false"
title: "OTP Authentication"
layout: single
tags: ["auth", "signup", "login"]
---

# User Authentication System with OTP Verification

This document provides UML diagrams for the signup and login processes of our user authentication system, which includes OTP verification via email.

## Complete Authentication Flow State Diagram


```mermaid
stateDiagram-v2
    [*] --> LandingPage

    LandingPage --> SignupProcess: Choose Signup
    LandingPage --> LoginProcess: Choose Login
    LandingPage --> PasswordResetProcess: Forgot Password

    %% Main Authentication Flows
    SignupProcess --> OTPVerificationFlow: After registration
    LoginProcess --> AuthenticatedState: Successful login (verified user)
    LoginProcess --> OTPVerificationFlow: Successful login (unverified user)
    PasswordResetProcess --> LoginProcess: After password reset

    %% OTP Verification Flow
    OTPVerificationFlow --> LoginProcess: Successful verification
    OTPVerificationFlow --> OTPVerificationFlow: Request new OTP

    %% Final states
    AuthenticatedState --> [*]: Logout
```


This state diagram represents the complete authentication flow, including signup, OTP verification, login, resend OTP, and password reset processes. It shows how these processes are interconnected and the various paths a user might take depending on different conditions and actions.

## User Entity

```mermaid
classDiagram
    class User {
        +int id
        +string email
        +string full_name
        +string avatar
        +string password_hash
        +boolean is_verified
        +datetime created_at
        +datetime updated_at
    }

    class OTP {
        +int id
        +int user_id
        +string code
        +datetime expires_at
        +boolean is_used
        +datetime created_at
    }

    User "1" -- "many" OTP : generates
```

The User entity represents registered users in our system. Each user can have multiple OTP records associated with their account. The User entity stores essential information like email, name, and verification status, while the OTP entity tracks verification codes sent to users.

## Signup Process

```mermaid
sequenceDiagram
    actor User
    participant Frontend
    participant Backend
    participant Database
    participant EmailService

    User->>Frontend: Enters email, full_name, password, avatar
    Frontend->>Backend: POST /api/signup
    Backend->>Database: Check if email exists
    alt Email already exists
        Database->>Backend: Email exists
        Backend->>Frontend: 400 - Email already registered
        Frontend->>User: Show error message
    else Email is new
        Backend->>Database: Create user with is_verified=false
        Database->>Backend: Return user data
        Backend->>EmailService: Generate OTP and send to user's email
        EmailService->>User: Send OTP verification email
        Backend->>Frontend: 201 - User created, verification required (includes temporary verification_token)
        Frontend->>User: Show OTP verification screen
    end
```

The signup process creates a new unverified user account. Upon successful user creation, the backend generates a temporary verification token (JWT with short expiry) that is returned to the frontend along with the 201 response. This verification token contains the user's ID and is required for the subsequent OTP verification step. The token is separate from the authentication JWT and is only used for the verification process.

## Signup Activity Diagram

```mermaid
stateDiagram-v2
    [*] --> CollectUserData
    CollectUserData --> ValidateInput
    ValidateInput --> CheckEmailExists

    CheckEmailExists --> EmailExists: Email already registered
    EmailExists --> [*]: Display error

    CheckEmailExists --> CreateUser: Email is new
    CreateUser --> GenerateOTP
    GenerateOTP --> SendEmail
    SendEmail --> GenerateVerificationToken
    GenerateVerificationToken --> DisplayOTPScreen
    DisplayOTPScreen --> [*]
```

## OTP Verification Process

```mermaid
sequenceDiagram
    actor User
    participant Frontend
    participant Backend
    participant Database

    User->>Frontend: Enters OTP received in email
    Frontend->>Backend: POST /api/verify-otp (with verification_token and OTP)
    Backend->>Database: Validate OTP and token
    alt OTP is valid and not expired, token is valid
        Database->>Backend: OTP valid
        Backend->>Database: Update user is_verified=true
        Backend->>Database: Mark OTP as used
        Backend->>Frontend: 200 - Verification successful (includes JWT access_token)
        Frontend->>User: Show success message & redirect to login
    else OTP is invalid or expired or token invalid
        Database->>Backend: OTP invalid or expired or token invalid
        Backend->>Frontend: 400 - Invalid or expired OTP/token
        Frontend->>User: Show error message
    end
```

The OTP verification process validates the user's email address. The frontend must send both the OTP code and the verification token received during signup. The backend validates both the OTP and the verification token to ensure the request is legitimate. Once verified, the system marks the user as verified, marks the OTP as used, and returns a full access JWT token that can be used for subsequent authenticated requests. The verification token is valid for 5 minutes only.

## OTP Verification Activity Diagram

```mermaid
stateDiagram-v2
    [*] --> EnterOTP
    EnterOTP --> ValidateTokenAndOTP

    ValidateTokenAndOTP --> TokenInvalid: Invalid token
    TokenInvalid --> [*]: Show error

    ValidateTokenAndOTP --> OTPInvalid: Invalid OTP
    OTPInvalid --> [*]: Show error

    ValidateTokenAndOTP --> OTPExpired: OTP expired
    OTPExpired --> [*]: Show error

    ValidateTokenAndOTP --> MarkUserVerified: Valid OTP & token
    MarkUserVerified --> MarkOTPUsed
    MarkOTPUsed --> GenerateAccessToken
    GenerateAccessToken --> ShowSuccessMessage
    ShowSuccessMessage --> RedirectToLogin
    RedirectToLogin --> [*]
```

## Login Process

```mermaid
sequenceDiagram
    actor User
    participant Frontend
    participant Backend
    participant Database
    participant EmailService

    User->>Frontend: Enters email and password
    Frontend->>Backend: POST /api/login
    Backend->>Database: Check credentials
    alt Credentials valid
        Database->>Backend: User authenticated
        alt User is verified
            Backend->>Frontend: 200 - Return JWT access_token
            Frontend->>User: Redirect to dashboard
        else User is not verified
            Backend->>EmailService: Generate new OTP and send to user's email
            EmailService->>User: Send new OTP verification email
            Backend->>Frontend: 403 - Email not verified (includes new verification_token)
            Frontend->>User: Show verification prompt
        end
    else Credentials invalid
        Database->>Backend: Authentication failed
        Backend->>Frontend: 401 - Invalid credentials
        Frontend->>User: Show error message
    end
```

The login process authenticates users and provides access to the system. When credentials are valid and the user is verified, the backend returns a JWT access token that should be included in all subsequent authenticated requests. This token contains the user's ID and role information. If the user is not verified, the system automatically generates and sends a new OTP to the user's email, and the response includes a new verification token that can be used with the OTP verification endpoint to complete verification.

## Login State Diagram

```mermaid
stateDiagram-v2
    [*] --> EnterCredentials
    EnterCredentials --> ValidateCredentials

    ValidateCredentials --> InvalidCredentials: Authentication failed
    InvalidCredentials --> [*]: Show error

    ValidateCredentials --> CheckVerification: Authentication successful

    CheckVerification --> AccountVerified: is_verified = true
    AccountVerified --> GenerateJWT
    GenerateJWT --> RedirectToDashboard
    RedirectToDashboard --> [*]

    CheckVerification --> AccountUnverified: is_verified = false
    AccountUnverified --> GenerateNewOTP
    GenerateNewOTP --> SendVerificationEmail
    SendVerificationEmail --> GenerateVerificationToken
    GenerateVerificationToken --> ShowVerificationPrompt
    ShowVerificationPrompt --> [*]
```

## Resend OTP API

```mermaid
sequenceDiagram
    actor User
    participant Frontend
    participant Backend
    participant Database
    participant EmailService

    User->>Frontend: Requests new OTP
    Frontend->>Backend: POST /api/resend-otp (with verification_token)
    Backend->>Database: Validate verification_token
    alt Token is valid
        Backend->>EmailService: Generate new OTP and send to user's email
        EmailService->>User: Send new OTP verification email
        Backend->>Frontend: 200 - New OTP sent (includes new verification_token)
        Frontend->>User: Show notification that new OTP has been sent
    else Token is invalid or expired
        Backend->>Frontend: 400 - Invalid or expired token
        Frontend->>User: Show error message, prompt to login again
    end
```

The Resend OTP API allows users to request a new OTP if the previous one expires or gets lost. The frontend must provide the current verification token, which is validated by the backend. If valid, a new OTP is generated and sent, and a new verification token is issued. This provides users with a way to recover from lost or expired OTPs without starting the entire signup or login process again.

## Resend OTP Activity Diagram

```mermaid
stateDiagram-v2
    [*] --> RequestNewOTP
    RequestNewOTP --> ValidateToken

    ValidateToken --> TokenInvalid: Invalid or expired
    TokenInvalid --> ShowError
    ShowError --> PromptLogin
    PromptLogin --> [*]

    ValidateToken --> GenerateNewOTP: Token valid
    GenerateNewOTP --> SendEmail
    SendEmail --> GenerateNewToken
    GenerateNewToken --> ShowNotification
    ShowNotification --> [*]
```

## Password Reset with OTP

```mermaid
sequenceDiagram
    actor User
    participant Frontend
    participant Backend
    participant Database
    participant EmailService

    User->>Frontend: Requests password reset
    Frontend->>Backend: POST /api/request-password-reset
    Backend->>Database: Find user by email
    alt User found
        Backend->>EmailService: Generate OTP and send to user's email
        EmailService->>User: Send OTP for password reset
        Backend->>Frontend: 200 - OTP sent (includes reset_token)
        Frontend->>User: Show OTP entry screen
        User->>Frontend: Enters OTP and new password
        Frontend->>Backend: POST /api/reset-password (with reset_token, OTP, and new password)
        Backend->>Database: Validate OTP and token
        alt OTP valid and token valid
            Backend->>Database: Update password and mark OTP as used
            Backend->>Frontend: 200 - Password updated
            Frontend->>User: Show success message & redirect to login
        else OTP invalid or token invalid
            Backend->>Frontend: 400 - Invalid OTP or token
            Frontend->>User: Show error message
        end
    else User not found
        Backend->>Frontend: 200 - OTP sent (security through obscurity)
        Frontend->>User: Show OTP entry screen
    end
```

The password reset process provides a secure way for users to regain access to their accounts. When a reset is requested, the backend generates a special reset token (JWT with limited scope and short expiry) that is returned to the frontend. This token, along with the OTP and new password, must be included in the password reset request. The backend validates both the OTP and reset token before allowing the password change. This ensures that only users with access to the email account can reset the password.

## Password Reset Activity Diagram

```mermaid
stateDiagram-v2
    [*] --> RequestPasswordReset
    RequestPasswordReset --> CheckUserExists

    CheckUserExists --> UserNotFound: No account with email
    UserNotFound --> SendGenericResponse: Security by obscurity
    SendGenericResponse --> ShowOTPScreen

    CheckUserExists --> UserFound: Account exists
    UserFound --> GenerateOTP
    GenerateOTP --> SendOTPEmail
    SendOTPEmail --> GenerateResetToken
    GenerateResetToken --> ShowOTPScreen

    ShowOTPScreen --> EnterOTPAndNewPassword
    EnterOTPAndNewPassword --> ValidateOTPAndToken

    ValidateOTPAndToken --> InvalidOTPOrToken: Invalid
    InvalidOTPOrToken --> ShowError
    ShowError --> [*]

    ValidateOTPAndToken --> OTPAndTokenValid: Valid
    OTPAndTokenValid --> UpdatePassword
    UpdatePassword --> MarkOTPUsed
    MarkOTPUsed --> ShowSuccess
    ShowSuccess --> RedirectToLogin
    RedirectToLogin --> [*]
```
## Example User Data

| User ID | Email | Registration Status | Last Login | OTP Attempts |
|---------|-------|---------------------|------------|--------------|
| 1001 | alice@example.com | Verified | 2025-05-20 | 1 |
| 1002 | bob@example.com | Unverified | 2025-05-22 | 3 |
| 1003 | charlie@example.com | Verified | 2025-05-18 | 1 |
| 1004 | diana@example.com | Pending | 2025-05-24 | 2 |
| 1005 | evan@example.com | Verified | 2025-05-23 | 1 |
