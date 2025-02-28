|           |                      |
| --------- | -------------------- |
| Authors   | Simon Goumaz         |
| Reviewers | Pierre-Mikaël Legris |
| Date      | 2022-06              |
| Version   | 1                    |

# Simple password policy support


## Context

Customers are increasingly constrained by regulations which cover password policies. While it's usually a 'tick-box' concern, and customers can implement whatever checks they like in their own app code, we do need to provide at least basic server-side support for password policies. Currently, Pryv.io enforces no rules at all on passwords.

Euris/Biocorp is one such customer and needs this to be compliant with French HDS rules:

> - SECU_AUTH-3. Complexité des mots de passe applicatifs : Vérifier que les mots de passe utilisés pour se connecter aux serveurs respectent un certain niveau de complexité.
> - SECU_AUTH-4. Changement régulier des mots passe applicatifs : Vérifier qu’une configuration a été mise ne place pour forcer le changement des mots de passe d’accès à l’application à une fréquence régulière définie avec une politique de gestion d’historique.


## Proposal overview

### A. Support password complexity rules

- New passwords must contain characters from either 3/4 or 4/4 of these categories: lowercase, uppercase, number, symbol
  → API server config setting `auth.passwordComplexityMinCharCategories`: [0|3|4], default 0 (no check)
- New passwords must have a minimum length of 8 to 64 characters
  → API server config setting `auth.passwordComplexityMinLength`: number, default 0

Affects:
- Create user
- Change password
- Reset password

### B. Support password renewal rules

#### B1. Password age

- Maximum age: how many days a password _can_ be used before it _must_ be changed
  → API server config setting `auth.passwordAgeMaxDays`: number, default 0 (never expires)
- Minimum age: how many days a password _must_ be used before it _can_ be changed
  → API server config setting `auth.passwordAgeMinDays`: number, default 0 (can be changed right away)

Affects:
- Login:
  - If `auth.passwordAgeMaxDays` is enabled, login returns property `passwordExpires` (timestamp) on success
  - If `auth.passwordAgeMinDays` is enabled, login returns property `passwordCanBeChanged` (timestamp) on success
  - Fail if password expired (user must then reset password)

#### B2. Prevent old passwords reuse

- How many unique passwords must be used before an old password can be reused
  → API server config setting `auth.passwordPreventReuseHistoryLength`: number, default 0 (don't prevent password reuse)

Affects:
- Change password
- Reset password

### Platform settings

- Exposed platform settings: under `ADVANCED_API_SETTINGS`
  - `PASSWORD_COMPLEXITY_MIN_CHAR_CATS` → `auth.passwordComplexityMinCharCategories`
  - `PASSWORD_COMPLEXITY_MIN_LENGTH` → `auth.passwordComplexityMinLength`
  - `PASSWORD_AGE_MAX_DAYS` → `auth.passwordAgeMaxDays`
  - `PASSWORD_AGE_MIN_DAYS` → `auth.passwordAgeMinDays`
  - `PASSWORD_PREVENT_REUSE_HISTORY_LENGTH` → `auth.passwordPreventReuseHistoryLength`


## Implementation notes

### Affected methods

- **Create user**: `POST /users`
  - Params: `password`, …
  - WARNING: confusion around `/system/create-user`
- **Change password**: `{user endpoint}/account/change-password` ([API doc](https://api.pryv.com/reference/#change-password))
  _Requires open personal access session_
  - Params: `oldPassword`, `newPassword`
- **Reset password**: `{user endpoint}/account/reset-password` ([API doc](https://api.pryv.com/reference/#reset-password))
  - Params: `resetToken`, `newPassword`
- **Login**: `{user endpoint}/auth/login` ([API doc](https://api.pryv.com/reference/#login-user))

### Tests

→ `account.test.js`, `login.test.js`, `system.test.js`

**New password complexity rules**: When password complexity rules are enabled in the config, **Create user… | Change password… | Reset password…**
- …must accept the new password if it contains characters from enough categories
- …must return an error if the new password does not contains characters from enough categories
- …must accept the new password if it is long enough
- …must return an error if the new password is too short

**Password reuse rules**: When password reuse rules are enabled in the config, **Change password… | Reset password…**
- …must accept the new password if different from the N last passwords used
- …must return an error if the new password is found in the N last passwords used

**Password age rules**: When password age rules are enabled, **Change password… | Reset password…**
- …must accept the new password if the current one's age is greater than the set minimum
- …must return an error if the current password's age is below the set minimum

**Password age rules**: When password age rules are enabled, **Login…**
- …must succeed if the password is not yet expired, returning the planned expiration (max age) timestamp (`passwordExpires`) and possible change (min age) timestamp (`passwordCanBeChanged`)
- …must return an error if the password has expired, indicating the date it did so and the max age setting value
