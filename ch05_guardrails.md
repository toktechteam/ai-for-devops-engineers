# Chapter 5: Guardrails (Block Secrets Before They Reach the Model, Mask PII Yourself)

💸 **Cost:** well under $0.05 for the whole lab (guardrail checks are billed per 1,000 characters checked; nothing is charged while the guardrail sits unused)
⏱ **Time:** 25–30 minutes
🧹 **Cleanup:** delete the guardrail at the end of this chapter

**The purpose of this chapter, in one sentence:** put a real safety check in front of the AI calls you've been making since Chapter 2, so a secret or a customer's personal data never actually reaches the model in the first place.

**Why anyone should spend 25–30 minutes on this:** Chapter 3 proved that AI is a great log reader. This chapter proves it can be a *safe* log reader too — the chapter you'd point to if a security reviewer ever asked "what stops this AI feature from leaking something it shouldn't?" That question comes up in almost every real AI project, so this is one of the more interview-relevant chapters in the book.

**By the end of this chapter, you will have:**
- A Bedrock Guardrail that **blocks** AWS keys, private keys and database passwords, before anything reaches the model
- A local masking function in Python that replaces emails, IP addresses and phone numbers with placeholder tags — deterministic, and under your own control
- Tested the guardrail from the CLI with `apply-guardrail` — no model call, no model cost
- Upgraded the Chapter 3 log explainer into `safe_explain.py`: every log is checked for secrets first, then scrubbed of PII, before it ever reaches the model

🎥 [VIDEO ch05 | QR code + short link | ~8 min | unlisted]

## Why this matters

In Chapter 3 you sent log files to an AI model. Real logs are messy. Applications print their full config on startup, debug lines dump environment variables, and error messages include connection strings with passwords in them. Auth logs are full of user emails and IP addresses.

If you pipe those logs into a model as-is, you have copied production secrets and personal data to a third place. Nobody reviewed that, and it may break your company's data policy or India's DPDP Act obligations. This is also the first question a security team asks when someone proposes "let's use AI on our logs": *what stops secrets and customer data from going into the model?*

This chapter builds two layers to answer that: a **Bedrock Guardrail** that blocks secrets before anything is sent anywhere, and a **local masking step** in your own Python code that strips personal data out before the (already secret-free) log reaches the model.

## Concept

**DevOps analogy:** the guardrail is `gitleaks` or a pre-commit secret scanner, but for text going to an AI model — it stops a request the same way a scanner stops a commit that contains an AWS key. The local masking step is closer to a log-shipping filter (like a Logstash or Fluent Bit processor) that strips fields before they leave your infrastructure.

**Why two different mechanisms instead of one guardrail doing both?**

| Job | Tool | Why |
|---|---|---|
| **Block secrets** (AWS keys, private keys, DB passwords) | Bedrock Guardrail, `BLOCK` action | Tested and reliable: both the built-in secret types and custom regex fired correctly on every test run. Nothing must be missed here — a leaked key is a real incident. |
| **Mask PII** (emails, IPs, phone numbers) | Your own Python regex, applied locally | During testing, Bedrock Guardrails' `ANONYMIZE` action did not reliably fire — not for the built-in `EMAIL`/`IP_ADDRESS`/`PHONE` types, and not for custom regex configured with `ANONYMIZE`. `BLOCK` worked every time; `ANONYMIZE` did not. Rather than ship a masking step that silently does nothing, this chapter masks PII in code you can see and test yourself. |

This is a genuinely useful lesson beyond this one chapter: **when a managed security feature doesn't behave the way its documentation implies, don't paper over it — verify it, and fall back to something you can prove works.** A guardrail that appears configured but silently lets PII through is worse than no guardrail, because it creates false confidence.

**Where each check sits:**

```
log file
   │
   ▼
ApplyGuardrail (source = INPUT)      ← checks for SECRETS only; no model is called
   │
   ├── secret found  → BLOCKED  → stop. Nothing is sent anywhere else.
   └── no secret     → continue
   │
   ▼
mask_pii() (local Python, re.sub)    ← replaces emails/IPs/phone numbers with tags
   │
   ▼
Converse (Nova Micro)                ← only ever sees secret-free, PII-masked text
```

