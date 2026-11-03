# Story Decomposition: Sign Up — Set Password

> Source requirement: `requirements/sign-up.md` (steps 4–8 and the related rules). Steps 1–3 are covered by `[User-story-1]sign-up-email-request.md`.

## INVEST Analysis

### Abstract Task: "Sign Up"

**Analysis Dimensions**:
- **Core Responsibility**: Let a new visitor create an account by proving they own an email address, then choosing a password.
- **Primary Operations**:
  - Submit an email address to start sign-up (validate, check uniqueness, check for a pending sign-up, issue a time-limited token, send a verification email)
  - Open the link to show the password form
  - Submit the password together with the token (check the password format and that the token exists, has not expired and has not been invalidated; save the user; invalidate the token)
- **Key Constraints** (stated in the requirement):
  - Users are unique by email address
  - Email format: lowercase ASCII local part of at least 1 character + `@` + lowercase ASCII domain
  - Tokens expire 15 minutes after they are issued
  - Tokens must be secure and can't be guessed or brute-forced
  - Password must be at least 8 characters; it is saved hashed
  - The token is invalidated once the user is saved
- **Derived from the steps** (not stated as rules in the requirement):
  - Only one live sign-up token per email address at a time (step 2 rejects a repeat submission before the existing token expires)
  - Opening the link only shows the password form; the token is checked on submit and invalidated only after a successful submission (steps 4–7)
- **Technical Complexity**: Medium — standard form handling plus outbound email and time-limited tokens
- **Business Complexity**: Medium — several branches (invalid format, already registered, pending sign-up, unknown/expired/invalidated token)

### INVEST Evaluation (whole feature)
- ✅ **Independent**: The feature as a whole does not depend on other features
- ✅ **Negotiable**: Page wording, error messages and email content can be discussed
- ✅ **Valuable**: Without sign-up there are no users
- ✅ **Estimable**: Well-understood flow
- ❌ **Small**: About 6 functional points (email entry, duplicate/pending checks, token and email, password form, submit checks, account creation) and 4–5 days together go beyond the 3-functional-point limit
- ✅ **Testable**: Every step has a clear observable outcome

**Conclusion**: Needs splitting.

### Split Strategy

**Split dimension**: By business step of the journey (each step ends at a page the user can see and a state QA can check).

| Story | Functional points | Business value delivered on its own |
|-------|-------------------|-------------------------------------|
| STORY-001-001 Sign-up email request | 1. Email entry and validation 2. Duplicate/pending checks 3. Token issue and verification email | Collects verified-intent sign-up requests; the verification email reaches the user |
| STORY-001-002 Set password (this file) | 1. Password form opened from the link 2. Submit checks (password format; token not found, expired or invalidated) 3. Account creation, token invalidation and success page | Turns a verification link into a real account |

STORY-001-002 needs the sign-up tokens that STORY-001-001 issues. This is a simple one-way order (001 before 002), not a shared implementation dependency.

---

## [STORY-001-002] Sign-Up Password Setup and Account Creation Development

### Background
After STORY-001-001, a new visitor has a verification email with a link to `/sign-up/set-password?token=<token>`. This story finishes the journey. The link opens a password form. The visitor submits a password together with the token, and if both are valid the system creates their account.

Opening the page only shows the form (step 4). The token is checked when the password is submitted (step 5) and invalidated after the user is saved (steps 6–7). So opening the link, however many times, does not affect sign-up.

Key points:
- This is where a user account is actually created; it completes the Sign Up feature.
- Opening the link without submitting does not affect the token.
- Once an account exists, STORY-001-001 rejects further sign-ups for that address.

### Business Value
- Let **new visitors** turn a verification link into a working account with a password of their choice, in a single page
- Protect **users** by storing only a hashed password, requiring a password of at least 8 characters, and making each link usable only once
- Reach the **system goal** of users who are unique by email address and own that address

### Dependencies and Assumptions
- **Prerequisites**: STORY-001-001 (sign-up email request), which issues the sign-up tokens and verification emails.
- **Data assumptions**:
  - A sign-up token is linked to exactly one email address and expires 15 minutes after it is issued.
  - For testing, a valid token can be obtained by completing STORY-001-001's flow (or seeded directly).
- **Integration points**: None beyond the user store. No email is sent in this story.
- **Business constraints**:
  - On submit, the token must exist, must not have expired and must not have been invalidated (step 5).
  - The token is invalidated once the user is saved (step 7).
  - Tokens must be secure and can't be guessed or brute-forced.
  - Password must be at least 8 characters. No other password rule applies.
  - The password is saved hashed (step 6).
- **Derived from the steps** (not stated as rules in the requirement):
  - Opening `/sign-up/set-password?token=<token>` only shows the password form; it does not check, use up or expire the token (step 4).
  - A failed submission leaves the token valid, since it is invalidated only after the user is saved (steps 5–7).
