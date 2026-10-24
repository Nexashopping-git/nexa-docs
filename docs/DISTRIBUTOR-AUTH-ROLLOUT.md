# Distributor email and Google sign-in rollout

## What changed

- `POST /auth/distributor/login` exchanges a Firebase ID token for a NexaShopping session only when its email is verified and assigned to an active distributor with a profile.
- Admin onboarding now requires a login email. Admins can add an email to existing distributors in Accounts. Changing an email revokes that distributor's existing NexaShopping sessions.
- The distributor panel has email/password sign-in, Google sign-in, and a **Set or reset password** link. It no longer requests or verifies distributor panel SMS codes.
- Customer OTP and the separate diary portal OTP remain active.

## Before deploying

1. In Firebase project `nexashopping-a3bdd`, enable **Email/Password** and **Google** under Authentication → Sign-in method. Add the distributor panel's production host, `distributor.nexashopping.in`, to Authentication → Settings → Authorized domains. Keep the development host authorized if local sign-in is needed.
2. Confirm the server's existing Firebase service account file is present at `FIREBASE_CREDENTIALS_FILE`. The API now needs it both to verify distributor tokens and to provision login users during admin onboarding.
3. Schedule a brief login cutover: deploying `nexa-server` retires the old distributor OTP endpoints immediately. Deploy `nexa-server`, then `nexa-admin`. The updated admin panel sends distributor emails to the updated account API.
4. In Admin → Accounts, add a valid login email to each existing distributor. New distributors must be created with an email. The server creates a Firebase user with an unknown random password if needed; it never sends a password to the admin.
5. Deploy `nexa-distributor` after the existing emails are assigned. Ask each distributor to select **Set or reset password** and enter the assigned email. Firebase emails a password reset link. After setting the password, they sign in with email/password or Google using the assigned email.

## Checks and recovery

- A wrong password shows a retry message. An unassigned or inactive email shows a support message. A missing email verification prompts the distributor to verify the email. Password reset responses do not confirm whether an account exists.
- If a distributor does not receive the reset email, check the assigned email spelling, spam folder, Firebase email templates and sending limits, and whether that email exists in Firebase Authentication. They can request another link from the panel.
- If Google sign-in reports that the email uses another credential, sign in with email/password. Firebase account linking can be added separately if that case occurs for a verified non-Gmail Google address.
- Existing customers and diary portal users keep their OTP flow. Retiring the panel OTP routes does not remove the SMS configuration needed by those flows.
