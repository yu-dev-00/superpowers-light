# サブエージェント・ドキュメントレビュー導入 Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** brainstorming と writing-plans にセルフレビュー後のサブエージェント・ドキュメントレビューループを追加する。

**Architecture:** 既存のセルフレビューはそのまま残し(二段構え)、その直後に旧実装(コミット e6221a4 で削除)由来のレビュアープロンプトテンプレートを復元・調整して無条件ディスパッチする。SKILL.md への編集は追記中心で、upstream マージ時のコンフリクト面を最小化する。

**Tech Stack:** Markdown スキルファイルのみ。コード・依存追加なし。

**Spec:** `docs/superpowers/specs/2026-07-31-subagent-document-review-design.md`

## Global Constraints

- 作業ブランチは `dev`(main は upstream ミラーのためコミット禁止)
- 変更対象は `skills/brainstorming/` と `skills/writing-plans/` のみ。他スキルは触らない
- 既存文面の書き換えは「This is a checklist you run yourself — not a subagent dispatch.」の一文の置換と、チェックリスト/フローチャートへの行追加のみ。それ以外は追記
- フォーク独自セクションには `(superpowers-light addition)` マーカーを付け、将来のマージで識別可能にする
- レビュアーには会話履歴を渡さない。対象ファイルパスのみ(計画レビューはスペックパスも)
- レビューループは最大 3 周、超過時はユーザーへエスカレーション。指摘は助言扱い

---

### Task 1: writing-plans のレビュアープロンプトファイルを作成

**Files:**
- Create: `skills/writing-plans/plan-document-reviewer-prompt.md`

**Interfaces:**
- Produces: Task 2 の SKILL.md 追記セクションが `plan-document-reviewer-prompt.md` をファイル名で参照する

- [ ] **Step 1: ファイルを以下の内容で作成する**

````markdown
# Plan Document Reviewer Prompt Template

Use this template when dispatching a plan document reviewer subagent.

**Purpose:** Verify the plan is complete, matches the spec, and has proper task decomposition.

**Dispatch after:** The complete plan is written and the inline Self-Review fixes are done.

```
Task tool (general-purpose):
  description: "Review plan document"
  prompt: |
    You are a plan document reviewer. Verify this plan is complete and ready for implementation.

    **Plan to review:** [PLAN_FILE_PATH]
    **Spec for reference:** [SPEC_FILE_PATH]

    This document already passed the author's inline self-review. Surface-level
    problems (typos, placeholders) have likely been fixed. Focus on the problems
    the author cannot see from inside their own context: misreadings of the spec,
    internal contradictions, and unstated assumptions.

    ## What to Check

    | Category | What to Look For |
    |----------|------------------|
    | Completeness | TODOs, placeholders, incomplete tasks, missing steps |
    | Spec Alignment | Plan covers spec requirements, no major scope creep, no spec misreadings |
    | Task Decomposition | Tasks have clear boundaries, steps are actionable |
    | Buildability | Could an engineer follow this plan without getting stuck? |

    ## Calibration

    **Only flag issues that would cause real problems during implementation.**
    An implementer building the wrong thing or getting stuck is an issue.
    Minor wording, stylistic preferences, and "nice to have" suggestions are not.

    Approve unless there are serious gaps — missing requirements from the spec,
    contradictory steps, placeholder content, or tasks so vague they can't be acted on.

    ## Output Format

    ## Plan Review

    **Status:** Approved | Issues Found

    **Issues (if any):**
    - [Task X, Step Y]: [specific issue] - [why it matters for implementation]

    **Recommendations (advisory, do not block approval):**
    - [suggestions for improvement]
```

**Reviewer returns:** Status, Issues (if any), Recommendations
````

- [ ] **Step 2: ファイルが作成されたことを確認する**

Run: `ls skills/writing-plans/`
Expected: `SKILL.md` と `plan-document-reviewer-prompt.md` が並ぶ

- [ ] **Step 3: Commit**

```bash
git add skills/writing-plans/plan-document-reviewer-prompt.md
git commit -m "feat(writing-plans): restore plan reviewer prompt template with self-review-delta focus"
```

