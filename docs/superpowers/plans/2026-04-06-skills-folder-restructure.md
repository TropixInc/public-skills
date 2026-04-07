# Skills Folder Restructure Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Reorganize `skills/` from a flat/offpix-nested structure into `user/`, `admin/`, and `references/` based on audience, eliminating the "offpix" wrapper.

**Architecture:** Pure file-move operation. Create target directories, move files using `git mv` to preserve history, remove empty old directories, verify no broken internal links.

**Tech Stack:** git, bash

---

## File Move Map

### Legend
- `S` = source path (relative to `skills/`)
- `D` = destination path (relative to `skills/`)

---

### Task 1: Create target directory structure

- [ ] **Step 1: Create all target directories**

```bash
cd /c/Users/ferna/W3block/public-skills
mkdir -p skills/_setup
mkdir -p skills/_meta
mkdir -p skills/references/{auth,checkout,commerce,configurations,contacts,contracts,kyc,loyalty,pass,pdf,settings,tokenization,tokens,user-profile,whitelist}
mkdir -p skills/user/{auth,checkout,tokens,pass,kyc}
mkdir -p skills/admin/{auth,commerce,configurations,contacts,contracts,kyc,loyalty,pass,pdf,settings,tokenization,user-profile,whitelist}
```

- [ ] **Step 2: Verify directories were created**

```bash
find skills/_setup skills/_meta skills/references skills/user skills/admin -type d | sort
```

---

### Task 2: Move `_setup/` files (from `skills/setup/`)

| # | Source | Destination |
|---|--------|-------------|
| 1 | `setup/SETUP_SKILL_INDEX.md` | `_setup/SETUP_SKILL_INDEX.md` |
| 2 | `setup/SETUP_PROJECT_BOOTSTRAP.md` | `_setup/SETUP_PROJECT_BOOTSTRAP.md` |
| 3 | `setup/SETUP_API_PATTERNS.md` | `_setup/SETUP_API_PATTERNS.md` |
| 4 | `offpix/OFFPIX_DOCUMENTATION_SPEC.md` | `_setup/DOCUMENTATION_SPEC.md` |

- [ ] **Step 1: Move files**

```bash
cd /c/Users/ferna/W3block/public-skills
git mv skills/setup/SETUP_SKILL_INDEX.md skills/_setup/
git mv skills/setup/SETUP_PROJECT_BOOTSTRAP.md skills/_setup/
git mv skills/setup/SETUP_API_PATTERNS.md skills/_setup/
git mv skills/offpix/OFFPIX_DOCUMENTATION_SPEC.md skills/_setup/DOCUMENTATION_SPEC.md
```

- [ ] **Step 2: Commit**

```bash
git add -A skills/_setup skills/setup skills/offpix/OFFPIX_DOCUMENTATION_SPEC.md
git commit -m "refactor: move setup and doc spec to skills/_setup"
```

---

### Task 3: Move `_meta/` files

| # | Source | Destination |
|---|--------|-------------|
| 1 | `offpix/OFFPIX_API_GAP_ANALYSIS.md` | `_meta/API_GAP_ANALYSIS.md` |
| 2 | `offpix/OFFPIX_SKILL_INDEX.md` | `_meta/OFFPIX_SKILL_INDEX.md` (archive) |

- [ ] **Step 1: Move files**

```bash
cd /c/Users/ferna/W3block/public-skills
git mv skills/offpix/OFFPIX_API_GAP_ANALYSIS.md skills/_meta/API_GAP_ANALYSIS.md
git mv skills/offpix/OFFPIX_SKILL_INDEX.md skills/_meta/OFFPIX_SKILL_INDEX.md
```

- [ ] **Step 2: Commit**

```bash
git add -A skills/_meta skills/offpix
git commit -m "refactor: move gap analysis and offpix index to skills/_meta"
```

---

### Task 4: Move `references/` files — API references + BOTH docs

