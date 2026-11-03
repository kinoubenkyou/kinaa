# Story Decomposition: Sign Up — Email Request

> Source requirement: `requirements/sign-up.md` (steps 1–3 and the related rules). Steps 4–8 are covered by `[User-story-2]sign-up-set-password.md`.

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
| STORY-001-001 Sign-up email request (this file) | 1. Email entry and validation 2. Duplicate/pending checks 3. Token issue and verification email | Collects verified-intent sign-up requests; the verification email reaches the user |
| STORY-001-002 Set password | 1. Password form opened from the link 2. Submit checks (password format; token not found, expired or invalidated) 3. Account creation, token invalidation and success page | Turns a verification link into a real account |

STORY-001-002 needs the sign-up tokens that STORY-001-001 issues. This is a simple one-way order (001 before 002), not a shared implementation dependency.

---

## [STORY-001-001] Sign-Up Email Request and Verification Email Development

### Background
Kinaa needs a way for new people to create an account. Before an account exists, the system has to confirm that the person controls the email address they gave. This story covers the first half of sign-up: the visitor enters an email address at `/sign-up`, the system checks it, and if everything is in order it sends an email with a personal, time-limited verification link and shows a "check your email" page.

Key points:
- It is the entry point of the whole sign-up journey; no account can be created without it.
- It stops sign-ups for addresses that are already registered, and stops repeated emails while a previous link is still valid.
- The second half (opening the link, setting a password, creating the account) is STORY-001-002.

### Business Value
- Give **new visitors** a simple, email-only first step to start creating an account
- Make sure, in the **account registration** scenario, that the address belongs to the person signing up before an account is created
- Keep **users unique by email address** by refusing addresses that are already registered
- Avoid flooding **an inbox** with verification emails by refusing a new request while an earlier link is still valid

### Dependencies and Assumptions
- **Prerequisites**: None. This is the first story of the Sign Up feature.
- **Data assumptions**:
  - Existing users (if any) are identified by email address.
  - No pending sign-up exists for an address until this story creates one.
- **Integration points**: An outbound email service able to send a message to any valid address.
- **Business constraints**:
  - A sign-up link is valid for 15 minutes from the moment it is issued.
  - Email addresses are accepted only in this format: a lowercase ASCII local part of at least 1 character, then `@`, then a lowercase ASCII domain. Addresses with uppercase letters are rejected, not converted to lowercase (this follows from the lowercase rule).
  - The requirement asks the system to tell the visitor when an address is already registered. This means anyone can find out whether an address has an account. This story follows the requirement as written.
- **Assumptions made where the source requirement is silent** (to confirm with the PO):
  - The visitor is redirected to `/sign-up/check-email` without waiting for the email to be delivered (step 3 says the redirect happens "while" the email is sent).

### Scope In
- `/sign-up` page with an email address input and a submit action
- Email format validation with a user-facing error message
- Refusing an address that already belongs to a user, with an error message
- Refusing an address that already has an unexpired sign-up link, with an error message
- Accepting a new request once the previous link for that address has expired
- Issuing a sign-up token that expires 15 minutes after it is issued
- Sending a verification email containing a link to `/sign-up/set-password?token=<token>`
- Redirecting the visitor to the `/sign-up/check-email` page after a successful request

### Scope Out
- The `/sign-up/set-password` and `/sign-up/success` pages, and account creation (STORY-001-002)
- A "resend email" button on the check-email page
- Accepting addresses with uppercase letters or non-ASCII characters
- Rate limiting or bot protection (CAPTCHA) on the sign-up form
- Sign-up through third-party providers (Google, GitHub, etc.)
- Log in, log out, password reset

### Acceptance Criteria

#### AC1: Sign-up page shows an email input
- **Given** a visitor who is not signed in
- **When** they go to `/sign-up`
- **Then** the page shows an email address input and a submit button

#### AC2: Successful request for a new email address
- **Given** no user has the address `alice@example.com`, and there is no pending sign-up for it
- **When** the visitor submits `alice@example.com` on `/sign-up` at 10:00
- **Then** the visitor is redirected to `/sign-up/check-email`
  - **And** one email is sent to `alice@example.com` with a link to `/sign-up/set-password?token=<token>`
  - **And** the token in that link is valid until 10:15

#### AC3: Each request gets its own token
- **Given** no user or pending sign-up exists for `alice@example.com` or `dave@example.com`
- **When** a visitor submits `alice@example.com` and then `dave@example.com`
- **Then** each address gets its own email, and the two links have different tokens

#### AC4: Email address already registered
- **Given** a user already exists with the address `bob@example.com`
- **When** a visitor submits `bob@example.com` on `/sign-up`
- **Then** the visitor stays on `/sign-up` and sees an error saying the address is already registered
  - **And** no email is sent to `bob@example.com`

#### AC5: Repeat request while the earlier link is still valid
- **Given** `carol@example.com` was submitted at 10:00, so its link is valid until 10:15
- **When** a visitor submits `carol@example.com` again at 10:10
- **Then** the visitor stays on `/sign-up` and sees an error saying a sign-up email was already sent and they should check their inbox
  - **And** no second email is sent
  - **And** the link from 10:00 is still valid

#### AC6: Repeat request after the earlier link has expired
- **Given** `carol@example.com` was submitted at 10:00, so its link expired at 10:15, and no user has that address
- **When** a visitor submits `carol@example.com` again at 10:16
- **Then** the visitor is redirected to `/sign-up/check-email`
  - **And** a new email with a new token is sent to `carol@example.com`, valid until 10:31

#### AC7: Valid email formats are accepted
- **Given** no user or pending sign-up exists for any of these addresses
- **When** a visitor submits each of them: `alice@example.com`, `a@example.com`, `john.doe@mail.example.org`
- **Then** each one is accepted and the visitor is redirected to `/sign-up/check-email`

#### AC8: Invalid email formats are rejected
- **Given** a visitor on `/sign-up`
- **When** they submit any of these values:

  | Value | Reason |
  |-------|--------|
  | *(empty)* | Nothing entered |
  | `Alice@example.com` | Uppercase letter in the local part |
  | `alice@Example.COM` | Uppercase letters in the domain |
  | `@example.com` | Empty local part |
  | `alice@` | Missing domain |
  | `alice.example.com` | Missing `@` |
  | `josé@example.com` | Non-ASCII character |

- **Then** the visitor stays on `/sign-up` and sees an error saying the email address format is invalid
  - **And** no email is sent and no sign-up token is created

#### Non-Functional Expectations
- If the same new address is submitted twice at almost the same moment (for example, a double-click), only one pending sign-up and one email result. The other submission is treated as a repeat request (AC5).
- Sign-up tokens must be secure and impossible to guess or brute-force, because whoever holds a token can finish the sign-up.

### Estimated Effort
2–3 days
