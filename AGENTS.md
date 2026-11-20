# Repository Context

This is a personal **TIL (Today I Learned)** repository containing structured study notes, code examples, and Socratic Tutor session summaries / skill maps.

# Commit Workflow Trigger

Whenever the user sends the message `commit` (or asks to commit changes), execute the following steps strictly:

1. **Inspect Changes:**
   - Run `git status` and `git diff` (staged and unstaged).
   - Read newly created/untracked note files to identify the exact topic(s) and concepts covered.

2. **Group into Atomic Commits:**
   - If changes span multiple unrelated subjects (e.g., React notes and SQL notes), split them into separate logical commits per topic.
   - If all changes belong to one topic or session, keep them in a single commit.

3. **Apply Commit Standards (Conventional Commits + 50/72 Rule):**
   - **Header Format:** `<type>(<topic>): <imperative summary>`
     - Use `docs` for all learning notes, concept explanations, and Socratic Tutor summaries/skill maps.
     - Use `chore` only for repo maintenance, templates, or rules updates.
     - `<topic>` must be short and lowercase (e.g., `react`, `ts`, `sql`, `git`, `algorithms`).
     - `<imperative summary>` must use imperative verbs (`add`, `summarize`, `update`, `refine`), stay under **50 characters**, and have **no trailing period**.
   - **Body (Required for multi-concept notes or Socratic Tutor milestones; optional for minor typo fixes):**
     - Separate from header with one blank line.
     - Wrap lines at **72 characters**.
     - Use concise bullet points (`- `) listing the specific concepts learned or skill-map milestones achieved.

4. **Output & Confirmation:**
   - Show the proposed commit message(s) and the exact `git add <files> && git commit -m "..." -m "..."` command(s).
   - **Guardrail:** Do NOT run `git commit` or `git push` automatically. Wait for explicit user approval before executing any Git write commands.