---

### Task 2: writing-plans/SKILL.md にレビューループを追記

**Files:**
- Modify: `skills/writing-plans/SKILL.md`(Self-Review セクション内の一文と、Execution Handoff 直前への追記)

**Interfaces:**
- Consumes: Task 1 の `plan-document-reviewer-prompt.md`(ファイル名で参照)

- [ ] **Step 1: 「not a subagent dispatch」の一文を置換する**

対象(1 箇所、Self-Review セクション冒頭):

```markdown
After writing the complete plan, look at the spec with fresh eyes and check the plan against it. This is a checklist you run yourself — not a subagent dispatch.
```

置換後:

```markdown
After writing the complete plan, look at the spec with fresh eyes and check the plan against it. Run this checklist yourself first — the subagent review loop comes after.
```

- [ ] **Step 2: 「## Execution Handoff」の直前に以下のセクションを挿入する**

```markdown
## Plan Review Loop (superpowers-light addition)

After your Self-Review fixes are in, dispatch a plan-document-reviewer subagent
using the template in `plan-document-reviewer-prompt.md`. Provide only the plan
file path and the spec file path — NEVER your session history. The reviewer must
read the documents fresh.

1. Dispatch one general-purpose subagent from the template, filling in
   [PLAN_FILE_PATH] and [SPEC_FILE_PATH].
2. If Issues Found: fix the issues yourself (you have the context), then
   re-dispatch the reviewer for the whole plan.
3. If Approved: summarize the reviewer's findings in chat, labeled as findings
   that survived self-review, then proceed to Execution Handoff.

**Loop guidance:**
- Maximum 3 review rounds. If not converged, surface the remaining
  disagreements to your human partner for a decision.
- Reviewer feedback is advisory — if you believe a finding is wrong, say so
  and explain why instead of blindly complying.

```

- [ ] **Step 3: 編集結果を確認する**

Run: `grep -n "not a subagent dispatch" skills/writing-plans/SKILL.md; grep -n "Plan Review Loop (superpowers-light addition)" skills/writing-plans/SKILL.md`
Expected: 1 つ目はヒットなし、2 つ目は 1 ヒット

- [ ] **Step 4: Commit**

```bash
git add skills/writing-plans/SKILL.md
git commit -m "feat(writing-plans): add subagent plan review loop after self-review"
```

---

### Task 3: brainstorming のレビュアープロンプトファイルを作成

**Files:**
- Create: `skills/brainstorming/spec-document-reviewer-prompt.md`

**Interfaces:**
- Produces: Task 4 の SKILL.md 追記セクションが `spec-document-reviewer-prompt.md` をファイル名で参照する

- [ ] **Step 1: ファイルを以下の内容で作成する**

````markdown
# Spec Document Reviewer Prompt Template

Use this template when dispatching a spec document reviewer subagent.

**Purpose:** Verify the spec is complete, consistent, and ready for implementation planning.

**Dispatch after:** Spec document is written and the inline Spec Self-Review fixes are done.

```
Task tool (general-purpose):
  description: "Review spec document"
  prompt: |
    You are a spec document reviewer. Verify this spec is complete and ready for planning.

    **Spec to review:** [SPEC_FILE_PATH]

    This document already passed the author's inline self-review. Surface-level
    problems (typos, placeholders) have likely been fixed. Focus on the problems
    the author cannot see from inside their own context: internal contradictions,
    ambiguous requirements, and unstated assumptions.

    ## What to Check

    | Category | What to Look For |
    |----------|------------------|
    | Completeness | TODOs, placeholders, "TBD", incomplete sections |
    | Consistency | Internal contradictions, conflicting requirements |
    | Clarity | Requirements ambiguous enough to cause someone to build the wrong thing |
    | Scope | Focused enough for a single plan — not covering multiple independent subsystems |
    | YAGNI | Unrequested features, over-engineering |

    ## Calibration

    **Only flag issues that would cause real problems during implementation planning.**
    A missing section, a contradiction, or a requirement so ambiguous it could be
    interpreted two different ways — those are issues. Minor wording improvements,
    stylistic preferences, and "sections less detailed than others" are not.

    Approve unless there are serious gaps that would lead to a flawed plan.

    ## Output Format

    ## Spec Review

    **Status:** Approved | Issues Found

    **Issues (if any):**
    - [Section X]: [specific issue] - [why it matters for planning]

    **Recommendations (advisory, do not block approval):**
    - [suggestions for improvement]
```