`ApplyGuardrail` is a standalone API — it checks text against a guardrail **without calling any model**, so you can check a log before deciding whether to send it anywhere. You can also attach a guardrail directly to a `converse()` call with `guardrailConfig`, which checks both the prompt and the model's answer; the same BLOCK-only scope applies there too.

**One honest limit:** a guardrail (or a regex mask) is a safety net, not the fix. The real fix is to stop applications from logging secrets and unmasked personal data in the first place. This chapter catches what slips through.

## Hands-on

All steps run in **CloudShell** (region `us-east-1`), signed in as the `opskart-admin` user from Chapter 0. No EC2 instance is needed for this chapter.

### Step 1: Write the guardrail policy (secrets only)

This guardrail's only job is blocking secrets. It uses both a built-in type and a regex for each one — belt and braces, since even the reliable `BLOCK` action is worth double-checking on a known format:

```bash
mkdir -p ~/ch05 && cd ~/ch05

cat > ~/ch05/sensitive-policy.json <<'EOF'
{
  "piiEntitiesConfig": [
    { "type": "AWS_ACCESS_KEY", "action": "BLOCK" },
    { "type": "AWS_SECRET_KEY", "action": "BLOCK" },
    { "type": "PASSWORD",       "action": "BLOCK" }
  ],
  "regexesConfig": [
    {
      "name": "aws-access-key-id",
      "description": "AWS access key IDs (long-term AKIA and temporary ASIA)",
      "pattern": "\\b(AKIA|ASIA)[0-9A-Z]{16}\\b",
      "action": "BLOCK"
    },
    {
      "name": "private-key-block",
      "description": "PEM private keys (RSA, EC, OpenSSH)",
      "pattern": "-----BEGIN [A-Z ]*PRIVATE KEY-----",
      "action": "BLOCK"
    },
    {
      "name": "db-connection-string",
      "description": "Database URLs with an inline password",
      "pattern": "(postgres|postgresql|mysql|mongodb(\\+srv)?)://[^:\\s]+:[^@\\s]+@",
      "action": "BLOCK"
    }
  ]
}
EOF
```

What each part does:
- **BLOCK**: AWS keys (built-in type and regex), passwords, private keys, and database URLs with a password inside (`postgres://user:password@host`). Nothing in this guardrail masks anything — its only outcome is "let it through" or "block the whole thing."
- The double backslashes (`\\b`, `\\s`) are required because the regex sits inside a JSON string.

### Step 2: Create the guardrail and publish version 1

```bash
GR_ID=$(aws bedrock create-guardrail \
  --region us-east-1 \
  --name opskart-secrets-guardrail \
  --description "Blocks AWS keys, private keys and DB passwords before they reach the model" \
  --sensitive-information-policy-config file://~/ch05/sensitive-policy.json \
  --blocked-input-messaging "Blocked by opskart-secrets-guardrail: the input contains secrets or credentials." \
  --blocked-outputs-messaging "Blocked by opskart-secrets-guardrail: the response contained sensitive data." \
  --query 'guardrailId' --output text)

echo "Guardrail ID: $GR_ID"
```

**Expected output:**

```
Guardrail ID: abc123xyz456
```

(Your ID will be a different random string.)

A new guardrail starts as a `DRAFT`. Publish a fixed version, the same way you'd tag a release instead of deploying `main`:

```bash
aws bedrock create-guardrail-version \
  --region us-east-1 \
  --guardrail-identifier "$GR_ID" \
  --description "v1 - secrets block only" \
  --query '{id: guardrailId, version: version}' --output table
```

**Expected output:**

```
-----------------------------
|  CreateGuardrailVersion   |
+---------------+-----------+
|      id       |  version  |
+---------------+-----------+
|  abc123xyz456 |  1        |
+---------------+-----------+
```

CloudShell forgets shell variables when the session times out, so save the ID to a file:

```bash
echo "export GR_ID=$GR_ID" > ~/ch05/guardrail.env
```