| # | Source | Destination |
|---|--------|-------------|
| 1 | `offpix/auth/AUTH_API_REFERENCE.md` | `references/auth/AUTH_API_REFERENCE.md` |
| 2 | `checkout/CHECKOUT_API_REFERENCE.md` | `references/checkout/CHECKOUT_API_REFERENCE.md` |
| 3 | `checkout/FLOW_CHECKOUT_OVERVIEW.md` | `references/checkout/FLOW_CHECKOUT_OVERVIEW.md` |
| 4 | `offpix/commerce/COMMERCE_API_REFERENCE.md` | `references/commerce/COMMERCE_API_REFERENCE.md` |
| 5 | `offpix/commerce/FLOW_COMMERCE_ORDER_PURCHASE.md` | `references/commerce/FLOW_COMMERCE_ORDER_PURCHASE.md` |
| 6 | `offpix/configurations/CONFIGURATIONS_API_REFERENCE.md` | `references/configurations/CONFIGURATIONS_API_REFERENCE.md` |
| 7 | `offpix/contacts/CONTACTS_API_REFERENCE.md` | `references/contacts/CONTACTS_API_REFERENCE.md` |
| 8 | `offpix/contracts/CONTRACTS_API_REFERENCE.md` | `references/contracts/CONTRACTS_API_REFERENCE.md` |
| 9 | `offpix/kyc/KYC_API_REFERENCE.md` | `references/kyc/KYC_API_REFERENCE.md` |
| 10 | `offpix/kyc/KYC_SKILL_INDEX.md` | `references/kyc/KYC_SKILL_INDEX.md` |
| 11 | `offpix/loyalty/LOYALTY_API_REFERENCE.md` | `references/loyalty/LOYALTY_API_REFERENCE.md` |
| 12 | `offpix/pass/PASS_API_REFERENCE.md` | `references/pass/PASS_API_REFERENCE.md` |
| 13 | `pass/FLOW_PASS_OVERVIEW.md` | `references/pass/FLOW_PASS_OVERVIEW.md` |
| 14 | `pass/FLOW_PASS_BENEFITS_VIEW.md` | `references/pass/FLOW_PASS_BENEFITS_VIEW.md` |
| 15 | `pass/FLOW_PASS_SHARE.md` | `references/pass/FLOW_PASS_SHARE.md` |
| 16 | `pass/PASS_API_REFERENCE.md` | `references/pass/PASS_FRONTEND_API_REFERENCE.md` |
| 17 | `offpix/pdf/PDF_API_REFERENCE.md` | `references/pdf/PDF_API_REFERENCE.md` |
| 18 | `offpix/settings/SETTINGS_API_REFERENCE.md` | `references/settings/SETTINGS_API_REFERENCE.md` |
| 19 | `offpix/tokenization/TOKENIZATION_API_REFERENCE.md` | `references/tokenization/TOKENIZATION_API_REFERENCE.md` |
| 20 | `tokens/TOKENS_API_REFERENCE.md` | `references/tokens/TOKENS_API_REFERENCE.md` |
| 21 | `user-profile/USER_PROFILE_API_REFERENCE.md` | `references/user-profile/USER_PROFILE_API_REFERENCE.md` |
| 22 | `offpix/whitelist/WHITELIST_API_REFERENCE.md` | `references/whitelist/WHITELIST_API_REFERENCE.md` |

- [ ] **Step 1: Move offpix API references**

```bash
cd /c/Users/ferna/W3block/public-skills
git mv skills/offpix/auth/AUTH_API_REFERENCE.md skills/references/auth/
git mv skills/offpix/commerce/COMMERCE_API_REFERENCE.md skills/references/commerce/
git mv skills/offpix/commerce/FLOW_COMMERCE_ORDER_PURCHASE.md skills/references/commerce/
git mv skills/offpix/configurations/CONFIGURATIONS_API_REFERENCE.md skills/references/configurations/
git mv skills/offpix/contacts/CONTACTS_API_REFERENCE.md skills/references/contacts/
git mv skills/offpix/contracts/CONTRACTS_API_REFERENCE.md skills/references/contracts/
git mv skills/offpix/kyc/KYC_API_REFERENCE.md skills/references/kyc/
git mv skills/offpix/kyc/KYC_SKILL_INDEX.md skills/references/kyc/
git mv skills/offpix/loyalty/LOYALTY_API_REFERENCE.md skills/references/loyalty/
git mv skills/offpix/pass/PASS_API_REFERENCE.md skills/references/pass/
git mv skills/offpix/pdf/PDF_API_REFERENCE.md skills/references/pdf/
git mv skills/offpix/settings/SETTINGS_API_REFERENCE.md skills/references/settings/
git mv skills/offpix/tokenization/TOKENIZATION_API_REFERENCE.md skills/references/tokenization/
git mv skills/offpix/whitelist/WHITELIST_API_REFERENCE.md skills/references/whitelist/
```