**Reviewer returns:** Status, Issues (if any), Recommendations
````

- [ ] **Step 2: ファイルが作成されたことを確認する**

Run: `ls skills/brainstorming/`
Expected: `SKILL.md`、`scripts`、`spec-document-reviewer-prompt.md`、`visual-companion.md` が並ぶ

- [ ] **Step 3: Commit**

```bash
git add skills/brainstorming/spec-document-reviewer-prompt.md
git commit -m "feat(brainstorming): restore spec reviewer prompt template with self-review-delta focus"
```

---

### Task 4: brainstorming/SKILL.md にレビューステップを追記

**Files:**
- Modify: `skills/brainstorming/SKILL.md`(チェックリスト、フローチャート、Spec Self-Review 直後への追記)

**Interfaces:**
- Consumes: Task 3 の `spec-document-reviewer-prompt.md`(ファイル名で参照)

- [ ] **Step 1: チェックリストの項目 7〜9 を差し替える**

対象(Checklist セクション末尾の 3 行):

```markdown
7. **Spec self-review** — quick inline check for placeholders, contradictions, ambiguity, scope (see below)
8. **User reviews written spec** — ask user to review the spec file before proceeding
9. **Transition to implementation** — invoke writing-plans skill to create implementation plan
```

置換後(4 行):

```markdown
7. **Spec self-review** — quick inline check for placeholders, contradictions, ambiguity, scope (see below)
8. **Spec subagent review** — dispatch a spec-document-reviewer subagent, fix and re-dispatch until approved, max 3 rounds (see below)
9. **User reviews written spec** — ask user to review the spec file before proceeding
10. **Transition to implementation** — invoke writing-plans skill to create implementation plan
```

- [ ] **Step 2: フローチャートにレビューノードを追加する**

対象(dot グラフ内の 2 箇所)。ノード宣言:

```dot
    "Spec Self-Review\n(fix inline)" [shape=box];
    "User reviews spec?" [shape=diamond];
```

置換後:

```dot
    "Spec Self-Review\n(fix inline)" [shape=box];
    "Subagent spec review\n(max 3 rounds)" [shape=box];
    "User reviews spec?" [shape=diamond];
```

エッジ:

```dot
    "Write design doc" -> "Spec Self-Review\n(fix inline)";
    "Spec Self-Review\n(fix inline)" -> "User reviews spec?";
```

置換後:

```dot
    "Write design doc" -> "Spec Self-Review\n(fix inline)";
    "Spec Self-Review\n(fix inline)" -> "Subagent spec review\n(max 3 rounds)";
    "Subagent spec review\n(max 3 rounds)" -> "User reviews spec?";
```

- [ ] **Step 3: Spec Self-Review セクション直後に以下を挿入する**

対象位置: 「Fix any issues inline. No need to re-review — just fix and move on.」の行と「**User Review Gate:**」の行の間。

```markdown
**Spec Subagent Review (superpowers-light addition):**
After your self-review fixes are in, dispatch a spec-document-reviewer subagent
using the template in `spec-document-reviewer-prompt.md`. Provide only the spec
file path — NEVER your session history. The reviewer must read the document fresh.

1. Dispatch one general-purpose subagent from the template, filling in [SPEC_FILE_PATH].
2. If Issues Found: fix the issues yourself, then re-dispatch the reviewer for the whole spec.
3. If Approved: summarize the reviewer's findings in chat, labeled as findings that
   survived self-review, then proceed to the User Review Gate.

Maximum 3 review rounds; if not converged, surface the remaining disagreements to
your human partner. Reviewer feedback is advisory — if you believe a finding is
wrong, say so and explain why instead of blindly complying.

```