If you come back later, run `source ~/ch05/guardrail.env` first.

### Step 3: Test the guardrail from the CLI (no model involved)

`apply-guardrail` checks text against the guardrail and returns the result. It is quick and costs very little, because no model is called.

**Test A: clean text (should pass untouched)**

```bash
aws bedrock-runtime apply-guardrail \
  --region us-east-1 \
  --guardrail-identifier "$GR_ID" \
  --guardrail-version 1 \
  --source INPUT \
  --content '[{"text":{"text":"Pod api-7f9c restarted 3 times: OOMKilled, memory limit 256Mi."}}]' \
  --query '{action: action, output: outputs[0].text}'
```

**Expected output:**

```json
{
    "action": "NONE",
    "output": null
}
```

`NONE` means nothing was found. `output` is `null` because there is nothing to replace, so you use your original text.

**Test B: PII alone, no secrets (should pass through untouched)**

This guardrail only watches for secrets, so a line with an email and an IP address — but no key or password — should also come back as `NONE`. Masking that happens in your own code (Step 5), not here:

```bash
aws bedrock-runtime apply-guardrail \
  --region us-east-1 \
  --guardrail-identifier "$GR_ID" \
  --guardrail-version 1 \
  --source INPUT \
  --content '[{"text":{"text":"Failed login for user dev.team@example.com from 10.24.8.17"}}]' \
  --query '{action: action, output: outputs[0].text}'
```

**Expected output:**

```json
{
    "action": "NONE",
    "output": null
}
```

This is correct and expected for this guardrail — it isn't supposed to touch PII. Step 5 handles that separately.

**Test C: a secret (should be blocked)**

The key below is AWS's official documentation example key. It is not a real credential. Never test with a real one.

```bash
aws bedrock-runtime apply-guardrail \
  --region us-east-1 \
  --guardrail-identifier "$GR_ID" \
  --guardrail-version 1 \
  --source INPUT \
  --content '[{"text":{"text":"Deploy failed. Env: AWS_ACCESS_KEY_ID=AKIAIOSFODNN7EXAMPLE"}}]' \
  --query '{action: action, output: outputs[0].text}'
```

**Expected output:**

```json
{
    "action": "GUARDRAIL_INTERVENED",
    "output": "Blocked by opskart-secrets-guardrail: the input contains secrets or credentials."
}
```

The whole input is replaced by your blocked message.

### Step 4: Create two realistic sample logs

**`leaky-app.log`** is a payment service that prints its database URL and an AWS key while starting up, then fails to connect. It should be **blocked**:

```bash
mkdir -p ~/ch05/logs

cat > ~/ch05/logs/leaky-app.log <<'EOF'
2026-09-29T10:14:02.118Z INFO  [payment-svc] Starting payment-svc v2.4.1 (pid 4121) on host ip-172-31-20-14
2026-09-29T10:14:02.301Z INFO  [payment-svc] Loading configuration from environment
2026-09-29T10:14:02.302Z DEBUG [payment-svc] DATABASE_URL=postgres://payments_app:Pr0d-S3cret-2026@payments-db.internal:5432/payments
2026-09-29T10:14:02.303Z DEBUG [payment-svc] AWS_ACCESS_KEY_ID=AKIAIOSFODNN7EXAMPLE
2026-09-29T10:14:02.304Z DEBUG [payment-svc] AWS_REGION=us-east-1
2026-09-29T10:14:02.410Z INFO  [payment-svc] HikariPool-1 - Starting...
2026-09-29T10:14:07.512Z ERROR [payment-svc] HikariPool-1 - Connection is not available, request timed out after 5000ms
2026-09-29T10:14:07.513Z WARN  [payment-svc] Unable to acquire JDBC connection to payments-db.internal:5432; retrying (1/3)
2026-09-29T10:14:12.720Z ERROR [payment-svc] HikariPool-1 - Connection is not available, request timed out after 5000ms
2026-09-29T10:14:12.721Z WARN  [payment-svc] Unable to acquire JDBC connection to payments-db.internal:5432; retrying (2/3)
2026-09-29T10:14:17.899Z ERROR [payment-svc] HikariPool-1 - Connection is not available, request timed out after 5000ms
2026-09-29T10:14:17.900Z WARN  [payment-svc] Unable to acquire JDBC connection to payments-db.internal:5432; retrying (3/3)
2026-09-29T10:14:22.901Z FATAL [payment-svc] Could not connect to database after 3 attempts: java.net.SocketTimeoutException: connect timed out
2026-09-29T10:14:22.905Z INFO  [payment-svc] Shutting down (exit code 1)
EOF
```

