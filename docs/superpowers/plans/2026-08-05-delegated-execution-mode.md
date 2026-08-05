# 委任実行モード(Execution Handoff 3 択化)Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** writing-plans の実行ハンドオフに第 3 の選択肢「委任実行」を追加し、コントローラサブエージェントが SDD ループを別ワークツリーで丸ごと実行できるようにする。

**Architecture:** SDD に委任コントローラ用プロンプトテンプレート(新規ファイル)と親側手順セクション(末尾追記)を加え、writing-plans の Execution Handoff を 3 択に書き換える。コントローラは決定的名のワークツリーで SDD ループを回し、QUESTION/DONE/FAILED の 3 ステータスだけを親に返す。

**Tech Stack:** Markdown スキルファイルのみ。コード・依存追加なし。

**Spec:** `docs/superpowers/specs/2026-08-05-delegated-execution-mode-design.md`

## Global Constraints

- 作業は分離ワークツリーのブランチで行い、完了後に dev へ統合する(main は upstream ミラーのためコミット禁止)
- 変更対象は `skills/subagent-driven-development/`(新規ファイル+SKILL.md 末尾追記)と `skills/writing-plans/SKILL.md`(Execution Handoff セクションのみ書き換え)。他は触らない
- フォーク独自箇所のマーカーは「(superpowers-light addition)」
- ステータス語彙は QUESTION / DONE / FAILED の 3 種のみ。迷ったら QUESTION
- ワークツリー名の導出式: `sdd-` + 計画ファイル basename から拡張子を除いたもの(例: `2026-08-05-foo.md` → `sdd-2026-08-05-foo`)。配置はネイティブツール時 `<リポジトリルート>/.claude/worktrees/<名前>/`、フォールバック時 `<リポジトリルート>/.worktrees/<名前>/` の 2 候補に固定(親はこの 2 候補を順に確認すれば発見できる)
- レポートパスの導出式: `<ワークツリー>/.superpowers/sdd/<計画 basename(拡張子なし)>/delegated-report.md`。コントローラは全ステータスメッセージに実際のワークツリー絶対パスを含め、親の事前計算とのずれを自己修正可能にする
- 計画中のコミット例では trailer を省略しているが、実装時は必ず Global Constraints の trailer を付けること
- コントローラは統合(finishing-a-development-branch、マージ、push)とワークスペース削除を行わない
- 各コミットメッセージ末尾に trailer「Co-Authored-By: Claude Fable 5 <noreply@anthropic.com>」を付ける

---

### Task 1: delegated-controller-prompt.md を作成

**Files:**
- Create: `skills/subagent-driven-development/delegated-controller-prompt.md`

**Interfaces:**
- Produces: プレースホルダー名 `[PLAN_FILE_PATH]` `[REPO_ROOT]` `[SDD_SKILL_DIR]` `[REPORT_FILE_PATH]`(Task 2 の親側手順がこの 4 つを埋める)

- [ ] **Step 1: ファイルを以下の内容で作成する**