- [ ] **Step 4: 編集結果を確認する**

Run: `grep -n "Spec subagent review\|Subagent spec review\|Spec Subagent Review" skills/brainstorming/SKILL.md`
Expected: チェックリスト 1 行、dot ノード宣言 1 行、dot エッジ 2 行、セクション見出し 1 行の計 5 行前後がヒット

- [ ] **Step 5: Commit**

```bash
git add skills/brainstorming/SKILL.md
git commit -m "feat(brainstorming): add subagent spec review step after self-review"
```

---

### Task 5: サブエージェントによる指示明瞭性テスト

**Files:**
- 変更なし(検証のみ。問題が見つかった場合は該当ファイルを修正)

**Interfaces:**
- Consumes: Task 1〜4 の全成果物

- [ ] **Step 1: writing-plans の挙動テストを実施する**

Task ツール(general-purpose)でサブエージェントを 1 体起動し、次のプロンプトを渡す(パスは絶対パスに展開すること):

```
以下は writing-plans スキルの内容です(ファイル: skills/writing-plans/SKILL.md と
skills/writing-plans/plan-document-reviewer-prompt.md を読むこと)。

あなたはこのスキルに従って架空の小さな機能(例: CLI ツールに --version フラグを追加)の
実装計画を書き終え、Self-Review の修正も済ませた直後だとする。

質問: 次にあなたが取る行動を、スキルの記述に基づいて順番に列挙せよ。
特に (a) 誰に何をディスパッチするか、(b) そのプロンプトに何を含め、何を含めては
いけないか、(c) 指摘が出た場合と出ない場合それぞれの次のステップ、(d) ループの
上限と超過時の動作、を明示すること。
```

- [ ] **Step 2: 応答を判定する**

Expected(すべて満たすこと):
- (a) general-purpose サブエージェントを 1 体、plan-document-reviewer-prompt.md のテンプレートでディスパッチすると答える
- (b) 計画パスとスペックパスのみを含め、会話履歴を含めてはいけないと答える
- (c) Issues Found なら自分で修正して全体を再ディスパッチ、Approved ならレビュアーの指摘をチャットに要約してから Execution Handoff に進むと答える
- (d) 最大 3 周、超過時はユーザー(human partner)にエスカレーションすると答える

1 つでも欠けたら、該当する記述(SKILL.md またはプロンプトファイル)を明確化して修正し、Step 1 を再実行する。

- [ ] **Step 3: brainstorming の挙動テストを実施する**

同様にサブエージェントを 1 体起動し、次のプロンプトを渡す:

```
以下は brainstorming スキルの内容です(ファイル: skills/brainstorming/SKILL.md と
skills/brainstorming/spec-document-reviewer-prompt.md を読むこと)。

あなたはこのスキルに従って設計を固め、スペックを docs/superpowers/specs/ に書き、
Spec Self-Review の修正も済ませた直後だとする。

質問: ユーザーにスペックのレビューを依頼する前に、あなたが取る行動を順番に列挙せよ。
特に (a) 誰に何をディスパッチするか、(b) そのプロンプトに何を含め、何を含めては
いけないか、(c) 指摘が出た場合の扱いとループ上限、を明示すること。
```

- [ ] **Step 4: 応答を判定する**

Expected(すべて満たすこと):
- (a) spec-document-reviewer-prompt.md のテンプレートで general-purpose サブエージェントを 1 体ディスパッチすると答える
- (b) スペックのファイルパスのみを含め、会話履歴を含めてはいけないと答える
- (c) 指摘は自分で修正して再ディスパッチ、最大 3 周、超過時はエスカレーション、指摘は助言扱いと答える

欠けがあれば該当記述を修正し、Step 3 を再実行する。

- [ ] **Step 5: 修正が発生した場合はコミットする**

```bash
git add skills/
git commit -m "fix(skills): clarify subagent review instructions per behavior test"
```

(修正がなければこのステップはスキップ)

- [ ] **Step 6: dev を push する**

```bash
git push origin dev
```

Expected: origin/dev が更新される
