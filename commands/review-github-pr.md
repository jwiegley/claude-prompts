Please analyze and review the GitHub PR: $ARGUMENTS.

Follow these steps:

1. Use `git worktree` to create a worktree and branch for the PR inside a `work` sub-directory. for example, if the PR number is 1024, then create a branch named `pr-1024` and a working tree that has checked out that branch in `work/pr-1024`.
2. Use `gh pr view` to get the PR details
3. Understand the problem described in the PR
4. Search the codebase for relevant files
5. Review any comments that have already been made to the PR
6. Run all tests to verify the PR
7. Ensure code passes linting and type checking
8. Present your review as a Markdown file in the working tree, so it can be reviewed.
9. Do not post any comments to the PR, only make your comments locally here to me.
10. If any of the review comments are particularly clear, concise and obviously important, suggest submitting them as comments yourself, after I have a chance to look over the review report and confirm your submission.

Remember the following:

- Use the GitHub CLI (`gh`) for all GitHub-related tasks.
- Use cpp-pro, python-pro, emacs-lisp-pro, rust-pro or haskell-pro as needed for diagnosing and analyzing PRs, fixing code, and writing any new code.
- If this worktree is anywhere under the "positron" or "pos" directories, then use pal to confer with gemini-3-pro-preview and gpt-5.2-pro to reach consensus on your deep analysis and review.
- Use Web Search and Perplexity as need for research and discovering resources.
- Use sequential-thinking when appropriate to break down tasks further.
- Use context7 whenever code examples might help.
