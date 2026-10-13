# Chapter 2 — Your First AI Call from a Server (EC2 + Bedrock, zero access keys)

---

**💸 Cost:** less than ₹5, paid from free credits (t3.micro about $0.01/hour + Nova Micro costs fractions of a paisa per call) · **⏱ Time:** 40–50 minutes · **🧹 Cleanup:** terminate the EC2 instance (Step 7). **Keep** the IAM role for Chapter 3.

**The purpose of this chapter, in one sentence:** call a real AI model from a real server, without ever putting a secret key on that server — the single habit that separates "I made an AI demo" from "I built this the way a production team would."

**Why anyone should spend 40–50 minutes on this:** this is the chapter that makes everything after it make sense. Chapters 3 through 7 all reuse the identity and the calling pattern you build here. If you understand *why* a server can call an AI model with no key anywhere on disk, every later chapter is just "the same trick, different wrapper" — Lambda instead of EC2, a guardrail in front, a dashboard behind. Skip this and the rest of the book feels like memorized commands instead of one idea applied five times.

**By the end of this chapter you will have:**

- ✅ An EC2 server that calls an AI model with **no access keys anywhere**
- ✅ An AI answer from the **AWS CLI** and from a **Python script** (copy-paste; no coding)
- ✅ Token usage printed for every call (this is how AI cost is measured)
- ✅ A least-privilege IAM policy you can explain in an interview
- ✅ Possibly **+$40 credits** (EC2 launch/terminate + Bedrock playground activities)

> 🎥 **Watch me do it:** the whole lab as one video.
> `🎥 [VIDEO ch02 | QR code + short link | 10–12 min | public YouTube]`
> Every step is written below. You don't need the video to finish the lab.

---

## Why this matters

**The problem with the usual tutorial:** most introductory tutorials run `aws configure`, paste in an access key, and call it done. That works for a five-minute demo, but a key stored on a server is a key that can be copied — from disk, from a backup, or from an accidental commit to a public repo. Once it's out, anyone can use it.

**What you'll actually learn here:** production systems avoid access keys entirely. Instead of handing a server a key, you hand it an **IAM role**. AWS issues temporary credentials to anything running with that role, rotates them automatically, and there is never a long-lived secret sitting on disk to leak. This chapter builds that setup end to end — the role, the policy, the server — then calls an AI model with it, from the CLI and from Python, and prints what the call actually cost.

**Why this is worth doing on a server at all**, instead of just running a script on your laptop: a server is the closest thing to how this works in a real product — an app, a bot, or a backend job that needs to call an AI model on its own, with nobody sitting there to type in a password. If a server can do it with zero stored secrets, so can anything else you build later.

---

## Concept: an IAM role is a visitor badge, not a house key

**The old way (access keys):** you copy the house key and hand it to the server. The key works forever until someone remembers to change the lock. If it gets copied, anyone can walk in.

**The right way (IAM role + instance profile):** the server gets a **visitor badge** at the gate. The badge says exactly which rooms it may enter (`bedrock:InvokeModel` on one model only). It **expires automatically** and a new one is issued without anyone touching it.

```
  ┌──────────────┐   1. "Who am I?"      ┌───────────────────────┐
  │  EC2 server  │ ────────────────────▶ │ Instance metadata     │
  │ (no keys!)   │ ◀──────────────────── │ (IMDS, inside AWS)    │
  └──────┬───────┘   2. temp credentials └───────────────────────┘
         │              from the IAM role (auto-rotated)
         │ 3. converse(prompt)  (signed with temp credentials)
         ▼
  ┌──────────────────────────┐
  │ Amazon Bedrock           │  IAM checks: is this role allowed
  │ Nova Micro model         │  to call THIS model? ✅ → answer
  └──────────────────────────┘
```

| You already know (DevOps) | In this lab |
|---|---|
| Kubernetes ServiceAccount + IRSA | EC2 **instance profile** + IAM role |
| Jenkins credentials store | Nothing to store; AWS gives temporary credentials |
| SSH with a `.pem` key | **Session Manager**: no SSH key, no port 22 open |
| `curl` to an API | `aws bedrock-runtime converse` to an AI model |