- [ ] **Step 2: Move root-level API references and BOTH docs**

```bash
cd /c/Users/ferna/W3block/public-skills
git mv skills/checkout/CHECKOUT_API_REFERENCE.md skills/references/checkout/
git mv skills/checkout/FLOW_CHECKOUT_OVERVIEW.md skills/references/checkout/
git mv skills/tokens/TOKENS_API_REFERENCE.md skills/references/tokens/
git mv skills/user-profile/USER_PROFILE_API_REFERENCE.md skills/references/user-profile/
git mv skills/pass/FLOW_PASS_OVERVIEW.md skills/references/pass/
git mv skills/pass/FLOW_PASS_BENEFITS_VIEW.md skills/references/pass/
git mv skills/pass/FLOW_PASS_SHARE.md skills/references/pass/
git mv skills/pass/PASS_API_REFERENCE.md skills/references/pass/PASS_FRONTEND_API_REFERENCE.md
```

- [ ] **Step 3: Commit**

```bash
git add -A skills/references skills/offpix skills/checkout skills/tokens skills/user-profile skills/pass
git commit -m "refactor: move API references and shared docs to skills/references"
```

---

### Task 5: Move `user/` files

| # | Source | Destination |
|---|--------|-------------|
| 1 | `auth/FLOW_SIGNUP.md` | `user/auth/FLOW_SIGNUP.md` |
| 2 | `checkout/CHECKOUT_SKILL_INDEX.md` | `user/checkout/CHECKOUT_SKILL_INDEX.md` |
| 3 | `checkout/FLOW_CHECKOUT_CART_CONFIRMATION.md` | `user/checkout/FLOW_CHECKOUT_CART_CONFIRMATION.md` |
| 4 | `checkout/FLOW_CHECKOUT_PAYMENT_CREDIT_CARD.md` | `user/checkout/FLOW_CHECKOUT_PAYMENT_CREDIT_CARD.md` |
| 5 | `checkout/FLOW_CHECKOUT_PAYMENT_CRYPTO.md` | `user/checkout/FLOW_CHECKOUT_PAYMENT_CRYPTO.md` |
| 6 | `checkout/FLOW_CHECKOUT_PAYMENT_PIX.md` | `user/checkout/FLOW_CHECKOUT_PAYMENT_PIX.md` |
| 7 | `checkout/FLOW_CHECKOUT_PAYMENT_STRIPE.md` | `user/checkout/FLOW_CHECKOUT_PAYMENT_STRIPE.md` |
| 8 | `checkout/FLOW_CHECKOUT_PAYMENT_TRANSFER.md` | `user/checkout/FLOW_CHECKOUT_PAYMENT_TRANSFER.md` |
| 9 | `checkout/FLOW_CHECKOUT_COMPLETION.md` | `user/checkout/FLOW_CHECKOUT_COMPLETION.md` |
| 10 | `checkout/FLOW_MY_ORDERS.md` | `user/checkout/FLOW_MY_ORDERS.md` |
| 11 | `tokens/TOKENS_SKILL_INDEX.md` | `user/tokens/TOKENS_SKILL_INDEX.md` |
| 12 | `tokens/FLOW_TOKENS_NFT_WALLET.md` | `user/tokens/FLOW_TOKENS_NFT_WALLET.md` |
| 13 | `tokens/FLOW_TOKENS_METADATA.md` | `user/tokens/FLOW_TOKENS_METADATA.md` |
| 14 | `tokens/FLOW_TOKENS_TRANSFER.md` | `user/tokens/FLOW_TOKENS_TRANSFER.md` |
| 15 | `tokens/FLOW_TOKENS_BALANCE.md` | `user/tokens/FLOW_TOKENS_BALANCE.md` |
| 16 | `tokens/FLOW_TOKENS_WALLET_CONNECT.md` | `user/tokens/FLOW_TOKENS_WALLET_CONNECT.md` |
| 17 | `pass/FLOW_PASS_SELF_USE.md` | `user/pass/FLOW_PASS_SELF_USE.md` |
| 18 | `offpix/pass/FLOW_PASS_BENEFIT_REDEMPTION.md` | `user/pass/FLOW_PASS_BENEFIT_REDEMPTION.md` |
| 19 | `offpix/kyc/FLOW_KYC_SUBMISSION.md` | `user/kyc/FLOW_KYC_SUBMISSION.md` |
| 20 | `offpix/contacts/FLOW_CONTACTS_KYC_SUBMISSION.md` | `user/kyc/FLOW_CONTACTS_KYC_SUBMISSION.md` |

