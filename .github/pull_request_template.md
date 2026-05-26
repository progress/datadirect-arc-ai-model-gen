## Summary

<!-- Briefly describe what this PR changes and why. -->

Fixes # <!-- link the related issue, if any -->

---

## Type of change

- [ ] Bug fix
- [ ] New feature or enhancement
- [ ] Documentation update
- [ ] Agent / prompt content change
- [ ] Refactor (no behavior change)
- [ ] Other (describe below)

---

## PR author checklist

### General
- [ ] My changes are focused -- this PR addresses one logical concern
- [ ] I have tested my changes manually and they work as expected
- [ ] I have updated relevant documentation (`README.md`, agent files, etc.) to reflect my changes

### Agent / prompt changes (complete if applicable)
- [ ] I have verified the agent produces correct output with the updated prompt
- [ ] I have not modified internal reference files (`global-lang-spec.md`, `rest-reference-template.rest`, status templates) without coordinating with a maintainer
- [ ] Generated output in `ai-output/` is **not** included in this PR (except intentional reference example updates)

### Security
- [ ] This PR does not introduce secrets, credentials, API keys, or tokens into any file
- [ ] This PR does not add new external dependencies without updating the Prerequisites section of `README.md`
- [ ] Any new file paths or inputs are validated or sanitized before use

### CLA
- [ ] I have signed the Progress Software CLA (external contributors only -- Progress employees can uncheck this)

---

## Testing notes

<!-- Describe what you tested and how. Include agent names, input files used, and a summary of observed output if applicable. -->