> **⏪ 60-second recap (for readers who skipped Chapter 1)**
> - **Amazon Bedrock** = AWS's service for calling AI models through an API. You don't run any GPU; you pay per use.
> - **Nova Micro** = Amazon's smallest, cheapest text model. It's fast and good enough for logs, summaries and Q&A.
> - **Tokens** = pieces of words. You pay per input token + per output token. About 1,000 tokens ≈ 750 English words.
> - **`us.amazon.nova-micro-v1:0`** = an *inference profile* ID. The `us.` prefix lets AWS route your call across US regions for capacity. You must use this ID, not the plain model name.
> - Amazon models need **no signup form**; Bedrock serverless models are switched on by default.

---

## Hands-on

**Where you run things:**

- Steps 1–3 → **CloudShell** (signed in as `opskart-admin`, region **us-east-1**)
- Steps 4–6 → **inside the EC2 server** (through Session Manager)
- Step 7 → **CloudShell** again

---

### Step 1 (CloudShell): Create the IAM role, the "visitor badge"

Copy the **whole block**, paste it, and press Enter.

```bash
ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)

# 1a. Trust policy: "EC2 servers are allowed to wear this badge"
cat > trust.json <<'EOF'
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Principal": { "Service": "ec2.amazonaws.com" },
    "Action": "sts:AssumeRole"
  }]
}
EOF

# 1b. Permission policy: "…and may call ONLY Nova Micro"
cat > bedrock-policy.json <<EOF
{
  "Version": "2012-10-17",
  "Statement": [{
    "Sid": "InvokeNovaMicroOnly",
    "Effect": "Allow",
    "Action": ["bedrock:InvokeModel", "bedrock:InvokeModelWithResponseStream"],
    "Resource": [
      "arn:aws:bedrock:*:${ACCOUNT_ID}:inference-profile/us.amazon.nova-micro-v1:0",
      "arn:aws:bedrock:*::foundation-model/amazon.nova-micro-v1:0"
    ]
  }]
}
EOF

# 1c. Create the role + attach both policies
aws iam create-role --role-name opskart-bedrock-ec2-role \
  --assume-role-policy-document file://trust.json --query 'Role.Arn' --output text

aws iam put-role-policy --role-name opskart-bedrock-ec2-role \
  --policy-name opskart-bedrock-invoke --policy-document file://bedrock-policy.json

aws iam attach-role-policy --role-name opskart-bedrock-ec2-role \
  --policy-arn arn:aws:iam::aws:policy/AmazonSSMManagedInstanceCore

# 1d. Instance profile = the "badge holder" EC2 actually wears
aws iam create-instance-profile --instance-profile-name opskart-bedrock-profile \
  --query 'InstanceProfile.Arn' --output text

aws iam add-role-to-instance-profile --instance-profile-name opskart-bedrock-profile \
  --role-name opskart-bedrock-ec2-role
```

**Expected output (2 lines; the other commands print nothing on success):**

```
arn:aws:iam::123456789012:role/opskart-bedrock-ec2-role
arn:aws:iam::123456789012:instance-profile/opskart-bedrock-profile
```

✅ **Why two resources in the policy?** The `us.` profile routes your call to the model in *any* US region, so IAM checks both the **profile** and the **model** there. Leave either one out and you get `AccessDenied`.

✅ **Why `AmazonSSMManagedInstanceCore`?** It lets Session Manager connect to the server. That means no SSH key and no open port 22.

---

### Step 2 (CloudShell): Launch the EC2 server wearing the badge

```bash
sleep 15   # IAM needs a few seconds to "spread" the new instance profile

AMI_ID=$(aws ssm get-parameter \
  --name /aws/service/ami-amazon-linux-latest/al2023-ami-kernel-default-x86_64 \
  --query 'Parameter.Value' --output text)
echo "Using AMI: $AMI_ID"

INSTANCE_ID=$(aws ec2 run-instances \
  --image-id $AMI_ID \
  --instance-type t3.micro \
  --iam-instance-profile Name=opskart-bedrock-profile \
  --tag-specifications 'ResourceType=instance,Tags=[{Key=Name,Value=opskart-bedrock-lab}]' \
  --query 'Instances[0].InstanceId' --output text)
echo "Launched: $INSTANCE_ID"
echo $INSTANCE_ID > ~/lab2-instance-id.txt   # saved for cleanup later
```

