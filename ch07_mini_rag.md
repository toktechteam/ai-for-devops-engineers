# Chapter 7: Mini-RAG (Answer Questions From a Runbook, No Vector Database)

💸 **Cost:** a few cents at most (a handful of short Nova Micro calls; the retrieval step itself runs locally and costs nothing)
⏱ **Time:** 30–35 minutes
🧹 **Cleanup:** none — this chapter creates no AWS resources, only local files and Bedrock calls

**The purpose of this chapter, in one sentence:** teach the model to answer questions only from your own document — a runbook — instead of guessing, and do it with plain Python instead of standing up a vector database you don't need yet.

**Why anyone should spend 30–35 minutes on this:** "RAG" (retrieval-augmented generation) is one of the most-asked-about AI patterns in interviews right now, and most explanations jump straight to a vector database, which is intimidating if you've never used one. This chapter gives you the real idea — find the relevant piece of your data first, then let the model answer only from that piece — in code short enough to read top to bottom in a few minutes, before you ever need a managed search service.

**By the end of this chapter, you will have:**
- Split a sample DevOps runbook into sections your code can search
- Built a retrieval step in plain Python — no vector database, no embeddings API, no extra AWS service — that finds the most relevant section for a question
- Wired that retrieved section into a Bedrock call so the model answers **only from the runbook**, instead of guessing
- Proven the model says "not in this runbook" for a question the runbook doesn't cover, instead of making something up

🎥 [VIDEO ch07 | QR code + short link | ~8 min | unlisted]

## Why this matters

By Chapter 3 you already sent a single log file to a model and got an explanation back. A runbook is different: it's long, it has many unrelated sections, and a real question usually only needs one or two of them. If you paste the whole runbook into every prompt, you pay for tokens the model doesn't need, and once the runbook grows past a few pages you'll hit the model's context limit anyway.

This is the same problem that "RAG" (retrieval-augmented generation) solves in production systems — find the relevant piece of your own data first, then hand only that piece to the model. Most tutorials jump straight to a vector database (OpenSearch, Pinecone, a Bedrock Knowledge Base) to do the "find the relevant piece" step. That's the right call once you have thousands of documents. For one runbook with a dozen sections, it's expensive and overkill — you'd be running a managed search cluster to search twelve paragraphs.

This chapter builds the same pattern at the smallest scale that still teaches it honestly: a scoring method you can read line by line, no new AWS service, no new bill.

## Concept

**DevOps analogy:** this is `grep -l` with judgment. Plain `grep` finds lines containing exact words; this chapter's retrieval step scores every section by how well its words overlap with the question's words, so "pod won't start" can still match a section titled "CrashLoopBackOff" if enough of the surrounding words line up — closer to how a search engine ranks pages than to an exact string match.

**The method: TF-IDF, by hand.** TF-IDF stands for **term frequency – inverse document frequency**. It scores a word in a section two ways:
- **Term frequency (TF):** how often the word appears in *this* section — more occurrences, higher weight.
- **Inverse document frequency (IDF):** how rare the word is *across all sections* — a word in every section (like "the," or in this runbook, "check") tells you nothing about which section matches, so it gets a low weight; a word that appears in only one section (like "CrashLoopBackOff") is a strong signal, so it gets a high weight.

Multiply the two together per word, sum over all the words a question and a section share, and you get a similarity score. The highest-scoring section is what you hand to the model.

**Why this is "RAG" and not just search:** the retrieved text doesn't just get displayed to a human — it gets inserted into the prompt as the model's only source of truth, with instructions to answer from it and nothing else. That's the "augmented generation" half. The two steps are:

```
question
   │
   ▼
score every runbook section against the question (local, pure Python)
   │
   ▼
take the top-scoring section(s)
   │
   ▼
prompt = "Answer only using this runbook section: <section> \n Question: <question>"
   │
   ▼
Converse (Nova Micro) → answer grounded in the runbook, or "not covered"
```


**When to graduate to a real vector database.** This hand-rolled scoring works well for a handful of documents with clearly distinct vocabulary, like one team's runbook. It gets worse as the document count grows into the hundreds, or when questions use completely different words than the document (a true vector/embedding search handles that better, because it matches on meaning, not shared words). Chapter 8's capstone is a good point to decide whether a real knowledge base is worth it for a specific project — this chapter is the lab that makes that trade-off a choice you understand rather than a default you reached for.

## Hands-on

All steps run in **CloudShell** (region `us-east-1`), signed in as the `opskart-admin` user from Chapter 0.

### Step 1: Create the sample runbook

A runbook with five independent incident sections — different enough in vocabulary that the scoring in Step 3 has something real to tell apart:

```bash
mkdir -p ~/ch07 && cd ~/ch07

cat > ~/ch07/runbook.md <<'EOF'
# Platform Runbook

## Pod CrashLoopBackOff
Symptom: `kubectl get pods` shows a pod cycling through CrashLoopBackOff. The pod starts, exits with a non-zero code, and Kubernetes restarts it on a backoff timer.
Diagnosis: run `kubectl logs <pod> --previous` to see the crash from the last attempt, not the current restart. Most causes are a missing environment variable, a failed health check before the app is ready, or an out-of-memory kill (check `kubectl describe pod <pod>` for `OOMKilled`).
Fix: correct the missing config or raise the memory limit in the deployment spec, then `kubectl rollout restart deployment/<name>`.
Escalate to: platform on-call if the same pod crash-loops after two fix attempts.

## Database Connection Pool Exhausted
Symptom: the application logs show `HikariPool-1 - Connection is not available, request timed out` or similar pool-timeout errors under load.
Diagnosis: check the current pool size against `max_connections` on the database (`SHOW max_connections;` on Postgres). A connection leak (connections opened but never closed) or a traffic spike past the pool's configured size are the two common causes.
Fix: for a leak, redeploy the last change that touched database code and roll back if the leak started there. For a capacity issue, raise the application's pool size and the database's `max_connections` together, never just one.
Escalate to: database on-call if `max_connections` is already near the database instance's hard limit.

## Disk Space Alert
Symptom: a CloudWatch alarm fires for disk utilization above 85% on an EC2 instance or an EBS-backed volume.
Diagnosis: run `df -h` to find the full mount, then `du -sh /var/log/* 2>/dev/null | sort -rh | head` to find what's consuming it. Application logs that were never rotated are the most common cause.
Fix: compress and archive old logs to S3, confirm `logrotate` is configured for the application's log directory, and resize the EBS volume if genuine data growth (not logs) is the cause.
Escalate to: nobody — this is a routine fix unless the disk fills again within 24 hours of cleanup, which means the root cause wasn't found.

## SSL Certificate Expiry
Symptom: clients report `NET::ERR_CERT_DATE_INVALID` or similar, or a monitoring check for certificate expiry fires.
Diagnosis: run `echo | openssl s_client -connect <host>:443 2>/dev/null | openssl x509 -noout -dates` to see the exact expiry date. Check whether auto-renewal (ACM, Let's Encrypt, or similar) is configured and whether it actually ran.
Fix: if using ACM behind an ALB, confirm the certificate is attached to the listener, not just issued. If using a manual certificate, renew and redeploy it immediately; this is always urgent once a cert has expired.
Escalate to: security on-call if the certificate was supposed to auto-renew and did not — that's a process failure worth a follow-up, not just a one-time fix.

## High API Latency
Symptom: p99 latency on an API endpoint climbs well above its normal baseline, with no corresponding traffic spike.
Diagnosis: check whether a downstream dependency (database, another internal service, a third-party API) is slow first — most latency incidents are not the service itself. Compare current CPU and memory against baseline; a resource-starved instance is the second most common cause.
Fix: if a downstream dependency is slow, that team's runbook applies instead. If the service itself is resource-starved, scale out (more instances) before scaling up (bigger instances), since it's faster and reversible.
Escalate to: the owning team's on-call for the slow dependency, if one is identified.
EOF

grep -c "^## " ~/ch07/runbook.md
```

**Expected output:** `5` — one line per section header.


### Step 2: Install boto3 (if needed)

```bash
pip3 install --user --quiet boto3
python3 -c "import boto3; print('boto3', boto3.__version__)"
```

**Expected output:** `boto3 1.x.x`. Everything else in this chapter uses only Python's standard library (`re`, `math`, `collections`) — no new packages.

### Step 3: Build the retriever and test it alone

Test retrieval by itself first, with no model call, so you can see exactly what gets picked before it's wired into anything:

```bash
cat > ~/ch07/retrieve.py <<'EOF'
import re
import math
import sys
from collections import Counter

STOPWORDS = {
    "a", "an", "and", "are", "as", "at", "be", "by", "for", "from", "has",
    "if", "in", "into", "is", "it", "its", "no", "not", "of", "on", "or",
    "that", "the", "this", "to", "was", "were", "will", "with", "you",
    "your", "what", "why", "how", "do", "does", "my",
}

def tokenize(text):
    words = re.findall(r"[a-z0-9']+", text.lower())
    return [w for w in words if w not in STOPWORDS]

def chunk_runbook(path):
    """Split on '## ' headers into (title, body) pairs."""
    chunks = []
    title, body_lines = None, []
    with open(path, encoding="utf-8") as f:
        for line in f:
            if line.startswith("## "):
                if title:
                    chunks.append((title, "".join(body_lines).strip()))
                title = line[3:].strip()
                body_lines = []
            elif not line.startswith("# "):
                body_lines.append(line)
    if title:
        chunks.append((title, "".join(body_lines).strip()))
    return chunks

def build_index(chunks):
    """Return (tf_idf_vectors, idf) for a list of (title, body) chunks."""
    doc_tokens = [tokenize(title + " " + body) for title, body in chunks]
    doc_freq = Counter()
    for tokens in doc_tokens:
        for term in set(tokens):
            doc_freq[term] += 1

    n_docs = len(doc_tokens)
    idf = {term: math.log((n_docs + 1) / (freq + 1)) + 1 for term, freq in doc_freq.items()}

    vectors = []
    for tokens in doc_tokens:
        tf = Counter(tokens)
        vec = {term: count * idf[term] for term, count in tf.items()}
        norm = math.sqrt(sum(v * v for v in vec.values())) or 1.0
        vectors.append({term: v / norm for term, v in vec.items()})
    return vectors, idf

def score_question(question, vectors, idf):
    """Cosine similarity between the question and every chunk vector."""
    tf = Counter(tokenize(question))
    q_vec = {term: count * idf.get(term, 0.0) for term, count in tf.items()}
    q_norm = math.sqrt(sum(v * v for v in q_vec.values())) or 1.0
    q_vec = {term: v / q_norm for term, v in q_vec.items()}

    scores = []
    for vec in vectors:
        dot = sum(q_vec.get(t, 0.0) * vec.get(t, 0.0) for t in q_vec)
        scores.append(dot)
    return scores

def top_match(question, chunks, vectors, idf):
    scores = score_question(question, vectors, idf)
    best_i = max(range(len(scores)), key=lambda i: scores[i])
    return chunks[best_i][0], chunks[best_i][1], scores[best_i], scores

if __name__ == "__main__":
    if len(sys.argv) != 2:
        sys.exit("Usage: python3 retrieve.py '<question>'")

    chunks = chunk_runbook("runbook.md")
    vectors, idf = build_index(chunks)
    title, body, best_score, all_scores = top_match(sys.argv[1], chunks, vectors, idf)

    print("=== SCORES (all sections) ===")
    for (t, _), s in sorted(zip(chunks, all_scores), key=lambda x: -x[1]):
        print(f"  {s:.3f}  {t}")

    print(f"\n=== BEST MATCH: {title} (score {best_score:.3f}) ===")
    print(body)
EOF
```

Test it against a question phrased completely differently from the section title, to see the scoring actually generalize, not just string-match:

```bash
python3 retrieve.py "my kubernetes pod keeps restarting and dying"
```

**Expected output** (scores will differ slightly, but the top match and its ordering should match):

```
=== SCORES (all sections) ===
  0.XXX  Pod CrashLoopBackOff
  0.0XX  Disk Space Alert
  0.0XX  Database Connection Pool Exhausted
  0.0XX  High API Latency
  0.0XX  SSL Certificate Expiry

=== BEST MATCH: Pod CrashLoopBackOff (score 0.XXX) ===
Symptom: `kubectl get pods` shows a pod cycling through CrashLoopBackOff. The pod starts, exits with a non-zero code, and Kubernetes restarts it on a backoff timer.
...
```

> 🖼️ [FIGURE ch07-fig01-scoreboard.png | full width | scores for all five sections on this exact question, winning section highlighted in orange — swap the illustrative numbers for the real ones once you've run this yourself]

Try one more, about certificates:

```bash
python3 retrieve.py "clients are seeing a certificate error in the browser"
```

**Expected output:** top match is `SSL Certificate Expiry`, with a visibly higher score than the other four sections.

### Step 4: Wire retrieval into a Bedrock call

This is the full mini-RAG script: retrieve the best section locally, then ask Nova Micro to answer **only** from it.

```bash
cat > ~/ch07/mini_rag.py <<'EOF'
import sys
import boto3
from retrieve import chunk_runbook, build_index, top_match

REGION = "us-east-1"
MODEL_ID = "us.amazon.nova-micro-v1:0"
RUNBOOK_PATH = "runbook.md"
MIN_SCORE = 0.05  # below this, treat it as "not found" rather than guessing

SYSTEM_PROMPT = (
    "You are a runbook assistant. You will be given one runbook section and a question. "
    "Answer using ONLY the information in that section. "
    "If the section does not actually answer the question, say exactly: "
    "\"This isn't covered in the runbook section I found — escalate or check another source.\" "
    "Do not use any knowledge from outside the provided section. Keep the answer to 3-5 sentences."
)

client = boto3.client("bedrock-runtime", region_name=REGION)

def answer(question):
    chunks = chunk_runbook(RUNBOOK_PATH)
    vectors, idf = build_index(chunks)
    title, body, score, _ = top_match(question, chunks, vectors, idf)

    print(f"=== RETRIEVED SECTION: {title} (score {score:.3f}) ===")

    if score < MIN_SCORE:
        print("Score too low — skipping the model call, nothing in the runbook is a good match.")
        return "This isn't covered in the runbook — escalate or check another source."

    prompt = f"Runbook section — {title}:\n{body}\n\nQuestion: {question}"

    response = client.converse(
        modelId=MODEL_ID,
        system=[{"text": SYSTEM_PROMPT}],
        messages=[{"role": "user", "content": [{"text": prompt}]}],
        inferenceConfig={"temperature": 0.1, "maxTokens": 300},
    )

    text = response["output"]["message"]["content"][0]["text"]
    usage = response["usage"]
    print(f"[tokens] input={usage['inputTokens']} output={usage['outputTokens']}")
    return text

if __name__ == "__main__":
    if len(sys.argv) != 2:
        sys.exit("Usage: python3 mini_rag.py '<question>'")
    print("\n=== ANSWER ===")
    print(answer(sys.argv[1]))
EOF

cd ~/ch07
python3 mini_rag.py "why does my database keep timing out under load"
```

**Expected output** (wording will vary slightly on each run):

```
=== RETRIEVED SECTION: Database Connection Pool Exhausted (score 0.3XX) ===
[tokens] input=2XX output=XX

=== ANSWER ===
This is likely a connection pool exhaustion issue. Check your current pool size against the database's max_connections setting -- common causes are a connection leak (connections opened but never closed) or a traffic spike exceeding the configured pool size. For a leak, check and possibly roll back the last change that touched database code. For a capacity issue, raise both the application's pool size and the database's max_connections together, not just one of them. Escalate to the database on-call if max_connections is already near the instance's hard limit.
```

Notice the answer reflects only the retrieved section — no mention of pods, certificates or disk space, because the model never saw those sections.

### Step 5: Test the "not in the runbook" case

A question the runbook genuinely doesn't cover should come back honest instead of invented:

```bash
python3 mini_rag.py "how do I rotate the company's annual encryption key policy"
```

**Expected output:** the retrieved score will be low (none of the five sections are really about this), and the answer should be close to:

```
=== RETRIEVED SECTION: SSL Certificate Expiry (score 0.0XX) ===
...
=== ANSWER ===
This isn't covered in the runbook section I found — escalate or check another source.
```

If the score comes back above `MIN_SCORE` (0.05) because of incidental word overlap (both mention "certificate" or "key"), the model itself should still decline to answer from an unrelated section, because the system prompt tells it to. This is why the chapter has two safety nets — the score threshold and the prompt instruction — rather than relying on either alone.

## Troubleshooting

| Problem | Fix |
|---|---|
| `top_match` always returns the first section, regardless of the question | `runbook.md` wasn't split correctly — check that every section header is exactly `## ` (two hashes, one space), matching `chunk_runbook`'s check |
| Every score comes back as `0.000` | The question has no words in common with any section after stopword removal — try a less generic phrasing, or check `STOPWORDS` isn't accidentally stripping a word that matters |
| `ModuleNotFoundError: No module named 'retrieve'` when running `mini_rag.py` | Run it from `~/ch07` (`cd ~/ch07` first) — `retrieve.py` must be in the same directory, or on `PYTHONPATH` |
| The model answers using outside knowledge instead of the runbook | Check the exact system prompt was used — re-create `mini_rag.py` with the heredoc above rather than editing it by hand, since a small wording change can loosen the "only from this section" instruction |
| Scores all look similar and retrieval picks the wrong section for an obvious question | With only five short sections, scores can be close for overlapping vocabulary (e.g. both "Disk Space" and "Database" mention storage-adjacent words). This is the generalization limit flagged in the Concept section — a sign to move to real embeddings once you have more than a handful of documents |
| `AccessDeniedException` calling `converse` | Confirm you're signed in as `opskart-admin` with `aws sts get-caller-identity`, and that the Bedrock IAM policy from Chapter 2 still allows `bedrock:InvokeModel` on a wildcard-region `us.amazon.nova-micro-v1:0` ARN |

## Interview questions

1. What does TF-IDF measure, in plain terms, and why does a common word across every document get a low weight instead of a high one?
2. Why would you retrieve a relevant chunk of text before calling the model, instead of just pasting the entire document into every prompt?
3. At what point does a hand-rolled scoring method like this chapter's stop being good enough, and what would you reach for instead?
4. This chapter has two separate safety nets against the model inventing an answer — a score threshold and a system-prompt instruction. Why both, instead of relying on just one?

## Checklist

- [ ] Wrote `runbook.md` with five independent sections
- [ ] Built `retrieve.py` and confirmed it picks the right section for differently-worded questions
- [ ] Built `mini_rag.py` and got a grounded, runbook-only answer
- [ ] Tested an out-of-scope question and got "not covered" instead of a guess
- [ ] Understand when this approach stops being good enough and a real vector database is worth it
