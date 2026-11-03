# Sign Up

1. The user goes to `/sign-up`, an input for email address displays
2. The user submits the email address
   - Display error if
     - The email address has invalid format
     - A user already has the email address
     - The same email address is submitted again before the existing token expired
3. Else, the user is redirect to `/sign-up/check-email`, while the system sends an email to the address with a link to `/sign-up/set-password?token=<token>`
4. The user goes to the path, an input for password displays
5. The user submits the password and the token are sent together
   - Display error if
     - The password has invalid format
     - The token can't be found
     - The token expired
     - The token is invalidated
6. A user entity with the email address and the hashed password is saved to the system
7. The token is invalidated
8. The user is redirected to `/sign-up/success` 

- Tokens expire in 15 minutes
- Email address format must be: lowercase string in ASCII of at least 1 character + `@` + a lowercase domain in ASCII
- Password must be at least 8 characters
- Token must be secure and can't be guessed or brute-forced 
- Users are unique by email address