- [ ] **Step 1: Move user auth & checkout**

```bash
cd /c/Users/ferna/W3block/public-skills
git mv skills/auth/FLOW_SIGNUP.md skills/user/auth/
git mv skills/checkout/CHECKOUT_SKILL_INDEX.md skills/user/checkout/
git mv skills/checkout/FLOW_CHECKOUT_CART_CONFIRMATION.md skills/user/checkout/
git mv skills/checkout/FLOW_CHECKOUT_PAYMENT_CREDIT_CARD.md skills/user/checkout/
git mv skills/checkout/FLOW_CHECKOUT_PAYMENT_CRYPTO.md skills/user/checkout/
git mv skills/checkout/FLOW_CHECKOUT_PAYMENT_PIX.md skills/user/checkout/
git mv skills/checkout/FLOW_CHECKOUT_PAYMENT_STRIPE.md skills/user/checkout/
git mv skills/checkout/FLOW_CHECKOUT_PAYMENT_TRANSFER.md skills/user/checkout/
git mv skills/checkout/FLOW_CHECKOUT_COMPLETION.md skills/user/checkout/
git mv skills/checkout/FLOW_MY_ORDERS.md skills/user/checkout/
```

- [ ] **Step 2: Move user tokens**

```bash
cd /c/Users/ferna/W3block/public-skills
git mv skills/tokens/TOKENS_SKILL_INDEX.md skills/user/tokens/
git mv skills/tokens/FLOW_TOKENS_NFT_WALLET.md skills/user/tokens/
git mv skills/tokens/FLOW_TOKENS_METADATA.md skills/user/tokens/
git mv skills/tokens/FLOW_TOKENS_TRANSFER.md skills/user/tokens/
git mv skills/tokens/FLOW_TOKENS_BALANCE.md skills/user/tokens/
git mv skills/tokens/FLOW_TOKENS_WALLET_CONNECT.md skills/user/tokens/
```

- [ ] **Step 3: Move user pass & kyc**

```bash
cd /c/Users/ferna/W3block/public-skills
git mv skills/pass/FLOW_PASS_SELF_USE.md skills/user/pass/
git mv skills/offpix/pass/FLOW_PASS_BENEFIT_REDEMPTION.md skills/user/pass/
git mv skills/offpix/kyc/FLOW_KYC_SUBMISSION.md skills/user/kyc/
git mv skills/offpix/contacts/FLOW_CONTACTS_KYC_SUBMISSION.md skills/user/kyc/
```

- [ ] **Step 4: Commit**

```bash
git add -A skills/user skills/auth skills/checkout skills/tokens skills/pass skills/offpix
git commit -m "refactor: move end-user facing docs to skills/user"
```

---

### Task 6: Move `admin/` files — auth, commerce, configurations, contacts

- [ ] **Step 1: Move admin auth**

```bash
cd /c/Users/ferna/W3block/public-skills
git mv skills/offpix/auth/AUTH_SKILL_INDEX.md skills/admin/auth/
git mv skills/offpix/auth/FLOW_AUTH_SIGNIN.md skills/admin/auth/
git mv skills/offpix/auth/FLOW_AUTH_SIGNUP.md skills/admin/auth/
git mv skills/offpix/auth/FLOW_AUTH_PASSWORD_RESET.md skills/admin/auth/
git mv skills/offpix/auth/FLOW_AUTH_OAUTH.md skills/admin/auth/
```

