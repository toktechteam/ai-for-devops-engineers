# Chapter 3 — AI Log Explainer (feed a real error log to AI, get the root cause and fix)

---

**Cost:** under ₹5 from free credits (t3.micro at about $0.01/hour; each explanation costs a fraction of a paisa) · **Time:** 30–40 minutes · **Cleanup:** terminate the EC2 instance (Step 6). Keep the IAM role for later chapters.

**The purpose of this chapter, in one sentence:** turn the raw AI call from Chapter 2 into a tool you'd actually reach for at 2 a.m. — point it at any error log, and get back a plain-English root cause and fix instead of a wall of text to decode yourself.

**Why anyone should spend 30–40 minutes on this:** reading logs under pressure is most of what the job actually looks like day to day, more than any single AWS service. This chapter is the first one in the book that produces something genuinely useful on its own — not just a proof that AI works, but a script you'll likely keep using on real incidents after you finish the lab.

**By the end of this chapter you will have:**

- A script that takes any error log and returns a plain-English **root cause and fix**
- Hands-on practice piping real files into an AI model from a server
- Three worked examples on logs DevOps engineers hit constantly (Kubernetes, Nginx, Linux OOM)
- A second portfolio-ready script in your GitHub repo

> 🎥 **Video walkthrough:** the full lab, step by step.
> `🎥 [VIDEO ch03 | QR code + short link | 8–10 min | public YouTube]`
> Everything is written out below, so the video is optional.

---

## Why this matters

Reading logs is most of the job. A pod crashes, a gateway throws 502s, a process gets killed overnight — and the answer is usually somewhere in a wall of log lines you have to scan under pressure. Junior engineers lose hours here; senior engineers pattern-match in seconds because they've seen it before.

An AI model has "seen" millions of these logs. If you pipe an error log to it with the right instruction, it will point at the likely root cause and suggest concrete fixes in seconds. This chapter turns the `ask.py` idea from Chapter 2 into a purpose-built **log explainer** you can run on any log file — a genuinely useful tool, not just a demo.

This is not about replacing your own judgment. The AI gives you a fast first hypothesis; you still confirm it. But going from "where do I even start" to "check the memory limits first" is exactly the speed-up that makes you look senior.

---

## Concept: an AI log explainer is a smarter `grep`

You already pipe logs through tools to make sense of them. This adds one more tool to that chain — one that understands the log instead of just matching text.

| What you do today | What this lab adds |
|---|---|
| `grep -i error app.log` to find error lines | Send those lines to a model that explains *why* the error happens |
| Paste a stack trace into a search engine | Get a targeted answer without leaving the terminal |
| Ask a senior colleague "what does this mean?" | Get a first opinion instantly, any hour |
| Read the docs for an unfamiliar error | Get the likely cause *and* the fix commands together |

The mechanics are identical to Chapter 2 — the server uses its IAM role, no keys — but instead of a hard-coded question, the script reads a **file** and wraps it in a prompt that asks for a root cause and a fix.

```
  ┌──────────────┐   reads      ┌──────────────┐
  │ error log    │ ───────────▶ │ explain_log  │
  │ (file)       │              │ .py          │
  └──────────────┘              └──────┬───────┘
                                       │ prompt: "explain the root cause
                                       │ and give the fix" + log text
                                       ▼
                                ┌──────────────────────┐
                                │ Amazon Bedrock        │
                                │ Nova Micro            │
                                └──────┬────────────────┘
                                       │ root cause + fix
                                       ▼
                                  your terminal
```

---

## Hands-on

**Where you run things:**

- Steps 1–2 → **CloudShell** (signed in as `opskart-admin`, region **us-east-1**)
- Steps 3–5 → **inside the EC2 server** (through Session Manager)
- Step 6 → **CloudShell** again

---

### Step 1 (CloudShell): Launch a fresh EC2 server with the Chapter 2 role

You deleted the instance at the end of Chapter 2 but kept the IAM role and instance profile. Launch a new instance that wears the same badge.

