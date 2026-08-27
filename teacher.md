# Teacher Guide

Everything the provisioning workflow needs lives in `dev4Info.txt` at the repo root.
Change it, commit, push -- no workflow edits required.

```
# PREFIX: dev4-
# TEMPLATE: fsprojectportfolioiv_handout-dev4-starting-materials-v1-1-PPIV_HANDOUT2604
# COURSE_CODE: <sha-256 of the course code>
```

There is no student roster. Anyone who can open an issue on this repo and knows the
course code gets a repository. There is no cap.

**Repositories are permanent.** A student keeps the same repo for the whole course --
they are not reissued month to month. A returning student who submits the form again is
pointed back at the repo they already have.

---

## The course code

One code for the life of the course. Hand it out once; every student who enters the
course uses the same one. There is nothing to rotate month to month.

### Generating the hash

```bash
printf '%s' "your-course-code" | sha256sum | awk '{print $1}'
```

Use `printf '%s'`, **not** `echo`. `echo` appends a newline that gets hashed, producing a
digest that never matches and an "incorrect code" error with no explanation.

Paste the result after `# COURSE_CODE: ` in `dev4Info.txt`, commit, and push. Share the
plaintext code via FSO. Never commit the plaintext.

The code is compared case-sensitively.

### Changing the template

Update `TEMPLATE` when the starting materials change. Existing student repos are not
touched; only repos created afterwards use the new template.

> If you change `TEMPLATE`, add that repo to the GitHub App's selected repository list,
> or provisioning fails with a 404. Don't forget to click **Save**.

---

## What the workflow does

1. Student opens an issue from the **Request Assignment Repository** template and enters
   the course code.
2. The workflow redacts the code from the issue body immediately.
3. Validates it against `COURSE_CODE`; on failure, comments the reason and closes the
   issue as *not planned*.
4. Checks whether `PREFIX + githubusername` already exists. If so, points the student at
   their existing repo and closes the issue. This is the normal path for a returning student.
5. Generates a private repo from `TEMPLATE` (generate, not fork). The repo has whatever
   branches the template has -- the workflow creates none.
6. Adds the student as a **write** collaborator, which emails them an invitation.
7. Comments the repo URL and closes the issue.

---

## Listing student repositories

Repositories are named `<PREFIX><githubusername>`, e.g. `dev4-jdoe`. No topics are set,
so filter by the name prefix.

```bash
# Names
gh repo list FSProjectPortfolioIV --source --limit 1000 --json name --jq '.[].name' | grep '^dev4-'

# Name and URL
gh repo list FSProjectPortfolioIV --source --limit 1000 --json name,url --jq '.[] | "\(.name)	\(.url)"' | grep '^dev4-'

# Count
gh repo list FSProjectPortfolioIV --source --limit 1000 --json name --jq '.[].name' | grep -c '^dev4-'
```

> `gh repo list` requires the `repo` scope. Run `gh auth login` if you haven't.
> `--limit 1000` is the ceiling; raise it if the org ever exceeds that many repos.

---


## Cloning student repositories

Repositories are named `<PREFIX><githubusername>`, e.g. `dev4-jdoe`.

```bash
# Dry run -- see what would be cloned
gh repo list FSProjectPortfolioIV --source --limit 1000 --json sshUrl --jq '.[].sshUrl' | grep 'dev4-'

# Clone them all into the current directory
gh repo list FSProjectPortfolioIV --source --limit 1000 --json sshUrl --jq '.[].sshUrl' | grep 'dev4-' | xargs -I {} git clone {}
```

Each repo clones into its own subdirectory.

---

## Deleting student repositories

```bash
# Dry run FIRST -- verify the list
gh repo list FSProjectPortfolioIV --source --limit 1000 --json name --jq '.[].name' | grep '^dev4-'

# Then delete (uncomment the pipe once you've verified)
gh repo list FSProjectPortfolioIV --source --limit 1000 --json name --jq '.[].name' | grep '^dev4-' # | xargs -I {} gh repo delete "FSProjectPortfolioIV/{}" --yes
```

> **Warning:** `--yes` skips confirmation. Deletion is permanent. Always run the dry run first.

---

## Managing enrollment issues

```bash
# List closed issues
gh issue list --repo FSProjectPortfolioIV/DEV4CourseEnrollment --state closed --json number,title --jq '.[] | "\(.number)\t\(.title)"'

# Count them
gh issue list --repo FSProjectPortfolioIV/DEV4CourseEnrollment --state closed --json number --jq 'length'

# Delete all closed issues (uncomment the pipe once verified)
gh issue list --repo FSProjectPortfolioIV/DEV4CourseEnrollment --state closed --json number --jq '.[].number' # | xargs -I {} gh issue delete {} --repo FSProjectPortfolioIV/DEV4CourseEnrollment --yes
```

The `cleanup-issues.yml` workflow already redacts comments on issues closed more than
30 minutes ago, so repo URLs don't linger. This is only for tidying the list.

---

## Troubleshooting

| Symptom | Cause |
|---|---|
| "code is incorrect" for everyone | Hash generated with `echo` instead of `printf '%s'`, or `COURSE_CODE` not pushed |
| 404 when generating the repo | `TEMPLATE` not in the GitHub App's selected repository list |
| Workflow doesn't trigger | Issue missing the `provision-repo` label -- students must use the issue template, not a blank issue |
| Repo has no files | Template was empty, or the 45s generate wait timed out |
