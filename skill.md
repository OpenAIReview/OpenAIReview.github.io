---
name: openverification-review
description: >
  Review one paper of the OpenAI math collection (github.com/openai/math) for mathematical
  errors with parallel sub-agents, and write a JSON review to upload to OpenVerification
  (openaireview.org/contribute.html).
  Usage: /openverification-review <paper id> in Claude Code, $openverification-review <paper id> in Codex.
---

Review the paper that the user names, and write one JSON file that the user uploads to OpenVerification. Do every step below in order.

The paper is a pure mathematics manuscript written by an AI model. The review must focus on mathematical correctness: false statements, steps that do not follow, hypotheses that are used but never established, and cited results that are misquoted or applied outside their range.

## Input

The user gives a paper id. It is the name of the paper's folder under `preprints/` in the repository `openai/math`, for example `The-p-adic-section-conjecture-September-24-2026`. If the user gives no id, ask for one and point them to https://openaireview.org/contribute.html, which lists the papers that have no review yet.

The work folder of this run is `openverification_review/<paper id>`, under the folder where you started. If it already exists, add a suffix such as `-2` to its name, so that this run starts in an empty folder. Below, `<WORK>` is this folder. Your shell can start each command in a different folder, so write every path in full.

## Step 1. Check that the paper has no review, and get the LaTeX source

First check that the paper is still open:

```bash
python3 - "<paper id>" <<'EOF'
import json, sys, urllib.request
paper = sys.argv[1]
load = lambda url: json.load(urllib.request.urlopen(url, timeout=30))
try:
    ours = load("https://openaireview.org/math/data/index.json")
    uploads = load("https://openaireview-backend-947059889174.us-central1.run.app/contributed-reviews/openai-math")
    taken = set(ours.get("reviewed_ids", [])) | {p["id"] for p in ours["papers"]} | {r["paper_id"] for r in uploads["reviews"]}
    print("TAKEN" if paper in taken else "OPEN")
except Exception as err:
    print("UNKNOWN", err)
EOF
```

If the result is `TAKEN`, stop and tell the user that this paper already has a review, and that https://openaireview.org/contribute.html lists the open papers. If it is `UNKNOWN`, continue.

Then download only this paper's folder, at the latest version of the collection:

```bash
mkdir -p "<WORK>"
git clone --filter=blob:none --no-checkout https://github.com/openai/math.git "<WORK>/math"
git -C "<WORK>/math" sparse-checkout set "preprints/<paper id>"
git -C "<WORK>/math" checkout
git -C "<WORK>/math" rev-parse HEAD
```

The last command prints the commit id (40 characters). Keep it for the review file, so that the website shows the PDF of the same version that you read.

Find the main `.tex` file. It contains `\begin{document}`. If several files do, use the one that gives the longest paper after its `\input`, `\include`, and `\subfile` commands are followed. Read the main file and every file it includes. Do not edit any of these files, because every quote in your review must be an exact copy of the source.

If the paper's folder does not exist, stop and tell the user.

## Step 2. Understand the paper

Read the complete source, including all appendices. Then write `<WORK>/summary.md` with these parts:

- The main theorems, each with its exact statement and location.
- The chain of the proof: which lemmas and propositions each main theorem uses, in order.
- Key definitions and notation.
- Every result that the paper takes from outside (a cited paper, a book, or another paper in the same collection), with the place where the paper uses it.
- A map of the sections, one line each.

## Step 3. Review with sub-agents

### 3a. Plan

Plan 7 to 10 sub-agents.

**Section sub-agents.** Give each one a section or a group of closely related sections. Together they must cover every lemma and every proof step in the chain of each main theorem, line by line.

**Cross-cutting sub-agents.** Choose 3 to 5 that fit the paper, for example:

- Does each cited result say what the paper claims, and does the paper meet its hypotheses at the place of use?
- Do the theorems stated in the abstract and introduction match what the body proves, with the same hypotheses?
- Are the symbols and definitions used the same way in every section?
- Does any step rest on another paper in the collection, and is that dependence stated?
- Does each case analysis cover all cases, and does each induction have a correct base case and step?

### 3b. Launch

Start all sub-agents in parallel if your tool supports sub-agents. If it does not, do each planned review yourself, one after another, with the same instructions.

Give each sub-agent these instructions, with the placeholders filled in:

```
You are a careful, expert mathematician. You check part of the paper "<TITLE>" for mathematical errors.

Read, in this order:
1. <WORK>/summary.md, for the main theorems, the notation, and the chain of the proof.
2. Your assigned part of the LaTeX source: <FILES AND SECTIONS>.
3. Other sections of the source when you need them to check a cross-reference.

Your focus: <ONE SENTENCE>.

Check every lemma and every proof step in your part line by line. Recompute the key steps yourself. Do not accept a step because the paper says that it is clear.

What to report:
- A statement that is false, or a step that does not follow from what came before.
- A hypothesis that a step uses but that the paper never establishes.
- A cited result that is misquoted, or that is applied outside the range where it holds.
- A sign error, a wrong constant, a wrong index, or a missing case.
- A symbol used in a way that contradicts its definition.
- A claim in the abstract or introduction that is stronger than what the body proves.

For each issue, say what concerned you, what you checked to resolve it, and what remains wrong. If you could not verify a cited result because you cannot read its source, say so plainly.

Be lenient with introductions that simplify on purpose, with forward references, and with notation that is defined later.

Do not report formatting, typesetting, or wording, or anything that a reader in the field resolves at once.

Favor well-developed arguments over surface observations. Combine findings that share a root cause. Report independent issues separately.

Write your findings as a JSON array to <WORK>/comments/<NAME>.json. Each issue is an object with:
  title        short descriptive title
  quote        an exact copy of a passage from the LaTeX source, character for character
  explanation  your reasoning
  confidence   "high", "medium", or "low"
Write all mathematics in LaTeX between dollar signs in title and explanation, with no plain-text or Unicode math. In JSON, write each LaTeX backslash as two backslashes. If you find no issue, write [].
If you cannot write the file, give the same JSON array as your final answer.
```