> If you skipped Chapter 2 or deleted the role, go back and run Chapter 2, Step 1 first — it creates `opskart-bedrock-ec2-role` and `opskart-bedrock-profile`. This chapter needs both.

```bash
AMI_ID=$(aws ssm get-parameter \
  --name /aws/service/ami-amazon-linux-latest/al2023-ami-kernel-default-x86_64 \
  --query 'Parameter.Value' --output text)

INSTANCE_ID=$(aws ec2 run-instances \
  --image-id $AMI_ID \
  --instance-type t3.micro \
  --iam-instance-profile Name=opskart-bedrock-profile \
  --tag-specifications 'ResourceType=instance,Tags=[{Key=Name,Value=opskart-log-explainer}]' \
  --query 'Instances[0].InstanceId' --output text)
echo "Launched: $INSTANCE_ID"
echo $INSTANCE_ID > ~/lab3-instance-id.txt
```

**Expected output:**

```
Launched: i-0123456789abcdef0
```

### Step 2 (CloudShell): Wait until it's ready, then connect

```bash
INSTANCE_ID=$(cat ~/lab3-instance-id.txt)
aws ec2 wait instance-running --instance-ids $INSTANCE_ID && echo "Running"

aws ssm describe-instance-information \
  --filters "Key=InstanceIds,Values=$INSTANCE_ID" \
  --query 'InstanceInformationList[0].PingStatus' --output text
```

**Expected output:**

```
Running
Online
```

If you see `None`, wait 60 seconds and run the second command again — the SSM agent is still starting.

Now connect: **EC2 console → Instances → select `opskart-log-explainer` → Connect → Session Manager → Connect.**

🎥 Video: `[00:00]` launch and connect

---

### Step 3 (inside EC2): Set up Python and get the sample logs

Switch to bash, set the region, and install the AWS Python library (same as Chapter 2):

```bash
bash
cd ~
export AWS_REGION=us-east-1
sudo dnf install -y python3-pip -q
pip3 install --user -q boto3
```

Now create the three sample log files. Copy each block as-is — these are the same files that ship in the book's GitHub repo under `chapter-03/sample-logs/`, so what you paste here is what you can commit there.

```bash
mkdir -p ~/logs && cd ~/logs
```

**`crashloop.log`** — a Kubernetes pod stuck in CrashLoopBackOff:

```bash
cat > ~/logs/crashloop.log <<'EOF'
2026-09-20T03:14:02Z kubectl describe pod web-7d9f8c6b7-x2kqp -n default
Name:         web-7d9f8c6b7-x2kqp
Namespace:    default
Node:         ip-10-0-1-23.ec2.internal
Status:       Running
Containers:
  app:
    Image:          123456789012.dkr.ecr.us-east-1.amazonaws.com/web:1.4.2
    State:          Waiting
      Reason:       CrashLoopBackOff
    Last State:     Terminated
      Reason:       OOMKilled
      Exit Code:    137
      Started:      Sun, 20 Sep 2026 03:12:41 +0000
      Finished:     Sun, 20 Sep 2026 03:13:58 +0000
    Ready:          False
    Restart Count:  14
    Limits:
      memory:  256Mi
    Requests:
      memory:  128Mi
Events:
  Type     Reason     Age                From     Message
  ----     ------     ----               ----     -------
  Normal   Pulled     3m (x14 over 22m)  kubelet  Container image already present on machine
  Warning  BackOff    2m (x38 over 20m)  kubelet  Back-off restarting failed container app in pod web-7d9f8c6b7-x2kqp_default
  Warning  Unhealthy  1m (x9 over 15m)   kubelet  Liveness probe failed: Get "http://10.0.3.14:8080/healthz": dial tcp 10.0.3.14:8080: connect: connection refused
--- container logs (app) ---
2026-09-20T03:13:55Z [info] starting worker pool, size=8
2026-09-20T03:13:56Z [info] loading in-memory cache from snapshot (last known size: 340MB)
2026-09-20T03:13:57Z [warn] cache growing past 200MB, gc not keeping up
2026-09-20T03:13:58Z [fatal] runtime: out of memory: cannot allocate 41943040 bytes
EOF
```