- [ ] **Step 2: Move admin commerce**

```bash
cd /c/Users/ferna/W3block/public-skills
git mv skills/offpix/commerce/COMMERCE_SKILL_INDEX.md skills/admin/commerce/
git mv skills/offpix/commerce/FLOW_COMMERCE_PRODUCT_LIFECYCLE.md skills/admin/commerce/
git mv skills/offpix/commerce/FLOW_COMMERCE_PROMOTIONS.md skills/admin/commerce/
git mv skills/offpix/commerce/FLOW_COMMERCE_SPLITS_GATEWAYS.md skills/admin/commerce/
```

- [ ] **Step 3: Move admin configurations**

```bash
cd /c/Users/ferna/W3block/public-skills
git mv skills/offpix/configurations/CONFIGURATIONS_SKILL_INDEX.md skills/admin/configurations/
git mv skills/offpix/configurations/FLOW_CONFIGURATIONS_CONTEXT_LIFECYCLE.md skills/admin/configurations/
git mv skills/offpix/configurations/FLOW_CONFIGURATIONS_INPUT_MANAGEMENT.md skills/admin/configurations/
git mv skills/offpix/configurations/FLOW_CONFIGURATIONS_SIGNUP_ACTIVATION.md skills/admin/configurations/
```

- [ ] **Step 4: Move admin contacts**

```bash
cd /c/Users/ferna/W3block/public-skills
git mv skills/offpix/contacts/CONTACTS_SKILL_INDEX.md skills/admin/contacts/
git mv skills/offpix/contacts/FLOW_CONTACTS_KYC_APPROVAL.md skills/admin/contacts/
git mv skills/offpix/contacts/FLOW_CONTACTS_USER_MANAGEMENT.md skills/admin/contacts/
```

- [ ] **Step 5: Commit**

```bash
git add -A skills/admin skills/offpix
git commit -m "refactor: move admin auth, commerce, configurations, contacts docs"
```

---

### Task 7: Move `admin/` files — contracts, kyc, loyalty, pass

- [ ] **Step 1: Move admin contracts**

```bash
cd /c/Users/ferna/W3block/public-skills
git mv skills/offpix/contracts/CONTRACTS_SKILL_INDEX.md skills/admin/contracts/
git mv skills/offpix/contracts/FLOW_CONTRACTS_COLLECTION_MANAGEMENT.md skills/admin/contracts/
git mv skills/offpix/contracts/FLOW_CONTRACTS_ERC20_LIFECYCLE.md skills/admin/contracts/
git mv skills/offpix/contracts/FLOW_CONTRACTS_NFT_LIFECYCLE.md skills/admin/contracts/
git mv skills/offpix/contracts/FLOW_CONTRACTS_TOKEN_OPERATIONS.md skills/admin/contracts/
```

- [ ] **Step 2: Move admin kyc**

```bash
cd /c/Users/ferna/W3block/public-skills
git mv skills/offpix/kyc/FLOW_KYC_CONFIGURATION.md skills/admin/kyc/
git mv skills/offpix/kyc/FLOW_KYC_APPROVAL.md skills/admin/kyc/
```

- [ ] **Step 3: Move admin loyalty**

```bash
cd /c/Users/ferna/W3block/public-skills
git mv skills/offpix/loyalty/LOYALTY_SKILL_INDEX.md skills/admin/loyalty/
git mv skills/offpix/loyalty/FLOW_LOYALTY_BALANCE_OPERATIONS.md skills/admin/loyalty/
git mv skills/offpix/loyalty/FLOW_LOYALTY_CONTRACT_SETUP.md skills/admin/loyalty/
git mv skills/offpix/loyalty/FLOW_LOYALTY_REWARDS_PAYMENTS.md skills/admin/loyalty/
```

- [ ] **Step 4: Move admin pass**