**`auth-service.log`** is an auth service under a brute-force login attack. It contains emails, IPs and a phone number but no secrets, so it should pass the guardrail, then get **masked locally, then explained**:

```bash
cat > ~/ch05/logs/auth-service.log <<'EOF'
2026-09-29T11:02:10.004Z INFO  [auth-svc] Listening on :8443 (TLS enabled)
2026-09-29T11:02:41.337Z WARN  [auth-svc] Failed login for user dev.team@example.com from 203.0.113.50 (reason=bad_password)
2026-09-29T11:02:41.902Z WARN  [auth-svc] Failed login for user ops.lead@example.com from 203.0.113.50 (reason=bad_password)
2026-09-29T11:02:42.415Z WARN  [auth-svc] Failed login for user billing@example.com from 203.0.113.50 (reason=unknown_user)
2026-09-29T11:02:42.988Z WARN  [auth-svc] Failed login for user admin@example.com from 203.0.113.50 (reason=unknown_user)
2026-09-29T11:02:43.501Z WARN  [auth-svc] Failed login for user ops.lead@example.com from 203.0.113.50 (reason=bad_password)
2026-09-29T11:02:44.120Z WARN  [auth-svc] Failed login for user support@example.com from 203.0.113.50 (reason=unknown_user)
2026-09-29T11:02:44.733Z WARN  [auth-svc] Failed login for user ops.lead@example.com from 203.0.113.50 (reason=bad_password)
2026-09-29T11:02:45.310Z INFO  [auth-svc] Successful login for user qa.team@example.com from 10.24.8.17
2026-09-29T11:02:45.842Z WARN  [auth-svc] Failed login for user ops.lead@example.com from 203.0.113.50 (reason=bad_password)
2026-09-29T11:02:46.401Z WARN  [auth-svc] Failed login for user ops.lead@example.com from 203.0.113.50 (reason=bad_password)
2026-09-29T11:02:46.402Z ERROR [auth-svc] Account locked: ops.lead@example.com after 5 failed attempts in 5s
2026-09-29T11:03:15.771Z WARN  [auth-svc] 25 failed logins from 203.0.113.50 in 35s across 6 distinct usernames; no rate limit configured for /v1/login
2026-09-29T11:03:16.002Z ERROR [auth-svc] Paging on-call engineer at +1-555-0100: possible credential-stuffing attack
EOF

ls -1 ~/ch05/logs
```

**Expected output:**

```
auth-service.log
leaky-app.log
```

> **GitHub repo:** commit both files to `chapter-05/sample-logs/`, created exactly as above, so the repo and the book stay identical.

### Step 5: Build `safe_explain.py`

This is the Chapter 3 log explainer with two checks in front: the guardrail for secrets, then a local masking pass for PII. First make sure CloudShell's boto3 is new enough to have `apply_guardrail`:

```bash
pip3 install --user --upgrade --quiet boto3
python3 -c "import boto3; print('boto3', boto3.__version__)"
```

**Expected output:** `boto3 1.x.x`. Any recent version works. The upgrade makes sure `apply_guardrail` is available.

Now the script:

```bash
cat > ~/ch05/safe_explain.py <<'EOF'
import os
import re
import sys
import boto3

REGION = "us-east-1"
MODEL_ID = "us.amazon.nova-micro-v1:0"
GUARDRAIL_ID = os.environ.get("GR_ID")
GUARDRAIL_VERSION = "1"
MAX_LINES = 100

# Local PII masking. This runs in plain Python, not through Bedrock Guardrails --
# testing showed the guardrail's ANONYMIZE action does not reliably mask
# built-in PII types or custom regex, while BLOCK worked correctly every time.
# So: the guardrail's only job is blocking secrets; masking PII is done here,
# where the behavior is fully visible and testable.
PII_PATTERNS = [
    ("EMAIL",  re.compile(r"[A-Za-z0-9._%+-]+@[A-Za-z0-9.-]+\.[A-Za-z]{2,}")),
    ("IP",     re.compile(r"\b(?:\d{1,3}\.){3}\d{1,3}\b")),
    ("PHONE",  re.compile(r"\+?\d{1,3}[-.\s]?\d{3}[-.\s]?\d{4}\b")),
]

SYSTEM_PROMPT = (
    "You are a senior DevOps/SRE engineer. You will receive a log excerpt. "
    "Some values are masked as [EMAIL], [IP] or [PHONE]; treat these as placeholders "
    "standing in for real values, not literal text. "
    "Reply with exactly three sections and nothing else:\n"
    "ROOT CAUSE: one or two sentences.\n"
    "FIX: numbered steps an engineer can run now.\n"
    "PREVENT: two or three bullet points to stop it happening again."
)

client = boto3.client("bedrock-runtime", region_name=REGION)

def check_for_secrets(text):
    """Return (blocked, findings). Never returns the matched secret itself."""
    resp = client.apply_guardrail(
        guardrailIdentifier=GUARDRAIL_ID,
        guardrailVersion=GUARDRAIL_VERSION,
        source="INPUT",
        content=[{"text": {"text": text}}],
    )

    findings = []
    for assessment in resp.get("assessments", []):
        sip = assessment.get("sensitiveInformationPolicy", {})
        for entity in sip.get("piiEntities", []):
            findings.append((entity["type"], entity["action"]))
        for regex in sip.get("regexes", []):
            findings.append((regex["name"], regex["action"]))

    blocked = resp["action"] == "GUARDRAIL_INTERVENED"
    return blocked, findings

def mask_pii(text):
    """Replace emails, IPs and phone numbers with placeholder tags. Returns (masked_text, counts)."""
    counts = {}
    for tag, pattern in PII_PATTERNS:
        text, n = pattern.subn(f"[{tag}]", text)
        if n:
            counts[tag] = n
    return text, counts

def main():
    if not GUARDRAIL_ID:
        sys.exit("GR_ID is not set. Run: source ~/ch05/guardrail.env")
    if len(sys.argv) != 2:
        sys.exit("Usage: python3 safe_explain.py <logfile>")

    with open(sys.argv[1], encoding="utf-8", errors="replace") as f:
        log_text = "".join(f.readlines()[-MAX_LINES:])

    print("=== SECRETS CHECK (Bedrock Guardrail) ===")
    blocked, findings = check_for_secrets(log_text)
    if findings:
        for name, action in sorted(set(findings)):
            print(f"  - {name}: {action}")
    else:
        print("  - no secrets found")

    if blocked:
        print("\nBLOCKED: this log contains secrets. Nothing was sent to the model.")
        print("Fix the application so it stops logging credentials, then try again.")
        sys.exit(1)

    print("\n=== PII MASKING (local) ===")
    safe_text, counts = mask_pii(log_text)
    if counts:
        for tag, n in sorted(counts.items()):
            print(f"  - {tag}: {n} masked")
    else:
        print("  - nothing to mask")

    print("\n=== WHAT THE MODEL WILL SEE (first 5 lines) ===")
    print("\n".join(safe_text.splitlines()[:5]))

    response = client.converse(
        modelId=MODEL_ID,
        system=[{"text": SYSTEM_PROMPT}],
        messages=[{"role": "user", "content": [{"text": safe_text}]}],
        inferenceConfig={"temperature": 0.2, "maxTokens": 500},
    )

    print("\n=== AI EXPLANATION ===")
    print(response["output"]["message"]["content"][0]["text"])
    usage = response["usage"]
    print(f"\n[tokens] input={usage['inputTokens']} output={usage['outputTokens']}")

if __name__ == "__main__":
    main()
EOF
```