**Expected output:**

```
Using AMI: ami-0abc1234def567890
Launched: i-0123456789abcdef0
```

✅ No key pair, no security-group rule for SSH. We don't need either. The **latest** Amazon Linux 2023 image is fetched from SSM, so this command never goes out of date.

---

### Step 3 (CloudShell): Wait until the server is ready

```bash
INSTANCE_ID=$(cat ~/lab2-instance-id.txt)
aws ec2 wait instance-running --instance-ids $INSTANCE_ID && echo "✅ Running"

# Session Manager needs 1–3 more minutes. Re-run this until you see "Online":
aws ssm describe-instance-information \
  --filters "Key=InstanceIds,Values=$INSTANCE_ID" \
  --query 'InstanceInformationList[0].PingStatus' --output text
```

**Expected output:**

```
✅ Running
Online
```

✅ If you see `None`, wait 60 seconds and run the second command again. The SSM agent is still starting up.

Now connect: **EC2 console → Instances → tick `opskart-bedrock-lab` → Connect → Session Manager tab → Connect.** A black terminal opens in a new browser tab.

🎥 Video: `[00:00]` connect with Session Manager

---

### Step 4 (inside EC2): Prove there are no keys, only the role

Session Manager starts you in a basic shell. First switch to bash and your home folder:

```bash
bash
cd ~
export AWS_REGION=us-east-1
```

Now ask AWS who this server is:

```bash
aws sts get-caller-identity
ls ~/.aws 2>/dev/null || echo "No ~/.aws folder — zero keys on this server ✅"
```

**Expected output:**

```json
{
    "UserId": "AROAXXXXXXXXXXXXXXXXX:i-0123456789abcdef0",
    "Account": "123456789012",
    "Arn": "arn:aws:sts::123456789012:assumed-role/opskart-bedrock-ec2-role/i-0123456789abcdef0"
}
No ~/.aws folder — zero keys on this server ✅
```

✅ Look at the `Arn`: **`assumed-role/opskart-bedrock-ec2-role`**. The server is wearing the badge, and there isn't a single key file on disk.

⚠️ Forgot `export AWS_REGION=us-east-1`? Later commands fail with *"You must specify a region"*. Every new Session Manager tab needs the export again.

---

### Step 5 (inside EC2): Your first AI call, from the CLI

```bash
aws bedrock-runtime converse \
  --model-id us.amazon.nova-micro-v1:0 \
  --messages '[{"role":"user","content":[{"text":"You are a senior DevOps engineer. In 3 short bullet points, explain why EC2 should use IAM roles instead of access keys."}]}]' \
  --inference-config '{"maxTokens":200,"temperature":0.3}' \
  --query 'output.message.content[0].text' --output text
```

**Expected output (your wording will differ; AI answers vary every time):**

```
- IAM roles provide temporary credentials that rotate automatically, so there are no long-lived secrets to leak.
- Keys stored on a server can be stolen from disk, backups or code repos; roles leave nothing to steal.
- Permissions are managed centrally in IAM and can be changed or revoked without touching the server.
```

🎉 **You just called an AI model from a server with zero keys.** That single command is the foundation of every AI app on AWS.

✅ Want the token count too? Run the same command but change the `--query` to `'usage'`. You'll see `inputTokens`, `outputTokens` and `totalTokens`.

---

### Step 6 (inside EC2): The same call from Python, with a cost meter

**6a. Install the AWS Python library (boto3):**

```bash
sudo dnf install -y python3-pip -q
pip3 install --user -q boto3
python3 -c "import boto3; print('boto3', boto3.__version__)"
```

**Expected output:**

```
boto3 1.x.x
```

