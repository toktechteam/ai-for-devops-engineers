# Chapter 4: Serverless AI API (Lambda + Bedrock + Function URL)

💸 **Cost:** ~$0 (Lambda free tier: 1M requests + 400,000 GB-seconds/month; Bedrock Nova Micro calls cost fractions of a cent each)
⏱ **Time:** 25–30 minutes
🧹 **Cleanup:** delete the Lambda function and its execution role at the end of this chapter

**The purpose of this chapter, in one sentence:** turn the AI call you built in Chapter 2 — a script that only works if you log into a specific server — into an HTTPS API that anything on the internet can call, with no server for you to run at all.

**By the end of this chapter, you will have:**
- A Lambda function that calls Bedrock and returns an AI answer
- A public HTTPS endpoint for that function (a Lambda Function URL) — no API Gateway needed
- An IAM-authenticated endpoint, called with a signed request (not a wide-open public URL)
- A working pattern: HTTP request in → AI answer out, with no server to manage or patch

🎥 [VIDEO ch04 | QR code + short link | ~8 min | unlisted]

## Why this matters

**The problem with what you built in Chapter 2 and Chapter 3:** both work by SSHing (or Session Manager-ing) into an EC2 instance and running a script by hand. That's fine for learning, but it isn't something a real product can use. A website's "Ask AI" button, a Slack bot, a form that auto-summarizes a support ticket — none of them can log into your server and run a Python file. What they *can* do is send an HTTPS request to a URL and get a response back. This chapter closes that gap: the same Bedrock call from Chapter 2, reachable as a normal web API.

**What you'll actually learn here**, beyond "it works": how to package Python code for Lambda and deploy it from the CLI, how a Lambda Function URL exposes a function over HTTPS without standing up API Gateway, how to lock that URL down with IAM so it isn't a public bill-generator, and how to sign a request with temporary AWS credentials the way a real client application would. These are the same mechanics behind almost any "serverless API" you'll meet in a DevOps role, not just AI ones.

**Why Lambda, specifically, and not just "a smaller EC2":** it's not about EC2 being bad — it's about matching the tool to the shape of the work. Chapter 2's EC2 instance runs continuously whether or not anyone is asking it a question: you pay for every hour it's up, and you're on the hook for patching its OS. An AI API, in almost every real product, is *bursty* — it sits at zero most of the time and gets called occasionally. Lambda is built exactly for that shape: AWS runs your code only when a request arrives, scales it automatically if ten requests arrive at once, and charges only for the milliseconds it actually executes. You're not learning Lambda because it's trendy — you're learning it because "run a short task in response to an event, then stop" is one of the most common patterns in cloud and DevOps work, and an AI call is a perfect example of that event.

This chapter wires Lambda directly to Bedrock and exposes it as an HTTPS endpoint using a **Lambda Function URL** — a built-in way to call a Lambda function over HTTP without setting up API Gateway. This is the same shape used by real production AI features: a chatbot backend, a Slack bot, a webhook that summarizes an incoming ticket. You're building the smallest version of that pattern.

## Concept

**EC2 (Chapter 2) vs Lambda (this chapter):**

| | EC2 | Lambda |
|---|---|---|
| Runs | Continuously, until you stop it | Only while handling a request |
| You manage | OS patches, security groups, instance lifecycle | Nothing — AWS runs the runtime |
| Billing | Per second the instance is running | Per request + per millisecond of execution |
| Good for | Long-running processes, full control | Short tasks triggered by events |

**How a Lambda Function URL request flows:**

```
Client (curl / browser / app)
   │  HTTPS request, signed with AWS credentials
   ▼
Lambda Function URL  (auth_type = AWS_IAM)
   │  AWS verifies the signature before your code ever runs
   ▼
Lambda function (Python)
   │  calls bedrock-runtime.converse()
   ▼
Amazon Bedrock (Nova Micro)
   │  returns the AI's answer
   ▼
Lambda returns it as the HTTP response
```

**A note on authentication.** A Function URL can be created with `auth_type=NONE`, which makes it a fully public endpoint — anyone on the internet who has the URL can invoke it and run up your Bedrock bill. AWS's own default resource policy for `NONE` grants invoke access to `Principal: "*"` (everyone). This book uses `auth_type=AWS_IAM` instead: only callers who sign their request with valid AWS credentials — and whose IAM identity has been granted `lambda:InvokeFunctionUrl` — can call it. That's a five-minute extra step and it's the difference between "a demo you control" and "an open bill waiting to happen." If you later put a real client (a website, a mobile app) in front of this, you'd normally add API Gateway or CloudFront with its own auth layer — but for learning the Lambda-to-Bedrock pattern, `AWS_IAM` is the right default.