`````markdown
# Delegated Controller Prompt Template

Use this template when dispatching a delegated controller subagent
(Execution Handoff option 3). The controller runs the entire
subagent-driven-development loop in an isolated worktree and returns to the
parent only at QUESTION, DONE, or FAILED.

**Dispatch:** exactly ONE controller, on the most capable available model
(this is an architecture/judgment role; its own subagents are tiered per
Model Selection). Fill [PLAN_FILE_PATH], [REPO_ROOT], [SDD_SKILL_DIR], and
[REPORT_FILE_PATH] — [SDD_SKILL_DIR] is the directory containing the
subagent-driven-development SKILL.md the parent itself is following (in a
plugin install this is NOT inside the project repository).

```
Subagent (general-purpose):
  description: "Delegated controller: execute plan"
  model: [most capable available]
  prompt: |
    You are the delegated controller for one implementation plan. You
    execute the whole plan by dispatching your own implementer and reviewer
    subagents, following the subagent-driven-development skill. Your parent
    session stays out of the loop except when you send it a QUESTION.

    **Plan:** [PLAN_FILE_PATH]
    **Repository root:** [REPO_ROOT]
    **SDD skill directory:** [SDD_SKILL_DIR]
    **Report file:** [REPORT_FILE_PATH]

    ## Workspace (overrides using-git-worktrees defaults)

    All work happens in an isolated worktree of [REPO_ROOT] named
    deterministically after the plan: `sdd-` + the plan's basename without
    its extension (plan `2026-08-05-foo.md` → worktree `sdd-2026-08-05-foo`).

    1. Check whether that worktree already exists (`git worktree list`).
    2. If it exists: enter it (native tool with its `path` argument), read
       the SDD ledger inside, and resume from where it left off.
    3. If not: create it with the native worktree tool using that exact
       name (native tools place it under [REPO_ROOT]/.claude/worktrees/);
       if no native tool accepts a name, fall back to
       `git worktree add [REPO_ROOT]/.worktrees/<name>`.

    Include the actual absolute worktree path in EVERY status message you
    send (QUESTION, DONE, and FAILED), so the parent never has to guess
    which of the two locations was used.

    Every file edit and every git command MUST target this worktree. Never
    touch the parent repository checkout directly. Pass the worktree path
    explicitly to every subagent you dispatch and instruct them to run git
    as `git -C <worktree-path>`.

    ## Execute

    Read [SDD_SKILL_DIR]/SKILL.md and follow it end to end: per-task
    implementer dispatch, task reviews, fix loops, the ledger, and the
    final whole-branch review. Its helper scripts are at
    [SDD_SKILL_DIR]/scripts/ and its prompt templates are in
    [SDD_SKILL_DIR]/. Choose subagent models per its Model Selection
    section.

    **Boundary — do NOT integrate.** Stop after the final whole-branch
    review and its fix wave. Do not run finishing-a-development-branch, do
    not merge or push, and do not delete the workspace or the worktree.
    The integration decision belongs to the human via your parent session.

    ## Reporting protocol

    You return to your parent ONLY these statuses. No progress reports in
    between; never message the parent per task.

    - **QUESTION** — a human decision is needed: a plan contradiction, a
      load-bearing finding still open when the fix-loop breaker trips, a
      BLOCKED implementer you cannot unblock, or any judgment call you
      cannot settle yourself. Stop, and state the question, the options,
      and the context in your message. The parent replies with the answer;
      resume with your context intact. When unsure whether something is a
      QUESTION or a FAILED, choose QUESTION.
    - **DONE** — every task and the final whole-branch review (plus its
      fix wave) are complete. Write the full report to [REPORT_FILE_PATH]:
      commit list, per-task review results, parked/deferred findings, and
      the Q&A exchanges. Your message to the parent is a summary under 15
      lines: status, branch name, worktree path, commit range, one-line
      review summary, and open parked items.
    - **FAILED** — environment-level inability to continue only (worktree
      cannot be created, repository corrupted, required tools missing).
      Write what the ledger shows and why you cannot continue to
      [REPORT_FILE_PATH], and summarize it in your message.
```

**Controller returns:** QUESTION (question + options + context) | DONE
(short summary; detail in report file) | FAILED (cause + ledger position)
`````

- [ ] **Step 2: ファイルが作成されたことを確認する**

Run: `ls skills/subagent-driven-development/`
Expected: 既存ファイル群に `delegated-controller-prompt.md` が加わっている

- [ ] **Step 3: Commit**

```bash
git add skills/subagent-driven-development/delegated-controller-prompt.md
git commit -m "feat(sdd): add delegated controller prompt template"
```

---

### Task 2: SDD SKILL.md に Delegated Mode セクションを追記

**Files:**
- Modify: `skills/subagent-driven-development/SKILL.md`(ファイル末尾に追記)

**Interfaces:**
- Consumes: Task 1 の `delegated-controller-prompt.md` とそのプレースホルダー 4 種