**6b. Create `ask.py`.** Copy the whole block. You don't need to understand Python; the comments explain each part.

```bash
cat > ask.py <<'EOF'
import sys, boto3

MODEL_ID = "us.amazon.nova-micro-v1:0"
# Nova Micro price in USD per 1 million tokens (check the Bedrock pricing page for updates)
PRICE_IN, PRICE_OUT = 0.035, 0.14

client = boto3.client("bedrock-runtime", region_name="us-east-1")   # no keys: uses the IAM role

question = " ".join(sys.argv[1:]) or "What is a Kubernetes CrashLoopBackOff? Answer in 3 lines."

resp = client.converse(
    modelId=MODEL_ID,
    system=[{"text": "You are a senior DevOps engineer. Answer briefly and practically."}],
    messages=[{"role": "user", "content": [{"text": question}]}],
    inferenceConfig={"maxTokens": 300, "temperature": 0.3},
)

print(resp["output"]["message"]["content"][0]["text"])

u = resp["usage"]
cost = (u["inputTokens"] * PRICE_IN + u["outputTokens"] * PRICE_OUT) / 1_000_000
print(f"\n--- tokens in={u['inputTokens']} out={u['outputTokens']} "
      f"| latency={resp['metrics']['latencyMs']} ms | cost≈${cost:.6f}")
EOF
```

**6c. Run it:**

```bash
python3 ask.py
python3 ask.py "Give me 3 quick checks when a Linux server disk is 100% full"
```

**Expected output (answer text will differ):**

```
/home/ssm-user/.local/lib/python3.9/site-packages/boto3/compat.py:89: PythonDeprecationWarning:
Boto3 will no longer support Python 3.9 starting April 29, 2026. To continue receiving service
updates, bug fixes, and security updates please upgrade to Python 3.10 or later. More information
can be found here: https://aws.amazon.com/blogs/developer/python-support-policy-updates-for-aws-sdks-and-tools/
  warnings.warn(warning, PythonDeprecationWarning)
A CrashLoopBackOff means the container starts, crashes, and Kubernetes keeps restarting it
with increasing delays. Check `kubectl logs <pod> --previous` for the crash reason, then
`kubectl describe pod` for events like OOMKilled or failed probes.

--- tokens in=35 out=62 | latency=610 ms | cost≈$0.000010
```

⚠️ The `PythonDeprecationWarning` line is expected — Amazon Linux 2023 ships with Python 3.9, and boto3 is warning that support for it ends in April 2026. It doesn't affect this lab; ignore it (or upgrade Python if you want it gone).

✅ Look at the cost line: **about $0.00001 per question**. That's roughly 100,000 questions for ₹85. This is why Nova Micro is our lab model.

✅ `ask.py` is your **first portfolio file**. We reuse it in Chapter 3 (AI log explainer).

---

### 🎁 Bonus (2 minutes, +$20 credits): the Bedrock playground

AWS gives $20 in credits for trying a prompt in the Bedrock console.

1. Console search → **Amazon Bedrock → Playgrounds → Chat / Text**.
2. **Select model → Amazon → Nova Micro**.
3. Paste the same question from Step 5 → **Run**.
4. Compare: the playground and your EC2 server call the **same model through the same API**. The console is just a UI on top.

---

### Step 7 (CloudShell): Cleanup, terminating the server

Type `exit` twice in the Session Manager tab and close it. Back in **CloudShell**:

```bash
INSTANCE_ID=$(cat ~/lab2-instance-id.txt)
aws ec2 terminate-instances --instance-ids $INSTANCE_ID \
  --query 'TerminatingInstances[0].CurrentState.Name' --output text
aws ec2 wait instance-terminated --instance-ids $INSTANCE_ID && echo "✅ Terminated — billing stopped"
```

**Expected output:**

```
shutting-down
✅ Terminated — billing stopped
```

**Keep these for Chapter 3:** the role `opskart-bedrock-ec2-role` and the instance profile `opskart-bedrock-profile`. IAM roles cost nothing.

---

## Troubleshooting

