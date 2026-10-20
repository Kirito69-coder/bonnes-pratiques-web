# Review prompt

Use this prompt to ask an agent to audit a Web project with the web-best-practices skill.

## Example prompt

```text
Use web-best-practices to review this Web project.

1. Identify the project type and architecture.
2. Inspect the relevant files for this context.
3. List findings as facts or assumptions, with file and line references when possible.
4. Classify each finding as P0, P1, P2 or P3.
5. Propose minimal corrections.
6. Call out any legal, privacy or security-sensitive items for human validation.

Do not include secrets or credentials in the output.
```

## Notes

- The prompt should be adapted to the project’s actual stack and risk level.
- This prompt is an example, not a guaranteed result. Results depend on the agent, the project files available, and the reviewer’s validation.
