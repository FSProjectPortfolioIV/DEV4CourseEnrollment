# Teacher Guide

Everything needed lives in `dev4Info.txt` at the repo root.
If you need to update the template repo or the course claim code, update the `dev4Info.txt`, commit and push.

```
# PREFIX: dev4-
# TEMPLATE: PPIV_Handout
# COURSE_CODE: <hashed-claim-code>
```

**Repositories are permanent.** Retake students should continue from where they left off and incorporate any previous feedback. 

---

### The course code

One code for the life of the course. Hand it out once, every student who enters the course uses the same one. There is nothing to rotate month to month.

### Generating the hash

Write down a claim code to give out. Use the command below to generate the hash variant.
```bash
printf '%s' "claim-code" | sha256sum | awk '{print $1}'
```
Replace the `# COURSE_CODE` in the `dev4Info.txt` with the hashed variant.

### Changing the template

Update `# TEMPLATE` variable in `dev4Info.txt` when the starting materials change if you want a different repository to be the template.
If not, then commit any changes to the already created handout (PPIV_HANDOUT circa August 2026).

Existing student repos are not touched, only repos created afterwards use the new template.

---

### What the workflow does

1. Student creates an issue from the **Request Assignment Repository** template and enters the course code.
2. The workflow redacts the code from the issue body immediately.
3. Validates it against `# COURSE_CODE`, on failure, comments the reason and closes the issue as *not planned*.
4. Checks whether `# PREFIX + githubusername` already exists. If so, points the student at their existing repo and closes the issue. This is the normal path for a returning student.
5. Generates a private repo from `# TEMPLATE` (generate, not fork). The repo has whatever branches the template has.
6. Adds the student as a **write** collaborator, which emails them an invitation.
7. Comments the repo URL and closes the issue.

---

### Troubleshooting

| Symptom | Cause |
|---|---|
| "code is incorrect" for everyone | Hash generated with `echo` instead of `printf '%s'`, or `COURSE_CODE` not pushed |
| 404 when generating the repo | `TEMPLATE` not in the GitHub App's selected repository list |
| Workflow doesn't trigger | Issue missing the `provision-repo` label -- students must use the issue template, not a blank issue |
| Repo has no files | Template was empty, or the 45s generate wait timed out |
