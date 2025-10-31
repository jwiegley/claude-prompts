Ultrathink. Use task-orchestrator to analyze and coordinate work

Pay close attention to the comments and restrictions in @CLAUDE.md

When working on the provided project:

- Do not, under any circumstances, decrypt the SOPS secrets.yaml file. See the @CLAUDE.md file for extensive notes on this important security consideration.
- Use Web Search and Perplexity as need for research and discovering resources.
- Use sequential-thinking when appropriate to break down tasks further.
- Use context7 whenever code examples might help.
- Each time before you intend to build or switch to a new configuration, use `touch` a create a file named `/etc/nixos/.nixos-build`. Remove this file when the build or the switch is completed. If you see that this file already exists, wait for up to ten minutes, checking every 10 seconds during that time to see if the file has been removed. This way, multiple nixos jobs can be working with the system at the same time, but only one will be building or switching at any given time.
- Use memory-keeper with channel: <project_name>
- Save progress at every major milestone
- Document all decisions with category: "decision"
- Track implementation status with category: "progress"
- Before claiming anything is complete, save test results

The orchestrator will:

1. Initialize session with project name as channel
2. Save findings during investigation
3. Create checkpoint before major changes
4. Document what actually works vs what should work
5. Check task dependencies
6. Identify tasks that can run in parallel
7. Deploy executors for available work
8. Monitor progress and deploy new executors as tasks complete

Use task-executor for specific task implementation

When the orchestrator identifies task 2.3 needs work: The executor will implement that specific task

Do not pause your work to ask if I would like to review or continue. Work continuously until the entire project is completed. Commit your work to Git after each time you reach a point where no tasks are currently running, just before you begin the next task.