**A note on signed requests.** CloudShell (and most AWS environments) hand you **temporary** credentials — an access key, a secret key, and a session token that expires. All three travel together, and a signing tool that leaves out the session token produces a request AWS will reject with `"The security token included in the request is invalid"`. CloudShell also doesn't expose these as plain `AWS_ACCESS_KEY_ID`-style environment variables you can read directly — it manages them behind the scenes. Step 5 below uses `aws configure export-credentials`, a built-in AWS CLI command that resolves whatever credentials are currently active (CloudShell, an IAM role, SSO — anything) into real values you can hand to `curl`.

**A note on cross-region inference profiles.** The `us.` prefix on `us.amazon.nova-micro-v1:0` marks it as a *cross-region inference profile* — AWS can route your request to whichever US region (`us-east-1`, `us-east-2`, or `us-west-2`) has capacity at that moment, not necessarily the region you called from. Because of this, your IAM policy has to allow `bedrock:InvokeModel` on the foundation-model resource **in any region**, not just `us-east-1` — otherwise a request that happens to route to, say, `us-west-2` gets an `AccessDeniedException` even though everything else is set up correctly. Step 1 below uses a wildcard region (`arn:aws:bedrock:*::foundation-model/...`) for exactly this reason.

## Hands-on

### Step 1: Create the Lambda execution role

Open CloudShell and create a trust policy that lets Lambda assume a role:

```bash
cat > ~/lambda-trust-policy.json <<'EOF'
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": { "Service": "lambda.amazonaws.com" },
      "Action": "sts:AssumeRole"
    }
  ]
}
EOF

aws iam create-role \
  --role-name opskart-lambda-bedrock-role \
  --assume-role-policy-document file://~/lambda-trust-policy.json
```

Attach the AWS managed policy that lets Lambda write its own logs to CloudWatch:

```bash
aws iam attach-role-policy \
  --role-name opskart-lambda-bedrock-role \
  --policy-arn arn:aws:iam::aws:policy/service-role/AWSLambdaBasicExecutionRole
```

Add an inline policy that allows only the one Bedrock action this function needs, scoped to Nova Micro. Note the foundation-model resource uses a **wildcard region** (`bedrock:*::`) — the `us.` inference profile can route your call to any US region, so the policy has to allow the model in all of them, not just `us-east-1` (see the "cross-region inference profiles" note above):

```bash
cat > ~/lambda-bedrock-policy.json <<'EOF'
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": "bedrock:InvokeModel",
      "Resource": [
        "arn:aws:bedrock:*::foundation-model/amazon.nova-micro-v1:0",
        "arn:aws:bedrock:us-east-1:*:inference-profile/us.amazon.nova-micro-v1:0"
      ]
    }
  ]
}
EOF

aws iam put-role-policy \
  --role-name opskart-lambda-bedrock-role \
  --policy-name opskart-invoke-nova-micro \
  --policy-document file://~/lambda-bedrock-policy.json
```

**Expected output:** each command returns a JSON block confirming the role or policy was created — no errors.

Wait about 10 seconds for IAM to finish propagating the new role before the next step:

```bash
sleep 10
```

### Step 2: Write the Lambda function code

```bash
mkdir -p ~/lambda-ai-api && cd ~/lambda-ai-api

cat > lambda_function.py <<'EOF'
import json
import boto3

bedrock = boto3.client("bedrock-runtime", region_name="us-east-1")
MODEL_ID = "us.amazon.nova-micro-v1:0"

def lambda_handler(event, context):
    # Function URL puts the raw HTTP body here as a string
    body_raw = event.get("body") or "{}"
    try:
        body = json.loads(body_raw)
    except json.JSONDecodeError:
        body = {}

    question = body.get("question", "Say hello in one short sentence.")

    response = bedrock.converse(
        modelId=MODEL_ID,
        messages=[{"role": "user", "content": [{"text": question}]}],
        inferenceConfig={"temperature": 0.3, "maxTokens": 300},
    )

    answer = response["output"]["message"]["content"][0]["text"]
    usage = response["usage"]

    return {
        "statusCode": 200,
        "headers": {"Content-Type": "application/json"},
        "body": json.dumps(
            {
                "question": question,
                "answer": answer,
                "input_tokens": usage["inputTokens"],
                "output_tokens": usage["outputTokens"],
            }
        ),
    }
EOF
```

