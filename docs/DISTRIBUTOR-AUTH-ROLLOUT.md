# Distributor email and Google sign-in rollout

## What changed

- `POST /auth/distributor/login` exchanges a Firebase ID token for a NexaShopping session only when its email is verified and assigned to an active distributor with a profile.
- Admin onboarding now requires a login email. Admins can add an email to existing distributors in Accounts. Changing an email revokes that distributor's existing NexaShopping sessions.
- Creating an active distributor, activating a pending distributor, or assigning a new login email to an active distributor sends a welcome email from `nexashopping9@gmail.com` with a Firebase password link. Admin sees whether delivery succeeded and can resend it from the account detail.
- The distributor panel has email/password sign-in, Google sign-in, and a **Reset password** link. Reset requests use the same NexaShopping Gmail sender. It no longer requests or verifies distributor panel SMS codes.
- Customer OTP and the separate diary portal OTP remain active.

## Before deploying

1. In Firebase project `nexashopping-a3bdd`, enable **Email/Password** and **Google** under Authentication → Sign-in method. Add the distributor panel's production host, `distributor.nexashopping.in`, to Authentication → Settings → Authorized domains. Keep the development host authorized if local sign-in is needed.
2. Confirm the server's existing Firebase service account file is present at `FIREBASE_CREDENTIALS_FILE`. The API needs it to verify tokens, provision distributor identities, and generate password links.
3. In VPS `nexa-server/.env`, set `SMTP_USER=nexashopping9@gmail.com`, `SMTP_APP_PASSWORD` to a Google App Password created for that account, and `DISTRIBUTOR_PANEL_URL=https://distributor.nexashopping.in`. Enable 2-Step Verification on the Gmail account first. Never commit the credential.
4. Schedule a brief login cutover: deploying `nexa-server` retires the old distributor OTP endpoints immediately. Deploy `nexa-server`, then `nexa-admin`.
5. In Admin → Accounts, add a login email to each existing distributor. New distributors must be created with an email. The server creates a Firebase user with an unknown random password if needed, then sends the welcome link when the account is active. Pending accounts receive it on activation. If delivery fails, the account remains created and Admin shows **Resend welcome email**.
6. Deploy `nexa-distributor`. Distributors use the welcome link to set their first password. The panel's **Reset password** action sends another link when needed. They can sign in with email/password or Google using the assigned email.

## Checks and recovery

- A wrong password shows a retry message. An unassigned or inactive email shows a support message. A missing email verification prompts the distributor to verify the email. Password reset responses do not confirm whether an account exists.
- If a distributor does not receive the email, check the assigned email spelling, spam folder, Gmail App Password, Gmail sending limits, server logs, and whether that email exists in Firebase Authentication. Admin can resend the welcome message, or the distributor can request a reset link from the panel.
- If Google sign-in reports that the email uses another credential, sign in with email/password. Firebase account linking can be added separately if that case occurs for a verified non-Gmail Google address.
- Existing customers and diary portal users keep their OTP flow. Retiring the panel OTP routes does not remove the SMS configuration needed by those flows.
