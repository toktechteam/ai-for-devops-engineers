# Chapter 0 — Start Here: Safe AWS Setup Before Any AI Work

---

**Cost:** ₹0 (the card verification charge is refunded) · **Time:** 45–60 minutes · **Cleanup:** none — everything in this chapter is kept for the rest of the book.

> 🎥 **Video walkthrough:** the full chapter, step by step.
> `🎥 [VIDEO ch00 | QR code + short link | 8–12 min | public YouTube]`
> Everything is also written out below, so the video is optional.

**The purpose of this chapter, in one sentence:** get your AWS account into a safe, ready state — locked down, cost-alerted, and in the right region — so every lab after this one is about AI, not about fixing account problems.

**Why anyone should spend 45–60 minutes on this before touching AI at all:** almost every "it didn't work" moment in the rest of this book traces back to one of three things — the wrong region, no cost alert so a mistake goes unnoticed, or an identity (root) that should never be used day to day. This chapter removes all three in one sitting, once, so you never have to think about them again.

**By the end of this chapter you will have:**

- An AWS account with **$100 in free credits** (with a path to $200)
- The root user locked down with MFA
- A separate IAM admin user for daily work
- A **$5 budget alert** on your email (this step also earns **+$20** in credits)
- The region set to **us-east-1**
- CloudShell open and verified with 6 commands

---

## Why this chapter exists

Every AI lab in this book runs inside an AWS account. Before writing a single line of AI code, that account needs the same basic hygiene you'd expect on any production system: a locked-down root identity, a working-day identity with defined permissions, and a cost alert so nothing runs up a bill unnoticed.

This chapter sets that up once. It also gets you into **CloudShell** — a terminal that runs in your browser, with the AWS CLI already installed — so no software needs to be installed on your laptop for any lab in this book.

**What you'll actually learn here:** the exact one-time account setup every AWS course skips over — root vs. an admin user, why permissions and billing visibility are split by default, and how a budget alert turns "I hope nothing goes wrong" into "I'll get an email if something does." You'll use this same setup pattern in any AWS account you ever work in, lab or production.

---

## Concept: treat your AWS account like a new production server

| What you already do for a new server | What this chapter does for AWS |
|---|---|
| `root`/administrator login used only for emergencies | **Root user** = the email you signed up with. Lock it with MFA; don't use it day to day. |
| A named account for daily work, with defined permissions | An **IAM admin user** for daily work |
| MFA or key-based access | **MFA** on both the root user and the admin user |
| A disk or spend alert at a set threshold | A **budget alert** at 80% of $5 |
| A data center close to your users | A **region**. This book uses **us-east-1** (N. Virginia), where Bedrock models become available first. |
| A prepared jump box with tools installed | **CloudShell** — a browser-based terminal, already signed in, with the AWS CLI pre-installed |

### The free credits, briefly

Since mid-2025, new AWS accounts receive:

- **$100** automatically at sign-up.
- **Up to $100 more**, $20 each, for five short activities: launching and terminating an EC2 instance, creating an RDS database, building a Lambda function with a Function URL, running a prompt in the Bedrock playground, and setting up a budget. This book naturally covers four of the five.
- Credits are valid for **6 months**.

**Free plan or Paid plan?** You choose one at sign-up.

| | Free plan | Paid plan |
|---|---|---|
| Can you be charged? | No — usage stops when credits run out | Yes, beyond your credit balance |
| After 6 months | Account closes unless you upgrade (data kept 90 days) | Keeps running |
| Service access | Most commonly used services (sufficient for this book) | All services |

**Recommendation:** choose the **Free plan**. It removes any risk of an unexpected bill while you're learning, and you can upgrade later with one click. Aim to finish the labs and push your code to GitHub within the 6-month window.

---

## Setup, Part A: account and IAM

### Step 1: Create the AWS account and secure the root user

Follow AWS's own account setup guide, which covers account creation, the root user, and enabling MFA:

📄 **AWS official guide:** [Setting up your environment – Module 1](https://docs.aws.amazon.com/hands-on/latest/setup-environment/module-one.html)

You'll need an email you check daily, a phone number, and a debit or credit card with **international/online transactions enabled** — the most common reason sign-ups fail in India, so turn this on in your bank app before you start.

Follow the guide through:
1. Account creation and email verification
2. Choosing the **Free plan**
3. Card verification (a small refundable charge, typically about ₹2)
4. Enabling **MFA on the root user** — use an authenticator app (Google Authenticator, Microsoft Authenticator, Authy) or a passkey

> Account activation can take anywhere from a few minutes to a few hours.

### Step 2: Create your IAM admin user

Follow AWS's guide to setting up an initial IAM identity:

📄 **AWS official guide:** [Getting started: create an administrative user](https://docs.aws.amazon.com/IAM/latest/UserGuide/getting-started-account-iam.html)

For this book, create the user with the following:

- **User name:** `opskart-admin`
- **Console access:** enabled, with a custom password
- **Permissions:** attach the **AdministratorAccess** policy directly
- **MFA:** enabled on this user as well, the same way you enabled it on root
- Save the **console sign-in URL** shown after creation (it looks like `https://123456789012.signin.aws.amazon.com/console`) along with the password

Once this user exists and has MFA enabled, **sign out of root** and sign back in as `opskart-admin`. From this point on, root is used only for account-level tasks, not daily work.

> **In a company setting:** most organizations use **IAM Identity Center** (SSO) rather than individual IAM users, so staff sign in once with a company identity and receive short-lived credentials. For a personal learning account, an IAM admin user with MFA is simpler and sufficient. This distinction comes up in interviews — see the questions at the end of this chapter.

### Step 3: Allow the admin user to see billing information

By default, even an administrator IAM user cannot view Billing and Budgets — only the root user can enable this.

1. Sign in as **root** → top right, **account name → Account**.
2. Scroll to **IAM user and role access to Billing information → Edit**.
3. Enable **Activate IAM Access → Update**.

### Step 4: Set the region to us-east-1

Signed in as `opskart-admin`: top right, next to your user name → region dropdown → **US East (N. Virginia) us-east-1**.

> AWS resources are scoped to a region. If a resource ever appears to be "missing," check the region selector first.

### Step 5: Create a $5 budget alert (+$20 credits)

1. Signed in as `opskart-admin`: search bar → **Billing and Cost Management → Budgets → Create budget**.
2. Choose **Customize (advanced) → Cost budget → Next**.
3. Name: `opskart-5usd-alert` · Period: **Monthly** · Renewal: **Recurring** · Method: **Fixed** · Amount: **5.00**.
4. **Budget scope → Advanced options → Charge types:** **disable Credits.**

   Credits offset your bill, so with Credits included your cost always shows as $0 and the alert never triggers. Excluding Credits shows the actual spend.
5. **Next → Add an alert threshold:**
   - Alert 1: **80%** of budgeted amount, trigger **Actual**, notify your email
   - Alert 2: **100%**, trigger **Forecasted**, notify your email
6. **Next → Next → Create budget.**

> Every lab in this book costs well under $1. An alert at $4 or above means something was left running — see Chapter 10 (Cleanup checklist).

---

## Setup, Part B: CloudShell

Signed in as `opskart-admin`, region **us-east-1**: click the terminal icon in the top navigation bar, or search for "CloudShell." The first launch takes 30–60 seconds.

**CloudShell reference:**

| Property | Value |
|---|---|
| Cost | Free |
| Authentication | Already signed in as your console user — no access keys needed |
| Persistent storage | 1 GB in your home directory (`~`), per region |
| Idle timeout | Session ends after roughly 20–30 minutes of inactivity; files in `~` are preserved |
| Maximum session length | About 12 hours |
| Storage retention | Deleted if CloudShell is unused in a region for 120 days |

Run the following six commands and compare the output.

#### Command 1: Confirm your identity

```bash
aws sts get-caller-identity
```

**Expected output:**

```json
{
    "UserId": "AIDAXXXXXXXXXXXXXXXXX",
    "Account": "123456789012",
    "Arn": "arn:aws:iam::123456789012:user/opskart-admin"
}
```

The `Arn` should end with `user/opskart-admin`. If it ends with `:root`, sign out and sign back in as the admin user.

#### Command 2: Confirm the region

```bash
echo $AWS_REGION
```

**Expected output:**

```
us-east-1
```

CloudShell uses whichever region is selected in the console. If this is wrong, close CloudShell, switch the region, and reopen it.

#### Command 3: Confirm root MFA is enabled

```bash
aws iam get-account-summary --query 'SummaryMap.AccountMFAEnabled'
```

**Expected output:**

```
1
```

`1` confirms root MFA is on. `0` means Step 1 needs to be revisited.

#### Command 4: Set an account alias

```bash
aws iam create-account-alias --account-alias my-ai-lab-2026
aws iam list-account-aliases --output text
```

Replace `my-ai-lab-2026` with a name of your choice — it must be unique across all of AWS, using lowercase letters, numbers and hyphens only.

**Expected output:**

```
ACCOUNTALIASES	my-ai-lab-2026
```

Your sign-in URL becomes `https://my-ai-lab-2026.signin.aws.amazon.com/console`, which is easier to remember than the 12-digit account ID.

#### Command 5: Confirm the budget exists

```bash
ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
aws budgets describe-budgets --account-id $ACCOUNT_ID \
  --query 'Budgets[].[BudgetName,BudgetLimit.Amount,BudgetLimit.Unit,TimeUnit]' \
  --output table
```

**Expected output:**

```
---------------------------------------------------------
|                    DescribeBudgets                    |
+---------------------+-------+------+------------------+
|  opskart-5usd-alert |  5.0  |  USD |  MONTHLY         |
+---------------------+-------+------+------------------+
```

#### Command 6: Confirm the toolchain and create a test file

```bash
aws --version
python3 --version
df -h ~
echo "Day 0 done by $(whoami) on $(date)" > ~/day0.txt
cat ~/day0.txt
```

**Expected output (versions may differ):**

```
aws-cli/2.x.x Python/3.x.x Linux/... exec-env/CloudShell exe/x86_64...
Python 3.x.x
Filesystem      Size  Used Avail Use% Mounted on
/dev/loop0      974M   ...  ...    ...  /home
Day 0 done by cloudshell-user on Thu Sep 24 ... UTC 2026
```

AWS CLI **v2** is required for later labs (specifically the `converse` command).

### Uploading and downloading files (used in Chapters 3 and 8)

- **Download:** CloudShell → **Actions → Download file** → enter the full path, e.g. `/home/cloudshell-user/day0.txt` → **Download**.
- **Upload:** **Actions → Upload file** → select a file from your computer. It lands in `/home/cloudshell-user/`.
- For files larger than a few MB, use S3 instead — it's faster.

---

## Troubleshooting

| Issue | Cause | Fix |
|---|---|---|
| Card declined during sign-up | International/online transactions disabled on the card | Enable them in your bank app, then retry with the same or a different card |
| "Your account is being activated" for an extended period | Standard activation check | Wait up to 24 hours; check email for any verification request |
| Budgets page shows "You don't have permission" as admin | Step 3 (IAM billing access) was skipped | Sign in as root → Account → **Activate IAM Access** |
| Budget alert never triggers | Credits are included in the cost calculation | Edit the budget → Charge types → disable **Credits** |
| `get-caller-identity` returns `:root` | CloudShell opened while signed in as root | Sign out, sign back in as the admin user |
| `echo $AWS_REGION` returns the wrong region | Console region was set incorrectly | Switch to us-east-1 in the console, then reopen CloudShell |
| `create-account-alias` returns `EntityAlreadyExists` | The alias is already taken account-wide | Choose a more specific alias, e.g. add your name |
| CloudShell shows "Session ended" | 20–30 minutes of inactivity | Click **Restart** — files in `~` are preserved |
| Paste doesn't work in CloudShell | Browser keyboard shortcut conflict | Use **Ctrl+Shift+V**, or right-click → Paste |

---

## Interview questions

**1. Why shouldn't the root user be used for daily work?**
Root has account-wide permissions that cannot be restricted by IAM policy, and it can close the account or change billing settings. If it's compromised, everything is exposed. Standard practice: enable MFA on root, avoid creating access keys for it, and use a separate identity for daily work.

**2. IAM user vs. IAM Identity Center — which would you use in a company?**
**IAM Identity Center** (SSO). Users authenticate once with a corporate identity provider (Google Workspace, Azure AD, Okta) and receive short-lived credentials, with access managed centrally across multiple accounts. IAM users have long-lived passwords and keys, which is better suited to break-glass access or a single personal account.

**3. How do you prevent an unexpected AWS bill?**
Several layers: AWS Budgets with both actual and forecasted alerts, Cost Anomaly Detection, tagging so every resource has an owner, scheduled shutdown or cleanup automation for non-production resources, and, at the organization level, Service Control Policies that restrict expensive instance types or regions. Note that alerts only notify — they don't stop spending unless paired with budget actions.

---

## Cleanup

Nothing to remove. Keep the admin user, MFA, budget and account alias — every later chapter depends on them.

The one action to take: sign out of root, and store the root password and MFA device somewhere safe. It shouldn't be needed again in this book.

---

## Chapter 0 checklist

- [ ] Account created on the Free plan; credits visible under Billing → Credits
- [ ] Root MFA enabled (Command 3 returns `1`)
- [ ] IAM billing access activated
- [ ] `opskart-admin` created with AdministratorAccess and MFA
- [ ] Region set to us-east-1
- [ ] Budget `opskart-5usd-alert` created with Credits excluded
- [ ] All 6 CloudShell commands return the expected output
- [ ] `day0.txt` downloaded locally

**Next → Chapter 1: AI in one page for DevOps engineers**, followed by your first Bedrock command.