**`nginx-502.log`** — an Nginx reverse proxy that can't reach its upstream:

```bash
cat > ~/logs/nginx-502.log <<'EOF'
2026-09-21 09:02:11 [error] 1842#1842: *5011 connect() failed (111: Connection refused) while connecting to upstream, client: 203.0.113.44, server: api.example.com, request: "GET /v1/orders/8841 HTTP/1.1", upstream: "http://10.0.4.12:8081/v1/orders/8841", host: "api.example.com"
2026-09-21 09:02:11 172.31.6.201 - - [21/Sep/2026:09:02:11 +0000] "GET /v1/orders/8841 HTTP/1.1" 502 559 "-" "PostmanRuntime/7.36.0"
2026-09-21 09:02:14 [error] 1842#1842: *5013 connect() failed (111: Connection refused) while connecting to upstream, client: 203.0.113.45, server: api.example.com, request: "POST /v1/checkout HTTP/1.1", upstream: "http://10.0.4.12:8081/v1/checkout", host: "api.example.com"
2026-09-21 09:02:14 172.31.6.201 - - [21/Sep/2026:09:02:14 +0000] "POST /v1/checkout HTTP/1.1" 502 559 "-" "curl/8.4.0"
2026-09-21 09:02:19 [warn] 1842#1842: *5015 upstream server temporarily disabled while connecting to upstream, client: 203.0.113.46, server: api.example.com, upstream: "http://10.0.4.12:8081/v1/orders/8842"
2026-09-21 09:02:19 [error] 1842#1842: *5015 no live upstreams while connecting to upstream, client: 203.0.113.46, server: api.example.com, request: "GET /v1/orders/8842 HTTP/1.1", upstream: "http://backend_pool/v1/orders/8842", host: "api.example.com"
2026-09-21 09:02:19 172.31.6.201 - - [21/Sep/2026:09:02:19 +0000] "GET /v1/orders/8842 HTTP/1.1" 502 559 "-" "PostmanRuntime/7.36.0"
2026-09-21 09:02:25 [error] 1842#1842: *5018 connect() failed (111: Connection refused) while connecting to upstream, client: 203.0.113.47, server: api.example.com, request: "GET /health HTTP/1.1", upstream: "http://10.0.4.12:8081/health", host: "api.example.com"
EOF
```

**`oom-kill.log`** — the Linux kernel OOM-killer terminating a process:

```bash
cat > ~/logs/oom-kill.log <<'EOF'
2026-09-22T14:07:40.112201+00:00 ip-10-0-2-88 kernel: [884213.552145] java invoked oom-killer: gfp_mask=0x140dca(GFP_HIGHUSER_MOVABLE|__GFP_COMP), order=0, oom_score_adj=0
2026-09-22T14:07:40.112390+00:00 ip-10-0-2-88 kernel: [884213.552310] CPU: 1 PID: 24831 Comm: java Not tainted 6.1.0-1027-aws
2026-09-22T14:07:40.112512+00:00 ip-10-0-2-88 kernel: [884213.552480] Mem-Info:
2026-09-22T14:07:40.112601+00:00 ip-10-0-2-88 kernel: [884213.552601] active_anon:891204 inactive_anon:12043 isolated_anon:0
2026-09-22T14:07:40.112701+00:00 ip-10-0-2-88 kernel: [884213.552699]  free:9812 free_pcp:203 free_cma:0
2026-09-22T14:07:40.113001+00:00 ip-10-0-2-88 kernel: [884213.553012] Node 0 Normal free:39248kB min:41984kB low:52480kB high:62976kB
2026-09-22T14:07:40.113501+00:00 ip-10-0-2-88 kernel: [884213.553501] Out of memory: Killed process 24801 (java) total-vm:4823124kB, anon-rss:3980112kB, file-rss:2048kB, shmem-rss:0kB, UID:1000 pgtables:8192kB oom_score_adj:0
2026-09-22T14:07:40.113620+00:00 ip-10-0-2-88 kernel: [884213.553620] oom_reaper: reaped process 24801 (java), now anon-rss:0kB, file-rss:0kB, shmem-rss:0kB
2026-09-22T14:07:41.002011+00:00 ip-10-0-2-88 systemd[1]: app.service: Main process exited, code=killed, status=9/KILL
2026-09-22T14:07:41.002305+00:00 ip-10-0-2-88 systemd[1]: app.service: Failed with result 'oom-kill'.
2026-09-22T14:07:41.002501+00:00 ip-10-0-2-88 systemd[1]: app.service: Scheduled restart job, restart counter is at 3.
EOF
```

