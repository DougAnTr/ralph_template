## Ralph loop with Cursor CLI

This project is configured to run the [Ralph](https://github.com/snarktank/ralph) agent loop using the Cursor CLI (`agent`) as the primary AI coding tool, while still supporting the original Amp / Claude Code setup.

### Prerequisites

- **Git repository**: run Ralph from the root of a git repo.
- **`jq` installed**: used by the loop script (e.g. `sudo apt install jq` on Debian/Ubuntu).
- **Cursor CLI installed**:
  - Follow the official docs: see `https://cursor.com/docs/cli/overview`.
  - Ensure the `agent` command is available on your `PATH` in the shell where you run Ralph.
- **Optional tooling**:
  - Amp CLI and/or Claude Code CLI if you want to use `--tool amp` or `--tool claude` instead of Cursor.

### Ralph files in this repo

All Ralph-specific files live under `scripts/ralph`:

- `scripts/ralph/ralph.sh`: main Bash loop that runs the agent repeatedly.
- `scripts/ralph/CLAUDE.md`: prompt/instructions given to the agent each iteration.
- `scripts/ralph/prd.json`: PRD-style task list that Ralph uses to know what to work on.
- `scripts/ralph/progress.txt`: append-only log of what each iteration did and any learnings.

These mirror the structure described in the upstream Ralph repo, adapted to use Cursor CLI.

### Running Ralph with Cursor

From the project root:

```bash
./scripts/ralph/ralph.sh --tool cursor 10
```

- `--tool cursor` tells the loop to use Cursor’s `agent` CLI in non-interactive print mode.
- `10` is the maximum number of iterations to run before stopping (default is 10 if omitted).

On each iteration, the script will:

1. Read `scripts/ralph/prd.json` to determine the current feature branch and which user story to work on.
2. Call Cursor’s `agent` CLI with the instructions from `scripts/ralph/CLAUDE.md`.
3. Let the agent modify code, run checks, and (if configured in the prompt) commit changes.
4. Check the agent’s output for `<promise>COMPLETE</promise>`.
5. Stop early if **all** stories are complete, otherwise continue to the next iteration.

You will see iteration logs printed in the terminal while the loop runs.

### Switching tools (Amp / Claude Code)

The same Bash script also supports the original Ralph tools:

- **Cursor (recommended here)**:

  ```bash
  ./scripts/ralph/ralph.sh --tool cursor 10
  ```

- **Amp** (requires Amp CLI installed and configured):

  ```bash
  ./scripts/ralph/ralph.sh --tool amp 10
  ```

- **Claude Code** (requires `claude` CLI installed and authenticated):

  ```bash
  ./scripts/ralph/ralph.sh --tool claude 10
  ```

If you omit `--tool`, the script defaults to `amp` to match the upstream Ralph behavior.

### PRD and completion semantics

- `scripts/ralph/prd.json` defines:
  - `branchName`: the feature branch Ralph should use.
  - `userStories`: each story has an `id`, `title`, and `passes` flag.
- Ralph will:
  - Create or check out the branch from `branchName`.
  - Pick the highest-priority story where `passes: false`.
  - Implement **one** story per iteration.
  - Mark the story as passing by setting `passes: true` in `prd.json` once done.
  - Append a summary and learnings to `scripts/ralph/progress.txt`.
- When every story in `prd.json` has `passes: true`, the agent should output:

  ```text
  <promise>COMPLETE</promise>
  ```

  The Bash loop detects this marker and exits immediately, even if there are remaining iterations.

### Typical workflow

1. **Define your PRD**:
   - Create or update `scripts/ralph/prd.json` with your user stories.
   - Optionally use the Ralph PRD skills (see the upstream docs at `https://github.com/snarktank/ralph`) to generate this file.
2. **Review `scripts/ralph/CLAUDE.md`**:
   - Ensure it reflects this project’s coding standards and quality checks (typecheck, lint, tests, etc.).
3. **Run the loop with Cursor**:
   - `./scripts/ralph/ralph.sh --tool cursor 20`
4. **Monitor progress**:
   - Watch the terminal output.
   - Inspect `scripts/ralph/progress.txt` to see what each iteration did and what it learned.
   - Inspect git history to review commits created by the agent.

### Troubleshooting

- **`agent: command not found`**:
  - Install Cursor CLI and/or fix your `PATH` so `agent` is available, then rerun the script.
- **Loop hits max iterations without completing**:
  - Check `scripts/ralph/progress.txt` for errors or blockers.
  - Make sure your PRD stories are small enough to fit in a single agent context.
  - Adjust `max_iterations` (the last numeric argument) if needed.
- **You prefer interactive control**:
  - You can also run `agent` directly and paste the contents of `scripts/ralph/CLAUDE.md` into Cursor’s CLI or editor, but the Bash loop is designed for unattended runs.