What changed from Chapter 3:
- **`check_for_secrets()`** runs first on every log, using the Bedrock Guardrail. If it's BLOCKED, the script stops before anything else happens.
- **`mask_pii()`** runs next, entirely in Python with `re.sub`. It replaces emails, IPs and phone numbers with `[EMAIL]`, `[IP]`, `[PHONE]` tags. This is deterministic — you can read exactly what it does, and test it without AWS involved at all.
- **Findings print the type or tag only**, never the matched value. The Bedrock API response does include the matched text for secrets, and printing it would leak the secret into your terminal history and screen recordings.

### Step 6: Run it on both logs

**The leaky log (should be blocked):**

```bash
cd ~/ch05
python3 safe_explain.py logs/leaky-app.log
```

**Expected output** (the list may include extra types such as `PASSWORD`, and the order may differ):

```
=== SECRETS CHECK (Bedrock Guardrail) ===
  - AWS_ACCESS_KEY: BLOCKED
  - aws-access-key-id: BLOCKED
  - db-connection-string: BLOCKED

BLOCKED: this log contains secrets. Nothing was sent to the model.
Fix the application so it stops logging credentials, then try again.
```

No model call happened and no model tokens were used. The secret never left your account boundary for the model.

**The auth log (should pass the guardrail, get masked, then explained):**

```bash
python3 safe_explain.py logs/auth-service.log
```

**Expected output** (the AI wording will differ slightly on each run):

```
=== SECRETS CHECK (Bedrock Guardrail) ===
  - no secrets found

=== PII MASKING (local) ===
  - EMAIL: 8 masked
  - IP: 13 masked
  - PHONE: 1 masked

=== WHAT THE MODEL WILL SEE (first 5 lines) ===
2026-09-29T11:02:10.004Z INFO  [auth-svc] Listening on :8443 (TLS enabled)
2026-09-29T11:02:41.337Z WARN  [auth-svc] Failed login for user [EMAIL] from [IP] (reason=bad_password)
2026-09-29T11:02:41.902Z WARN  [auth-svc] Failed login for user [EMAIL] from [IP] (reason=bad_password)
2026-09-29T11:02:42.415Z WARN  [auth-svc] Failed login for user [EMAIL] from [IP] (reason=unknown_user)
2026-09-29T11:02:42.988Z WARN  [auth-svc] Failed login for user [EMAIL] from [IP] (reason=unknown_user)

=== AI EXPLANATION ===
ROOT CAUSE: A single source is making rapid login attempts against many usernames with no rate limit on /v1/login, which is a credential-stuffing or brute-force attack. It triggered an account lockout.

FIX:
1. Block the offending source IP at the WAF or security group now.
2. Add rate limiting on /v1/login (per IP and per username).
3. Review the locked account and force a password reset if needed.

PREVENT:
- Enable AWS WAF rate-based rules on the login endpoint.
- Add MFA for staff accounts.
- Alert on failed-login spikes per source IP.

[tokens] input=610 output=150
```

The exact masked counts depend on how many lines survive the `MAX_LINES = 100` tail cut, but the shape holds: the model correctly diagnosed a brute-force attack **without ever seeing a real email, IP or phone number**. Masking removed the personal data but kept the pattern that matters: same source, many users, many failures.

> **On attaching a guardrail to `converse()` directly.** You can pass `guardrailConfig={"guardrailIdentifier": GUARDRAIL_ID, "guardrailVersion": "1"}` to `converse()` to have Bedrock check the prompt and the model's answer for secrets in the same call. Since this guardrail is BLOCK-only, that's a reasonable extra layer on the output side too — but it does not replace the local `mask_pii()` step, which is still the only reliable way to scrub PII in this setup.

### Step 7: Cleanup

```bash
aws bedrock delete-guardrail --region us-east-1 --guardrail-identifier "$GR_ID"
aws bedrock list-guardrails --region us-east-1 --query 'guardrails[].name' --output text
```