Confirm all three landed:

```bash
ls -1 ~/logs
```

**Expected output:**

```
crashloop.log
nginx-502.log
oom-kill.log
```

**Push these to your GitHub repo now:** once your repo exists, copy the exact commands above (or the resulting files from `~/logs/`) into `chapter-03/sample-logs/` — that's what makes the repo's sample logs and the book's instructions the same file.

---

### Step 4 (inside EC2): Create the log explainer

Copy the whole block. As with Chapter 2, you don't need to read Python — the comments explain each part.

```bash
cat > ~/explain_log.py <<'EOF'
import sys, boto3

MODEL_ID = "us.amazon.nova-micro-v1:0"

if len(sys.argv) < 2:
    print("Usage: python3 explain_log.py <path-to-log-file>")
    sys.exit(1)

# Read the log file, and keep only the last 100 lines so we stay cheap and focused
with open(sys.argv[1]) as f:
    log_text = "".join(f.readlines()[-100:])

client = boto3.client("bedrock-runtime", region_name="us-east-1")   # no keys: uses the IAM role

SYSTEM = (
    "You are a senior DevOps/SRE engineer. You are given a raw error log. "
    "Respond in exactly three short sections with these headings: "
    "ROOT CAUSE (one or two sentences), "
    "FIX (numbered, concrete commands or config changes), "
    "PREVENT (one line on how to stop it recurring). "
    "Be specific and practical. Do not repeat the log back."
)

resp = client.converse(
    modelId=MODEL_ID,
    system=[{"text": SYSTEM}],
    messages=[{"role": "user", "content": [{"text": f"Here is the log:\n\n{log_text}"}]}],
    inferenceConfig={"maxTokens": 500, "temperature": 0.2},
)

print(resp["output"]["message"]["content"][0]["text"])

u = resp["usage"]
print(f"\n--- tokens in={u['inputTokens']} out={u['outputTokens']} "
      f"| latency={resp['metrics']['latencyMs']} ms")
EOF
```

Two things worth noticing in this script, because they come up in interviews:

- **We send only the last 100 lines.** Logs can be huge, and you pay per token. The tail almost always holds the actual failure. This is a simple, deliberate cost control.
- **`temperature=0.2`** keeps the answer focused and repeatable. For diagnosis you want consistency, not creativity.

---

### Step 5 (inside EC2): Explain the three logs

**5a. Kubernetes CrashLoopBackOff:**

```bash
python3 ~/explain_log.py ~/logs/crashloop.log
```

**Expected output (wording will vary):**

```
ROOT CAUSE
The container is being OOMKilled (exit code 137) — it exceeds its memory limit, so
Kubernetes restarts it repeatedly, producing the CrashLoopBackOff.

FIX
1. Check the current limits: kubectl describe pod web-7d9f -n default
2. Raise the memory limit in the deployment, e.g. resources.limits.memory: 512Mi
3. Re-apply: kubectl apply -f deployment.yaml
4. Watch it recover: kubectl get pods -w

PREVENT
Set memory requests/limits from real usage (kubectl top pod) and add an alert on OOMKilled events.

--- tokens in=48 out=121 | latency=680 ms
```

**5b. Nginx 502 Bad Gateway:**

```bash
python3 ~/explain_log.py ~/logs/nginx-502.log
```

You should get a root cause pointing at the upstream/backend being unreachable (wrong port, backend down, or a timeout), with fix steps to check the upstream service and Nginx's `proxy_pass` target.

**5c. Linux out-of-memory kill:**

```bash
python3 ~/explain_log.py ~/logs/oom-kill.log
```