```bash
cd /c/Users/ferna/W3block/public-skills
git mv skills/offpix/pass/PASS_SKILL_INDEX.md skills/admin/pass/
git mv skills/offpix/pass/FLOW_PASS_LIFECYCLE.md skills/admin/pass/
git mv skills/pass/PASS_SKILL_INDEX.md skills/admin/pass/PASS_FRONTEND_SKILL_INDEX.md
git mv skills/pass/FLOW_PASS_MANAGEMENT.md skills/admin/pass/
git mv skills/pass/FLOW_PASS_BENEFIT_VERIFICATION.md skills/admin/pass/
git mv skills/pass/FLOW_PASS_BENEFIT_TRACKING.md skills/admin/pass/
```

- [ ] **Step 5: Commit**

```bash
git add -A skills/admin skills/offpix skills/pass
git commit -m "refactor: move admin contracts, kyc, loyalty, pass docs"
```

---

### Task 8: Move `admin/` files — pdf, settings, tokenization, user-profile, whitelist

- [ ] **Step 1: Move admin pdf**

```bash
cd /c/Users/ferna/W3block/public-skills
git mv skills/offpix/pdf/PDF_SKILL_INDEX.md skills/admin/pdf/
git mv skills/offpix/pdf/FLOW_PDF_CERTIFICATE_GENERATION.md skills/admin/pdf/
git mv skills/offpix/pdf/FLOW_PDF_PASS_GENERATION.md skills/admin/pdf/
```

- [ ] **Step 2: Move admin settings**

```bash
cd /c/Users/ferna/W3block/public-skills
git mv skills/offpix/settings/SETTINGS_SKILL_INDEX.md skills/admin/settings/
git mv skills/offpix/settings/FLOW_SETTINGS_BILLING_MANAGEMENT.md skills/admin/settings/
git mv skills/offpix/settings/FLOW_SETTINGS_TENANT_CONFIGURATION.md skills/admin/settings/
```

- [ ] **Step 3: Move admin tokenization**

```bash
cd /c/Users/ferna/W3block/public-skills
git mv skills/offpix/tokenization/TOKENIZATION_SKILL_INDEX.md skills/admin/tokenization/
git mv skills/offpix/tokenization/FLOW_TOKENIZATION_COLLECTION_LIFECYCLE.md skills/admin/tokenization/
git mv skills/offpix/tokenization/FLOW_TOKENIZATION_EDITION_OPERATIONS.md skills/admin/tokenization/
git mv skills/offpix/tokenization/FLOW_TOKENIZATION_BULK_IMPORT.md skills/admin/tokenization/
```

- [ ] **Step 4: Move admin user-profile & whitelist**

```bash
cd /c/Users/ferna/W3block/public-skills
git mv skills/user-profile/USER_PROFILE_SKILL_INDEX.md skills/admin/user-profile/
git mv skills/user-profile/FLOW_PROFILE_WITHDRAWALS.md skills/admin/user-profile/
git mv skills/offpix/whitelist/WHITELIST_SKILL_INDEX.md skills/admin/whitelist/
git mv skills/offpix/whitelist/FLOW_WHITELIST_MANAGEMENT.md skills/admin/whitelist/
```

- [ ] **Step 5: Commit**

```bash
git add -A skills/admin skills/offpix skills/user-profile
git commit -m "refactor: move admin pdf, settings, tokenization, user-profile, whitelist docs"
```

---

### Task 9: Clean up empty old directories

- [ ] **Step 1: Remove empty directories**

```bash
cd /c/Users/ferna/W3block/public-skills
find skills/offpix skills/setup skills/auth skills/checkout skills/tokens skills/pass skills/user-profile -type d -empty -delete 2>/dev/null
```

- [ ] **Step 2: Verify no old directories remain**

```bash
ls -la skills/ | grep -v "^total"
# Expected: _meta  _setup  admin  references  user  (and no old dirs)
```

- [ ] **Step 3: Verify all files are accounted for**

```bash
find skills -type f -name "*.md" | wc -l
# Expected: 86 files (same total as before)
```

- [ ] **Step 4: Commit cleanup**

```bash
git add -A
git commit -m "refactor: remove empty legacy directories"
```

---

### Task 10: Final verification

- [ ] **Step 1: Print final tree**

```bash
find skills -type f -name "*.md" | sort
```

- [ ] **Step 2: Verify git status is clean**

```bash
git status
```