This function reads a JSON body with a `question` field, sends it to Nova Micro, and returns the answer plus token counts as JSON — the same shape as a real API.

### Step 3: Package and deploy the function

```bash
zip lambda-package.zip lambda_function.py

ROLE_ARN=$(aws iam get-role --role-name opskart-lambda-bedrock-role --query 'Role.Arn' --output text)
echo "Role ARN: $ROLE_ARN"

aws lambda create-function \
  --function-name opskart-ai-api \
  --runtime python3.13 \
  --role "$ROLE_ARN" \
  --handler lambda_function.lambda_handler \
  --zip-file fileb://lambda-package.zip \
  --timeout 30 \
  --memory-size 256
```

**Expected output:** a JSON block with `"FunctionName": "opskart-ai-api"` and `"State": "Pending"` (it becomes `Active` within a few seconds).

> ⚠️ **boto3 version note:** Lambda's built-in Python 3.13 runtime ships its own bundled version of boto3/botocore. If you see `Unknown service: 'bedrock-runtime'` or `Parameter validation failed` errors that don't match this chapter's expected output, the bundled SDK on your function is older than what `converse()` needs. Fix: package a newer boto3 alongside your code —
> ```bash
> pip3 install boto3 --target ~/lambda-ai-api/package --quiet
> cd ~/lambda-ai-api/package && zip -r ../lambda-package.zip . && cd ..
> zip lambda-package.zip lambda_function.py
> aws lambda update-function-code --function-name opskart-ai-api --zip-file fileb://lambda-package.zip
> ```
> Try the chapter as written first — many accounts already get a recent enough runtime. Only do this if you hit the error above.

### Step 4: Create the Function URL with IAM auth

```bash
aws lambda create-function-url-config \
  --function-name opskart-ai-api \
  --auth-type AWS_IAM
```

**Expected output:** a JSON block containing a `FunctionUrl` like `https://abcd1234xyz.lambda-url.us-east-1.on.aws/`. Save that URL — you'll use it below.

Grant your own IAM identity permission to invoke it. Get your identity first, then add the permission:

```bash
CALLER_ARN=$(aws sts get-caller-identity --query 'Arn' --output text)
echo "Caller: $CALLER_ARN"

aws lambda add-permission \
  --function-name opskart-ai-api \
  --statement-id allow-my-user-invoke \
  --action lambda:InvokeFunctionUrl \
  --principal "$CALLER_ARN" \
  --function-url-auth-type AWS_IAM
```

**Expected output:** a JSON block with a `"Statement"` field confirming the permission was added.

### Step 5: Call the endpoint with a signed request

```bash
FUNCTION_URL=$(aws lambda get-function-url-config --function-name opskart-ai-api --query 'FunctionUrl' --output text)
echo "Calling: $FUNCTION_URL"

(
  eval "$(aws configure export-credentials --format env)"
  curl -sS "$FUNCTION_URL" \
    --aws-sigv4 "aws:amz:us-east-1:lambda" \
    --user "${AWS_ACCESS_KEY_ID}:${AWS_SECRET_ACCESS_KEY}" \
    -H "X-Amz-Security-Token: ${AWS_SESSION_TOKEN}" \
    -H "Content-Type: application/json" \
    -d '{"question": "In one sentence, what is the difference between EC2 and Lambda?"}'
)
```

`aws configure export-credentials --format env` asks the CLI to resolve whatever credentials it's currently using — CloudShell's managed credentials, an IAM role, SSO, a static key, anything — and print them as `export AWS_ACCESS_KEY_ID=...` / `AWS_SECRET_ACCESS_KEY=...` / `AWS_SESSION_TOKEN=...` statements. `eval` loads those so `curl` can sign the request.

**Why the brackets `( ... )` matter:** they run everything inside a *subshell*, so the exported credentials disappear when the command finishes. Without them, those variables stay in your shell as a frozen snapshot. The AWS CLI prefers environment variables over CloudShell's auto-refreshing credentials, so once the snapshot expires (within the hour), **every** `aws` command in that tab fails with `Credentials were refreshed, but the refreshed credentials are still expired`. Keep the brackets.

> If you're on a long-term IAM user (static access keys, no session token), `export-credentials` will simply omit `AWS_SESSION_TOKEN` — the `-H "X-Amz-Security-Token: ${AWS_SESSION_TOKEN}"` line then sends an empty header, which AWS ignores safely.