When the sub-agents end, check that `<WORK>/comments/` has one file per planned sub-agent. If a sub-agent gave its findings as an answer, write them to its file yourself. If a file is still missing, run that review again or do it yourself. Do not continue with a missing file.

## Step 4. Consolidate

Read every file in `<WORK>/comments/`. Then:

- **Verify.** For each finding, check the mathematics yourself against the paper before you keep it. Remove findings that the context, a standard convention, or a later section resolves.
- **Merge by root cause.** Two findings share a root cause if one fix resolves both. Merge them into one issue that makes the strongest version of the argument. Keep findings separate when they need different fixes or affect different results.
- **Check each quote.** Each quote must occur in the source exactly. Fix it or pick another passage if it does not.
- **Keep uncertain findings.** Do not drop an issue because it feels small. Say in the explanation what is uncertain.

Give each issue a severity:

- `major`: the issue puts a main theorem or a key step of its proof in doubt.
- `moderate`: a real error or gap that is local and can be fixed.
- `minor`: an ambiguity or a mild overstatement that the reader can resolve from context.

Give each issue a `comment_type`: `methodology` (a proof step is wrong or has a gap), `claim_accuracy` (a statement or a citation is inaccurate), `missing_information` (a needed argument or hypothesis is absent), or `presentation`.

## Step 5. Write the review file

Write `<WORK>/openverification_review.json`:

```json
{
  "collection": "openai-math",
  "paper_id": "<paper id>",
  "commit": "<the commit id from Step 1>",
  "agent": "<the coding agent you run in, for example Claude Code or Codex>",
  "model": "<the model you run as>",
  "issues": [
    {
      "title": "...",
      "quote": "...",
      "explanation": "...",
      "comment_type": "methodology",
      "severity": "major"
    }
  ]
}
```

Rules:

- Order the issues from most to least serious.
- `quote` is an exact copy of the LaTeX source and at most 3,000 characters. One or two sentences is best.
- `title` is at most 300 characters and `explanation` at most 6,000.
- Write all mathematics in `title` and `explanation` in LaTeX between dollar signs.
- Outside the dollar signs, write plain text in `title` and `explanation`. Do not use `\ref`, `\eqref`, `\cite`, or accent commands there, because the website does not resolve them. Name a theorem or an equation in words, and write accented letters directly (é).
- In JSON, write each LaTeX backslash as two backslashes (`\\tau` for $\tau$). A single backslash before `t`, `n`, `b`, `f`, or `r` becomes an invisible character and breaks the mathematics.
- At most 60 issues.
- `commit` is the output of `git -C "<WORK>/math" rev-parse HEAD`.
- State `agent` and `model` truthfully. Ask the user if you do not know them.

Then check the file:

```bash
python3 - "<WORK>" <<'EOF'
import json, os, pathlib, re, sys
os.chdir(sys.argv[1])
review = json.load(open("openverification_review.json"))
problems = []
if not re.fullmatch(r"[0-9a-f]{40}", review.get("commit", "")):
    problems.append("commit must be the full commit id (40 characters)")
if len(review["issues"]) > 60:
    problems.append("more than 60 issues")
squash = lambda text: re.sub(r"\s+", "", text)
source = squash("".join(p.read_text(errors="replace") for p in pathlib.Path("math/preprints", review["paper_id"]).rglob("*.tex")))
limits = {"title": 300, "quote": 3000, "explanation": 6000}
lost_backslash = re.compile(r"[\x00-\x09\x0b-\x1f]|\n(u|abla|eq|ot|otin|leq|geq)(?![a-z])")
for number, issue in enumerate(review["issues"], 1):
    if issue.get("severity") not in ("major", "moderate", "minor"):
        problems.append(f"issue {number}: bad severity")
    for field, limit in limits.items():
        if len(issue.get(field, "")) > limit:
            problems.append(f"issue {number}: {field} is longer than {limit} characters")
    if not issue.get("title", "").strip() or not issue.get("explanation", "").strip():
        problems.append(f"issue {number}: empty title or explanation")
    if lost_backslash.search(issue.get("title", "") + " " + issue.get("explanation", "")):
        problems.append(f"issue {number}: a LaTeX backslash was lost (write two backslashes in JSON)")
    if squash(issue.get("quote", "")) not in source:
        problems.append(f"issue {number}: the quote is not in the source")
print("\n".join(problems) or "no problems")
print(len(review["issues"]), "issues checked")
EOF
```

Fix every problem that the check reports, and run the check again until it reports none.

## Step 6. Tell the user

Give the user one or two sentences on what you found and the number of issues per severity. Then tell them:

```
The review is in <WORK>/openverification_review.json.
Upload it at https://openaireview.org/contribute.html
```
