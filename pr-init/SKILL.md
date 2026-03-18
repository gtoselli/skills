---
name: pr-init
description: Generate a PR description for the current branch changes vs main. Invoke with /pr-init.
disable-model-invocation: true
allowed-tools: Bash(git *)
---

Use git to gather context, then write a PR description.

Steps:

1. Get the current branch name with git branch --show-current. If it is main or master, stop and ask the user which branch to use instead.
2. Get commits vs main with git log main..HEAD --oneline
3. Get changed files with git diff main...HEAD --stat
4. Get the full diff with git diff main...HEAD
5. Identify the WHY — the reason this PR exists (the problem it solves, the goal it achieves). If it is not clearly derivable from the diff or commit messages, stop and ask the user before writing the description.
6. Write the PR description using the format below. Output ONLY the description markdown with no extra commentary so the user can copy-paste it directly into GitHub.

## Format

[Short summary paragraph — 2-3 sentences max. Lead with the goal or problem being solved, framed in business/product terms if possible. End with the key outcome or how it improves things. No bullet points here, just prose.]

**TLDR**

- [one bullet per logical chunk — name the thing and say what it does, keep it concrete]

**Notes**

- [caveats, non-obvious decisions, or anything worth flagging for reviewers — omit this section entirely if there is nothing to note]