You should get a root cause about the kernel OOM-killer terminating a process to free memory, with fixes around adding swap, raising memory, or capping the offending process.

**Try your own:** paste any error you're currently stuck on into a file and run the explainer on it.

```bash
cat > ~/logs/mine.log    # paste your log, then press Ctrl+D
python3 ~/explain_log.py ~/logs/mine.log
```

🎥 Video: `[00:00]` run all three logs

---

### Step 6 (CloudShell): Cleanup

Type `exit` twice to leave the Session Manager tab, then in **CloudShell**:

```bash
INSTANCE_ID=$(cat ~/lab3-instance-id.txt)
aws ec2 terminate-instances --instance-ids $INSTANCE_ID \
  --query 'TerminatingInstances[0].CurrentState.Name' --output text
aws ec2 wait instance-terminated --instance-ids $INSTANCE_ID && echo "Terminated — billing stopped"
```

**Expected output:**

```
shutting-down
Terminated — billing stopped
```

Keep the IAM role and instance profile — later chapters reuse them.

---

## Troubleshooting

| Issue | Cause | Fix |
|---|---|---|
| `Value (opskart-bedrock-profile) ... is invalid` at launch | The Chapter 2 profile was deleted | Re-run Chapter 2, Step 1 to recreate the role and instance profile |
| `Unable to locate credentials` inside EC2 | Instance launched without the profile | EC2 → Actions → Security → **Modify IAM role** → attach `opskart-bedrock-ec2-role` |
| `You must specify a region` | `AWS_REGION` not set in this shell | `export AWS_REGION=us-east-1` |
| `curl` returns nothing / files are empty | Repo path wrong, or no internet on the box | Use the inline `cat > ... <<'EOF'` method in the Step 3 note |
| `FileNotFoundError` from the script | Wrong path to the log file | Check `ls ~/logs`, and pass the full path |
| `AccessDeniedException ... bedrock:InvokeModel` | Role missing the Bedrock policy | Re-run Chapter 2, Step 1b (the `put-role-policy` command) |
| `PythonDeprecationWarning: Boto3 will no longer support Python 3.9` | AL2023 ships Python 3.9 | Harmless — the answer prints below the warning. Ignore it. |
| Answer ignores the headings format | Model drifted | Re-run — with `temperature=0.2` it's usually consistent; you can lower it to `0.1` |

---

## Interview questions

**1. Why send only the tail of a log to the model instead of the whole file?**
Cost and signal. You pay per input token, and a large log wastes tokens on lines that aren't relevant. The failure and its immediate context are almost always at the end, so the last N lines give the model what it needs at a fraction of the cost. In production you'd go further — filter to error/warning lines, or the lines around the first exception — before sending.

**2. Would you send production logs to an external AI model? What's the risk?**
Logs can contain secrets, PII, internal hostnames and IPs. Before sending anything externally you'd redact or mask sensitive fields, and confirm the data-handling terms of the service. Amazon Bedrock keeps requests within your AWS account boundary and doesn't use them to train the base models, which is a large part of why this lab uses Bedrock rather than a public chatbot. This is exactly what Chapter 5 (Guardrails) addresses.

**3. How would you turn this one-off script into something a team uses?**
Wrap it so it runs automatically: trigger it from a log alert (CloudWatch alarm → Lambda), feed it the relevant log slice, and post the explanation to Slack or attach it to the incident ticket. Keep a human in the loop for the fix. That's the path from "handy script" to "AIOps" — and it's essentially what the later chapters build toward.

---

## Chapter 3 checklist

- [ ] Fresh EC2 launched with the Chapter 2 instance profile
- [ ] Connected with Session Manager
- [ ] `boto3` installed and the three sample logs downloaded (or created inline)
- [ ] `explain_log.py` created
- [ ] All three logs produced a sensible root cause + fix
- [ ] Ran the explainer on one of your own logs
- [ ] EC2 terminated, role kept
- [ ] `explain_log.py` pushed to your GitHub repo

**Next → Chapter 4: Serverless AI API** — the same Bedrock call, but behind a Lambda function with a public URL, so there's no server to keep running at all.