Stuck? Find your error below first (Ctrl+F works). If it's not listed, rewatch that step in the 🎥 video.

| Error / problem | Why it happens | Fix |
|---|---|---|
| `Value (opskart-bedrock-profile) for parameter iamInstanceProfile.name is invalid` | IAM hadn't finished spreading the new profile | Wait 30 seconds and re-run Step 2 |
| `EntityAlreadyExists` in Step 1 | You ran Step 1 twice | Fine; the role already exists. Move on to Step 2. |
| Session Manager **Connect** button is greyed out | The SSM agent isn't online yet | Wait 2–3 minutes; re-run the `describe-instance-information` check until `Online` |
| PingStatus stays `None` for 5+ minutes | No internet route (default VPC deleted, or no public IP) | Check that **VPC → Your VPCs** has a default VPC. New accounts have one; don't delete it. |
| `You must specify a region` | `AWS_REGION` isn't set in this shell | `export AWS_REGION=us-east-1` |
| `Unable to locate credentials` inside EC2 | Instance launched without the profile | EC2 → Actions → Security → **Modify IAM role** → choose `opskart-bedrock-ec2-role` |
| `AccessDeniedException ... not authorized to perform: bedrock:InvokeModel` | The policy is missing the profile or the model ARN | Re-run Step 1b + the `put-role-policy` command. Check that the ARN in the error matches one in the policy. |
| `ValidationException: Invocation of model ID amazon.nova-micro-v1:0 with on-demand throughput isn't supported` | Plain model ID used instead of the profile | Use **`us.amazon.nova-micro-v1:0`** (with `us.`) |
| `aws: error: argument operation: Invalid choice ... converse` | Old AWS CLI on the server | `sudo dnf upgrade -y awscli-2`, then retry |
| `pip3: command not found` | pip isn't installed by default on AL2023 | `sudo dnf install -y python3-pip` |
| `PythonDeprecationWarning: Boto3 will no longer support Python 3.9...` when running `ask.py` | AL2023's default Python is 3.9; boto3 warns about its own future support cutoff | Harmless — the script still runs and prints the answer below the warning. Ignore it, or install Python 3.10+ if you want it gone. |
| Pasted command runs as garbage / stuck at `>` | The block was cut off (a missing `EOF` or quote) | Press **Ctrl+C** and paste the whole block again |

---

## Interview questions

**Q1. How does an EC2 instance get AWS credentials without access keys?**
Through an **instance profile** that holds an IAM role. The AWS SDK or CLI asks the **instance metadata service (IMDS)** for credentials. IMDS returns **temporary credentials** from STS, and they rotate automatically before they expire. Nothing is stored on disk. With **IMDSv2** (the default on AL2023), every request needs a session token, which blocks SSRF-style credential theft.

**Q2. Why did you give the role both an inference-profile ARN and a foundation-model ARN?**
A **cross-region inference profile** (`us.` prefix) can route the request to the model in several US regions. IAM authorizes the call on the profile *and* on the underlying model in whichever region serves it. So the policy needs both, and I used `*` for region only in the model ARN, while still limiting access to **one specific model**. That's least privilege: the role can't call any other model.

**Q3. Session Manager vs SSH: why use Session Manager?**
There's no SSH key to manage or leak, **no inbound port 22**, access is controlled by **IAM**, and every session can be **logged** to CloudWatch/S3 for audit. Many companies block SSH completely in production and allow only Session Manager.

---

## Chapter 2 checklist

- [ ] Role `opskart-bedrock-ec2-role` created with 2 policies
- [ ] EC2 launched with the instance profile, and **no key pair**
- [ ] Connected with Session Manager
- [ ] `get-caller-identity` shows **assumed-role**
- [ ] CLI `converse` returned an AI answer
- [ ] `ask.py` printed an answer + token count + cost
- [ ] (Bonus) Bedrock playground prompt run, for +$20 credits
- [ ] EC2 **terminated**, and the role kept

**Next → Chapter 3: AI log explainer.** We feed a real error log to `ask.py` and get the root cause and the fix back.