- **Assumptions made where the source requirement is silent** (to confirm with the PO):
  - If both the password and the token are invalid, the token error is shown, so the visitor doesn't fix their password only to find the link no longer works.
  - The visitor is **not** signed in automatically after sign-up.

### Scope In
- `/sign-up/set-password?token=<token>` page showing a password input and a submit button
- Opening the page any number of times does not change any sign-up state
- On submit, rejecting with an error on the page when the password is shorter than 8 characters
- On submit, rejecting with an error on the page when the token can't be found, has expired or has been invalidated
- Creating a user with the email address linked to the token and a hashed password
- Invalidating the token once the user is created
- Redirecting to the `/sign-up/success` page after the account is created

### Scope Out
- Entering the email address and sending the verification email (STORY-001-001)
- Signing in automatically after sign-up; the log-in feature itself
- A password confirmation ("type it again") field, a strength meter, or any password rule other than minimum length
- A maximum password length rule
- Resending or renewing an expired link from the set-password page
- Limiting the number of failed submissions
- Profile details (name, avatar, etc.) during sign-up
- A welcome email after sign-up

### Acceptance Criteria

#### AC1: Opening the link shows the password form
- **Given** `carol@example.com` requested sign-up at 10:00 and received a link whose token is valid until 10:15
- **When** Carol opens `/sign-up/set-password?token=<token>` at 10:05
- **Then** the page shows a password input and a submit button
  - **And** no account has been created for `carol@example.com` yet

#### AC2: Opening the link without submitting does not affect sign-up
- **Given** Carol's link (valid until 10:15) was already opened at 10:01 and again at 10:02 without submitting (for example, by an email scanner)
- **When** Carol opens the same link herself at 10:05 and submits the password `sunflower42`
- **Then** she is redirected to `/sign-up/success` and a user with `carol@example.com` exists

#### AC3: Submitting a valid password creates the account
- **Given** Carol is on `/sign-up/set-password?token=<token>` at 10:06 with a token for `carol@example.com` that is valid until 10:15
- **When** she submits the password `sunflower42` (11 characters)
- **Then** she is redirected to `/sign-up/success`
  - **And** a user with the email address `carol@example.com` now exists
  - **And** the stored password is hashed; the text `sunflower42` is not stored anywhere in readable form
  - **And** her token is invalidated

#### AC4: Password of exactly 8 characters is accepted
- **Given** Dave has a valid token for `dave@example.com`
- **When** he submits the password `abcd1234` (8 characters) on the set-password page
- **Then** he is redirected to `/sign-up/success` and a user with `dave@example.com` exists

#### AC5: Password shorter than 8 characters is rejected
- **Given** Erin has a valid token for `erin@example.com`
- **When** she submits the password `abc1234` (7 characters), or submits an empty password
- **Then** she stays on the set-password page and sees an error saying the password must be at least 8 characters
  - **And** no user with `erin@example.com` is created
  - **And** her token is still valid, so she can try again with a longer password until it expires

#### AC6: Unknown token is rejected on submit
- **Given** no sign-up token `not-a-real-token` exists
- **When** a visitor opens `/sign-up/set-password?token=not-a-real-token` (or the page with no token at all) and submits the password `abcd1234`
- **Then** the visitor stays on the page and sees an error saying the sign-up link is invalid or has expired
  - **And** no account is created

#### AC7: Expired token is rejected on submit
- **Given** Frank requested sign-up at 10:00, so his token expired at 10:15
- **When** he opens his link at 10:16 and submits the password `riverstone7`
- **Then** he stays on the page and sees the invalid-or-expired link error
  - **And** no user with `frank@example.com` is created
  - **And** he can submit `frank@example.com` again on `/sign-up` to get a new link (STORY-001-001 AC6)

#### AC8: Token expires while the visitor is on the page
- **Given** Grace opened `/sign-up/set-password?token=<token>` at 10:14, and her token expires at 10:15
- **When** she submits the valid password `moonlight99` at 10:17
- **Then** she stays on the page and sees the invalid-or-expired link error
  - **And** no user with `grace@example.com` is created

#### AC9: Invalidated token cannot be reused
- **Given** Carol finished sign-up with her token at 10:06 using the password `sunflower42` (AC3)
- **When** at 10:08 (before the original 10:15 expiry) the same link is used again, from a second browser tab, to submit the password `different99`
- **Then** the page shows the invalid-or-expired link error
  - **And** no new account is created, and Carol's stored password is unchanged

#### AC10: Token error takes priority over password error
- **Given** Frank's token expired at 10:15
- **When** he submits the password `abc` (3 characters) at 10:16
- **Then** he sees the invalid-or-expired link error rather than the password-length error

#### Non-Functional Expectations
- If the set-password form is submitted twice at almost the same moment (for example, a double-click), only one account is created for the email address. The first submission succeeds, and the other is treated as using an invalidated token (AC9).
- Opening the set-password page, any number of times, never changes any sign-up state (step 4).

### Estimated Effort
1.5–2 days