**Expected output:**

```json
{"question": "In one sentence, what is the difference between EC2 and Lambda?", "answer": "EC2 gives you a virtual server that runs continuously and that you manage yourself, while Lambda runs your code only when triggered and requires no server management.", "input_tokens": 17, "output_tokens": 34}
```

Try it with your own question by changing the `-d` payload.

### Step 6: Confirm it's actually locked down

Test that an unsigned request is rejected — this proves `AWS_IAM` is doing its job:

```bash
curl -sS "$FUNCTION_URL" -d '{"question": "test"}'
```

**Expected output:**

```json
{"Message":"Forbidden"}
```

If you see the AI's answer instead of `Forbidden` here, your Function URL is not actually IAM-protected — double-check Step 4.

### Step 7: Cleanup

```bash
aws lambda delete-function-url-config --function-name opskart-ai-api
aws lambda delete-function --function-name opskart-ai-api
aws iam delete-role-policy --role-name opskart-lambda-bedrock-role --policy-name opskart-invoke-nova-micro
aws iam detach-role-policy --role-name opskart-lambda-bedrock-role --policy-arn arn:aws:iam::aws:policy/service-role/AWSLambdaBasicExecutionRole
aws iam delete-role --role-name opskart-lambda-bedrock-role
```

**Expected output:** no errors; each command returns nothing on success.

## Troubleshooting

| Problem | Fix |
|---|---|
| `{"message":"The security token included in the request is invalid."}` when calling the Function URL | You signed with a stale, empty, or missing session token. Run `eval "$(aws configure export-credentials --format env)"` right before the `curl` call, as shown in Step 5, so the token is current |
| Later, every `aws` command fails with `Credentials were refreshed, but the refreshed credentials are still expired` | You ran the `eval` line outside the `( ... )` brackets, so expired copies of the credentials are stuck in your shell. Run `unset AWS_ACCESS_KEY_ID AWS_SECRET_ACCESS_KEY AWS_SESSION_TOKEN AWS_CREDENTIAL_EXPIRATION`, or open a new CloudShell tab |
| `Forbidden` when calling with `--aws-sigv4` | Check that `add-permission` in Step 4 used the exact ARN from `sts get-caller-identity`; re-run `get-caller-identity` and compare |
| `Unknown service: 'bedrock-runtime'` or validation errors from `converse()` | Bundled boto3 on the Lambda runtime is too old — package a newer boto3 as shown in the Step 3 callout |
| `AccessDeniedException` from Bedrock inside the Lambda logs, naming a resource ARN in a region you didn't expect (e.g. `us-west-2` when you're working in `us-east-1`) | The `us.` inference profile routed your request to a different US region. Make sure Step 1's foundation-model `Resource` entry uses a wildcard region — `arn:aws:bedrock:*::foundation-model/amazon.nova-micro-v1:0` — not a single region |
| Function returns `{"errorMessage": ...}` about `event['body']` | You sent a GET request or empty body — Function URLs need a POST with a JSON body via `-d` |
| `create-function` fails with `InvalidParameterValueException` about the role | IAM hasn't finished propagating the new role yet — re-run `sleep 10` and retry |
| Response takes 10+ seconds on the first call | This is a "cold start" — Lambda initializing a fresh execution environment. Later calls are faster while the environment stays warm |

To view the function's logs (useful for debugging):

```bash
aws logs tail /aws/lambda/opskart-ai-api --since 10m
```

## Interview questions

1. What's the difference between a Lambda Function URL with `auth_type=NONE` and `auth_type=AWS_IAM`, and what's the security risk of `NONE`?
2. Why does signing a request with temporary AWS credentials require a session token in addition to the access key and secret key, and why can't you always read that token from a plain environment variable?
3. When would you choose Lambda over a long-running EC2 instance for an AI workload, and when would EC2 still be the better choice?
4. What is a cross-region inference profile, and why does using one mean your IAM policy can't be scoped to a single region?

## Checklist

- [ ] Created `opskart-lambda-bedrock-role` scoped to only invoke Nova Micro
- [ ] Deployed `opskart-ai-api` Lambda function
- [ ] Created a Function URL with `auth_type=AWS_IAM`
- [ ] Called it successfully with a signed request
- [ ] Confirmed an unsigned request is rejected with `Forbidden`
- [ ] Deleted the Function URL, the function, and the role
