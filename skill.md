---
name: openverification-review
description: >
  Review one paper of the OpenAI math collection (github.com/openai/math) for mathematical
  errors with parallel sub-agents, and write a JSON review to upload to OpenVerification
  (openaireview.org/contribute.html).
  Usage: /openverification-review <paper id>
---

Review the paper that the user names, and write one JSON file that the user uploads to OpenVerification. Do every step below in order.

The paper is a pure mathematics manuscript written by an AI model. The review must focus on mathematical correctness: false statements, steps that do not follow, hypotheses that are used but never established, and cited results that are misquoted or applied outside their range.

## Input

The user gives a paper id. It is the name of the paper's folder under `preprints/` in the repository `openai/math`, for example `The-p-adic-section-conjecture-September-24-2026`. If the user gives no id, ask for one and point them to https://openaireview.org/contribute.html, which lists the papers that have no review yet.

Use this fixed version of the collection, so that your quotes match the PDF that the website shows:

    COMMIT=adc7f1241b42e322a6451854ab7e4b4c146bf78a

## Step 1. Get the LaTeX source

Make a work folder and download only this paper's folder:

```bash
mkdir -p openverification_review && cd openverification_review
git clone --filter=blob:none --no-checkout https://github.com/openai/math.git math
git -C math sparse-checkout set "preprints/<paper id>"
git -C math checkout $COMMIT
```

Find the main `.tex` file. It contains `\begin{document}`. If several files do, use the one that gives the longest paper after its `\input`, `\include`, and `\subfile` commands are followed. Read the main file and every file it includes. Do not edit any of these files, because every quote in your review must be an exact copy of the source.

If the folder does not exist at this commit, stop and tell the user.

## Step 2. Understand the paper

Read the complete source, including all appendices. Then write `summary.md` in the work folder with these parts:

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
Write all mathematics in LaTeX between dollar signs in title and explanation, with no plain-text or Unicode math. If you find no issue, write [].
```

## Step 4. Consolidate

Read every file in `comments/`. Then:

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

Write `openverification_review.json` in the work folder:

```json
{
  "collection": "openai-math",
  "paper_id": "<paper id>",
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
- At most 60 issues.
- State `agent` and `model` truthfully. Ask the user if you do not know them.

Then check the file. Run this from the work folder:

```bash
python3 - <<'EOF'
import json, pathlib, re
review = json.load(open("openverification_review.json"))
squash = lambda text: re.sub(r"\s+", "", text)
source = squash("".join(p.read_text(errors="replace") for p in pathlib.Path("math/preprints", review["paper_id"]).rglob("*.tex")))
for number, issue in enumerate(review["issues"], 1):
    assert issue["severity"] in ("major", "moderate", "minor"), f"issue {number}: bad severity"
    assert issue["title"] and issue["explanation"], f"issue {number}: empty field"
    if squash(issue["quote"]) not in source:
        print(f"issue {number}: the quote is not in the source")
print(len(review["issues"]), "issues checked")
EOF
```

Fix every quote that the check reports, and run the check again.

## Step 6. Tell the user

Give the user one or two sentences on what you found and the number of issues per severity. Then tell them:

```
The review is in openverification_review/openverification_review.json.
Upload it at https://openaireview.org/contribute.html
```