**Expected output:** the delete command prints nothing. The list command prints nothing, or only other guardrails you created yourself. `opskart-secrets-guardrail` should not be in it.

An unused guardrail costs nothing, but deleting it keeps your account clean. Chapter 8 recreates it with Terraform. Keep `~/ch05/sensitive-policy.json` and `~/ch05/safe_explain.py`, because you'll reuse both.

## Troubleshooting

| Problem | Fix |
|---|---|
| Any `aws` command fails with `Credentials were refreshed, but the refreshed credentials are still expired` | Expired credential variables are left over in your shell (usually from Chapter 4's `eval` step). Run `unset AWS_ACCESS_KEY_ID AWS_SECRET_ACCESS_KEY AWS_SESSION_TOKEN AWS_CREDENTIAL_EXPIRATION`, then `aws sts get-caller-identity`. Or open a new CloudShell tab |
| `ValidationException` on `create-guardrail` mentioning `piiEntitiesConfig` or `type` | A PII type name is misspelled. They are uppercase with underscores: `AWS_ACCESS_KEY` |
| `ValidationException` mentioning a regex `pattern` | A backslash isn't doubled inside the JSON. It must be `\\b`, not `\b`. Recreate the file with the exact heredoc in Step 1 |
| `ConflictException` on `create-guardrail`: name already exists | You already created it. Get the ID with `aws bedrock list-guardrails --region us-east-1 --query "guardrails[?name=='opskart-secrets-guardrail'].id" --output text` |
| `ResourceNotFoundException` on `apply-guardrail`, or `GR_ID is not set` | CloudShell restarted and lost the variable. Run `source ~/ch05/guardrail.env` |
| `ResourceNotFoundException` mentioning version `1` | You skipped `create-guardrail-version` in Step 2. Run it, or test against `--guardrail-version DRAFT` |
| `AttributeError: ... has no attribute 'apply_guardrail'` | Old boto3. Re-run `pip3 install --user --upgrade boto3` |
| Test C doesn't block, returns `"action": "NONE"` | Check the guardrail actually has version 1 published (`aws bedrock get-guardrail ... --guardrail-version 1`), and that you're testing `--guardrail-version 1`, not an unpublished `DRAFT` you forgot to publish |
| `mask_pii()` doesn't mask something you expect | This runs in plain Python `re`, independent of AWS — test the regex directly: `python3 -c "import re; print(re.findall(r'[A-Za-z0-9._%+-]+@[A-Za-z0-9.-]+\.[A-Za-z]{2,}', open('logs/auth-service.log').read()))"` |
| `AccessDeniedException` on `apply_guardrail` when running from EC2 or Lambda (not CloudShell) | The instance or Lambda role needs `bedrock:ApplyGuardrail` on the guardrail ARN (`arn:aws:bedrock:us-east-1:<account-id>:guardrail/<id>`). CloudShell works because `opskart-admin` has AdministratorAccess |

## Interview questions

1. What's the difference between BLOCK and ANONYMIZE in a Bedrock Guardrail, and why does this chapter only rely on BLOCK?
2. Why would you call `ApplyGuardrail` separately before the model call instead of only attaching the guardrail to `converse()`?
3. If a managed AWS security feature (like ANONYMIZE) doesn't behave as documented in your testing, what's the right way to respond — assume it's your config, keep debugging forever, or fall back to something you can verify? How do you decide?
4. Your security team says, "We can't use AI on logs, they contain customer data." How would you answer, and what would you still need to fix at the source?

## Checklist

- [ ] Wrote `sensitive-policy.json` with BLOCK for secrets only
- [ ] Created `opskart-secrets-guardrail` and published version 1
- [ ] Ran all three CLI tests: clean → `NONE`, PII-only → `NONE`, secret → blocked
- [ ] Created `leaky-app.log` and `auth-service.log`
- [ ] `safe_explain.py` blocked the leaky log with **no model call**
- [ ] `safe_explain.py` masked PII locally and explained the auth log
- [ ] Deleted the guardrail and kept `sensitive-policy.json` + `safe_explain.py` for later chapters