- [ ] **Step 1: ファイル末尾(「## Example Workflow」セクションの後)に以下を追記する**

````markdown

## Delegated Mode (superpowers-light addition)

The third Execution Handoff option: the parent session dispatches ONE
controller subagent that runs this entire skill in an isolated worktree.
The parent's context stays clean — it only relays questions and receives
the final report.

In delegated mode, the workspace instructions in
[delegated-controller-prompt.md](delegated-controller-prompt.md) override
this skill's Setup section and using-git-worktrees defaults: deterministic
naming wins, and an existing same-named worktree is resumed, never
recreated.

Parent-side procedure:

1. Compute the deterministic names from the plan file: worktree name
   `sdd-<plan basename without extension>` (located at
   `<repo root>/.claude/worktrees/<name>/` when a native worktree tool is
   used, or `<repo root>/.worktrees/<name>/` on the git fallback — check
   both when discovering an existing one), report file
   `<worktree>/.superpowers/sdd/<plan basename without extension>/delegated-report.md`.
   The controller echoes its actual worktree path in every status message,
   so precomputation only needs to be right for the dispatch prompt, not
   forever.
2. Dispatch exactly ONE controller on the most capable available model
   using [delegated-controller-prompt.md](delegated-controller-prompt.md),
   filling [PLAN_FILE_PATH], [REPO_ROOT], [SDD_SKILL_DIR] (this skill's
   own directory), and [REPORT_FILE_PATH]. Provide only those values —
   NEVER your session history. Record the returned agent id.
3. On **QUESTION**: relay the question to your human partner verbatim,
   wait for their answer, and send it back to the same agent id. The
   controller resumes with its context intact.
4. On **DONE**: read the short summary (open the full report file only if
   something needs checking), then use
   superpowers:finishing-a-development-branch with your human partner for
   the integration decision. Clean up the workspace and worktree only
   after integration completes.
5. On **FAILED**: read the report, fix the environment problem, then
   re-dispatch (step 2). The new controller finds the existing worktree by
   its deterministic name and resumes from the ledger inside.
6. Controller loss — sending a message to the agent id errors out, or the
   harness reports the agent dead — is handled exactly like FAILED:
   re-dispatch and let the ledger drive resumption. There is no active
   liveness monitoring.
````

- [ ] **Step 2: 編集結果を確認する**

Run: `grep -n "Delegated Mode (superpowers-light addition)" skills/subagent-driven-development/SKILL.md && tail -5 skills/subagent-driven-development/SKILL.md`
Expected: 見出しが 1 ヒットし、ファイル末尾が追記セクションの本文で終わっている

- [ ] **Step 3: Commit**

```bash
git add skills/subagent-driven-development/SKILL.md
git commit -m "feat(sdd): add parent-side Delegated Mode procedure"
```

---

### Task 3: writing-plans の Execution Handoff を 3 択化

**Files:**
- Modify: `skills/writing-plans/SKILL.md`(Execution Handoff セクションのみ)

**Interfaces:**
- Consumes: Task 2 の「Delegated Mode (superpowers-light addition)」セクション見出し(参照名として使用)

- [ ] **Step 1: Execution Handoff セクションを差し替える**

対象(現在のセクション全文、「## Execution Handoff」から次のファイル末尾または次セクションまで):

````markdown
## Execution Handoff

After saving the plan, offer execution choice:

**"Plan complete and saved to `docs/superpowers/plans/<filename>.md`. Two execution options:**

**1. Subagent-Driven (recommended)** - I dispatch a fresh subagent per task, review between tasks, fast iteration

**2. Inline Execution** - Execute tasks in this session using executing-plans, batch execution with checkpoints

**Which approach?"**

**If Subagent-Driven chosen:**
- **REQUIRED SUB-SKILL:** Use superpowers:subagent-driven-development
- Fresh subagent per task + two-stage review

**If Inline Execution chosen:**
- **REQUIRED SUB-SKILL:** Use superpowers:executing-plans
- Batch execution with checkpoints for review
````

置換後:

````markdown
## Execution Handoff

After saving the plan, offer execution choice:

**"Plan complete and saved to `docs/superpowers/plans/<filename>.md`. Three execution options:**

**1. Subagent-Driven (recommended)** - I dispatch a fresh subagent per task, review between tasks, fast iteration

**2. Inline Execution** - Execute tasks in this session using executing-plans, batch execution with checkpoints

**3. Delegated (superpowers-light addition)** - I hand the whole plan to one controller subagent that runs the subagent-driven loop in its own worktree; this session stays free and only relays its questions

**Which approach?"**

**If Subagent-Driven chosen:**
- **REQUIRED SUB-SKILL:** Use superpowers:subagent-driven-development
- Fresh subagent per task + two-stage review

**If Inline Execution chosen:**
- **REQUIRED SUB-SKILL:** Use superpowers:executing-plans
- Batch execution with checkpoints for review

**If Delegated chosen:**
- **REQUIRED SUB-SKILL:** Use superpowers:subagent-driven-development — follow its "Delegated Mode (superpowers-light addition)" section
- One controller subagent runs the whole loop; the parent only relays QUESTION / receives DONE or FAILED
````

- [ ] **Step 2: 編集結果を確認する**

Run: `grep -n "Three execution options\|If Delegated chosen" skills/writing-plans/SKILL.md`
Expected: 各 1 ヒット。`grep -n "Two execution options" skills/writing-plans/SKILL.md` は 0 ヒット

- [ ] **Step 3: Commit**

```bash
git add skills/writing-plans/SKILL.md
git commit -m "feat(writing-plans): three-way execution handoff with delegated mode"
```

---

### Task 4: 理解度テスト

**Files:**
- 変更なし(検証のみ。問題があれば該当ファイルを修正してコミット)

**Interfaces:**
- Consumes: Task 1〜3 の全成果物

- [ ] **Step 1: サブエージェント(general-purpose)を 1 体起動し、次のプロンプトを渡す(パスはワークツリーの絶対パスに展開)**

```
次の 3 ファイルを読むこと:
- <worktree>/skills/writing-plans/SKILL.md
- <worktree>/skills/subagent-driven-development/SKILL.md(末尾の Delegated Mode セクション)
- <worktree>/skills/subagent-driven-development/delegated-controller-prompt.md

あなたは writing-plans に従って計画を書き終え、ユーザーが実行方法として
「3. Delegated」を選んだ直後の親セッションだとする。

質問: あなたが取る行動を順番に列挙せよ。特に:
(a) 何体の何をどのモデルでディスパッチするか
(b) プロンプトに埋める 4 つの値と、含めてはいけないもの
(c) QUESTION / DONE / FAILED それぞれを受け取ったときの対応
(d) コントローラが死んだ場合の復旧手順(何をもって「死んだ」と判断するかを含む)
(e) コントローラ側がやってはいけないこと(統合まわり)
スキルに書かれていないことは推測で補わず「記載なし」と明示すること。
```

- [ ] **Step 2: 応答を判定する**

Expected(すべて満たすこと):
- (a) コントローラ 1 体、最上位モデル
- (b) [PLAN_FILE_PATH] [REPO_ROOT] [SDD_SKILL_DIR] [REPORT_FILE_PATH]。会話履歴は渡さない
- (c) QUESTION はユーザーへそのまま取り次ぎ回答を同じ agent id へ返す/DONE は要約を読み finishing-a-development-branch へ/FAILED は環境修復後に再ディスパッチ
- (d) メッセージ送信エラーまたはハーネスの死亡通知で判断し、再ディスパッチ(決定的名で既存ワークツリー発見→レジャー再開)
- (e) マージ・push・finishing-a-development-branch・ワークスペース/ワークツリー削除をしない

欠けがあれば該当記述を明確化して修正・コミットし、Step 1 を再実行する。

- [ ] **Step 3: 修正が発生した場合はコミットする**

```bash
git add skills/
git commit -m "fix(skills): clarify delegated-mode instructions per comprehension test"
```

(修正がなければスキップ)

---

### Task 5: 実地テスト(おもちゃ計画で委任モードを一周)

**Files:**
- 変更なし(スクラッチ領域でのみ作業。テスト後に破棄)

**Interfaces:**
- Consumes: Task 1〜4 の全成果物

- [ ] **Step 1: スクラッチ領域に使い捨てリポジトリを作る**

スクラッチディレクトリ配下に `delegated-e2e` ディレクトリを作成し、`git init` して空コミットを 1 つ置く。続いて `docs/plans/toy-plan.md` を次の内容で作成しコミットする(意図的な曖昧点: greeting の言語を指定しない):

````markdown
# Toy Plan

**Goal:** Add a greeting file.

### Task 1: greeting file

**Files:**
- Create: `hello.txt`

- [ ] **Step 1:** Create `hello.txt` containing a single-line greeting. The plan intentionally does not specify the language (English or Japanese) — if the choice matters, escalate rather than guess.
- [ ] **Step 2:** Commit with message "feat: add greeting"
````

- [ ] **Step 2: 委任モードを実行する**

SDD SKILL.md の Delegated Mode 手順どおり、コントローラを 1 体(最上位モデル)ディスパッチする。[PLAN_FILE_PATH]=toy-plan.md の絶対パス、[REPO_ROOT]=使い捨てリポジトリ、[SDD_SKILL_DIR]=このワークツリーの `skills/subagent-driven-development` の絶対パス、[REPORT_FILE_PATH]=導出式どおり。

- [ ] **Step 3: プロトコルを検証する**

Expected(すべて満たすこと):
- (a) コントローラが孫エージェント(実装者)をディスパッチして hello.txt を作らせる(コントローラ自身が直接編集しない)
- (b) 言語の曖昧点で QUESTION が返る → 「日本語で」と回答を返すと再開し、日本語の greeting で完了する
- (c) DONE 報告にブランチ名・ワークツリーパス・コミット範囲が含まれ、使い捨てリポジトリのデフォルトブランチにはマージされていない
- (d) 全ステータスメッセージに実ワークツリー絶対パスが含まれ、レポートファイルがその実パス配下の導出式の位置に存在する

QUESTION が一度も返らずに完了した場合、または他の期待が欠けた場合: テンプレート/セクションの該当記述を強化して修正・コミットし、Step 1 からやり直す(使い捨てリポジトリは作り直す)。

- [ ] **Step 4: テスト成果物を破棄する**

使い捨てリポジトリ(ワークツリー含む)をスクラッチ領域ごと削除する。

- [ ] **Step 5: 修正が発生した場合はコミットする**

```bash
git add skills/
git commit -m "fix(skills): harden delegated-mode instructions per e2e test"
```

(修正がなければスキップ)
