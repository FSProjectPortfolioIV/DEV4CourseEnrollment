# Setup Guide

This guide walks you through the complete initial setup of a Course Enrollment system. Complete every section in order before accepting student enrollment requests.

---

## Prerequisites

- **GitHub Organization** 
   - https://docs.github.com/en/organizations/collaborating-with-groups-in-organizations/creating-a-new-organization-from-scratch 
- **GitHub account** that is an **owner** of the organization you're using for this (or has org admin + repo admin on all relevant repos)
- **GitHub CLI** installed and authenticated (`gh auth login`)
- **Git** installed and configured
- Admin access to create repositories and secrets in your organization

---

## 1. Create the Enrollment Repository

If this repository does not already exist in the organization:

```bash
gh repo create YourOrganizationName/RepositoryNameForCourseEnrollment --private --confirm-init
```

Then push this project's code to it:

```bash
git remote add origin https://github.com/DEV4-FS/DEV4CourseEnrollment.git
git push -u origin main
```

> If you cloned this repo from an existing remote, skip this step.

---

## 2. Create Template Repositories

The provision workflow creates student repos from templates (see the `LABS` variable in `provision.yml`). Each template must:

1. **Exist** in your organization.
2. Be marked as a **template repository** (Settings > "Template repository" checkbox).

To create a template repo and mark it as a template:

```bash
gh repo create YourOrganizationName/LabName --private --template=false
# Repeat for however many lab repositories you have
```

Then in the GitHub UI for each repo: **Settings** > check **Template repository**.

Or via the API:

```bash
gh api --method PATCH /repos/YourOrganizationName/LabName -f is_template=true
```

> **Important:** The `is_template` flag must be `true` on every template repo. The `POST /repos/{owner}/{repo}/generate` endpoint will fail otherwise.

---

## 3. Create the `ORG_ADMIN_TOKEN` (optional -- not used by the current workflows)

The workflow uses a Personal Access Token (PAT) stored as `ORG_ADMIN_TOKEN` to create repos from templates, add collaborators, set topics, clone/push, and create pull requests — all across the organization.

### Option A: Classic PAT (recommended for simplicity)

1. Go to **GitHub > Settings > Developer settings > Personal access tokens > Tokens (classic)**
2. Click **Generate new token (classic)**
3. Set an appropriate expiration (e.g. end of the semester)
4. Select the **`repo`** scope (this covers all needed permissions: contents, issues, pull requests, collaborators, administration, topics)
5. Generate and copy the token

### Option B: Fine-Grained PAT (more secure, more setup)

1. Go to **GitHub > Settings > Developer settings > Personal access tokens > Fine-grained tokens**
2. Click **Generate fine-grained token**
3. Set **Resource owner** to `FullSailGameStudies`
4. Set **Repository access** to **All repositories** (or select every template + enrollment repo)
5. Under **Repository permissions**, grant:
   - **Administration: Read and write** — create repos from templates, add collaborators, set topics
   - **Contents: Read and write** — clone repos, push branches
   - **Issues: Read and write** — edit issue body (redaction), comment, close
   - **Pull requests: Read and write** — create grading PRs
   - **Metadata: Read-only** — mandatory companion permission (auto-granted)
6. Generate and copy the token

> **Note:** The token owner must be a **member** (or owner) of the organization. The `generate` template endpoint requires org membership.

---

## 4. Add the Token as a Repository Secret

Add the PAT from step 3 as a secret named `ORG_ADMIN_TOKEN` in the enrollment repository:

```bash
gh secret set ORG_ADMIN_TOKEN --repo YourOrganizationName/RepositoryNameForCourseEnrollment
# Paste the token when prompted, then press Enter
```

Verify it was saved:

```bash
gh secret list --repo YourOrganizationName/RepositoryNameForCourseEnrollment
```

You should see `ORG_ADMIN_TOKEN` in the list.

This can also be done in the Github UI. 
1. On Github navigate to your enrollment repository's settings, RepositoryNameForCourseEnrollment > Settings
2. Secrets and Variables > Actions
3. Then click 'New Repository Secret'
4. Name the secret ORG_ADMIN_TOKEN and paste the token in the Secret section.
5. Click 'Add Secret'

---

## 5. Create and Configure the App

We need to create a GitHub App under the organization to authorize access to the private template repositories in the provisioning script

1. On GitHub navigate to your enrollment repository's settings, RepositoryNameForCourseEnrollment > Settings
2. Developer Settings > GitHub Apps
3. Click 'New GitHub App'
4. Give it a name (ex: DEV4 Enrollment) and homepage URL (you can just use google here, it doesn't matter)
5. Under Webhook uncheck Active
6. Set the following permissions (set all to read and write)
   - Repository Permissions 
      - Actions, Administration, Contents, Custom Properties, Discussions, Issues, Pull Requests, and Secrets
   - Organization Permissions
      - Secrets
7. Click 'Create GitHub App'

---

## 6. Add App data as Repository Secrets

Now we need to add 2 more secrets related to the App. Lets get the data we need first

1. On GitHub navigate to your enrollment repository's settings, RepositoryNameForCourseEnrollment > Settings
2. Developer Settings > GitHub Apps > Click Edit next to your App created in the previous step
3. Copy the App ID number and save it for the next steps
4. Scroll to the bottom, under the Private Keys sections click Generate a private key
5. When you do so you will see a new key hash generated in the list, ignore this information.
6. Generating the key will also trigger a file to be downloaded to your computer, a pem file. Note where that files is for the next steps

Now that we have the data, lets create our secrets

1. On Github navigate to your enrollment repository's settings, RepositoryNameForCourseEnrollment > Settings
2. Secrets and Variables > Actions
3. Then click 'New Repository Secret'
4. Name the first secret App_ID and paste the App ID copied above in the Secret section.
5. Click 'Add Secret'
6. Then click 'New Repository Secret' again
7. Name this secret APP_PRIVATE_KEY
8. Go to the pem file that was downloaded to your computer in the previous section and open it (notepad/text edit/etc)
9. Copy the entire file including the begin and end lines with all the dashes.
10. Paste that into the Secret section for this new secret
11. Click 'Add Secret'

---

## 7. Configure `dev4Info.txt`

All course settings live in `dev4Info.txt` at the repo root. The workflow reads them at
run time, so you never edit the YAML for routine changes.

```
# PREFIX: dev4-
# TEMPLATE: fsprojectportfolioiv_handout-dev4-starting-materials-v1-1-PPIV_HANDOUT2604
# COURSE_CODE: <sha-256 of the course code>
```

| Key | Meaning |
|---|---|
| `PREFIX` | Prepended to the student's GitHub login to name their repo. Keep it fixed -- students keep one repo for the whole course. |
| `TEMPLATE` | The single template repository new repos are generated from. |
| `COURSE_CODE` | SHA-256 of the course code. One code for the life of the course. |

### Generate the course code hash

```bash
printf '%s' "your-course-code" | sha256sum | awk '{print $1}'
```

Use `printf '%s'`, **not** `echo` -- `echo` appends a newline and changes the digest.

Paste the result after `# COURSE_CODE: `, commit, and push. Share the **plaintext** code
with students via FSO. Never commit the plaintext.

### Set the organization name

`ORG_NAME` is the one value still in the workflow, at the top of
`.github/workflows/provision.yml`. Set it once for your organization.

---

## 8. Authorization model

There is **no student roster and no email check**. Anyone who can open an issue on this
repository and knows the course code receives a repository, subject to:

- the code matching `COURSE_CODE`
- the student not already having a repo

There is no seat cap.

This is deliberate: the course code is distributed through FSO, which is already
authenticated, so the code itself is the access control. Keep the enrollment repository
private so the code hash and configuration are not publicly readable.

---


## 9. Enable GitHub Actions and Workflow Permissions

### Enable Actions

1. Go to the enrollment repo: **Settings > Actions > General**
2. Under **Actions permissions**, select **Allow all actions and reusable workflows**
3. Click **Save**

### Workflow permissions for the `GITHUB_TOKEN`

The `provision.yml` workflow already declares the needed permissions in its `permissions:` block:

```yaml
permissions:
  issues: write
  contents: read
```

However, GitHub also has an org/repo-level setting that can override this. Verify:

1. Go to **Settings > Actions > General** in the enrollment repo
2. Scroll to **Workflow permissions**
3. Ensure **Read and write permissions** is enabled (or that the workflow's inline `permissions:` block is respected)
4. Click **Save**

---

## 10. Verify the Issue Template

The enrollment repo should have a GitHub issue template at `.github/ISSUE_TEMPLATE/student-invite.yml`. This template:

- Is titled "Request Assignment Repository"
- Auto-applies the `provision-repo` label (which triggers the workflow)
- Collects only the course code (no email, no roster)

If the file is missing or corrupted, restore it from this repository. The `provision-repo` label must exist in the repo for the issue template to apply it:

```bash
gh label create provision-repo --repo FSProjectPortfolioIV/DEV4CourseEnrollment --color FBCA04 --description "Triggers the repository provisioning workflow"
```

---

## 11. Test the Workflow

Before announcing to students, test the full flow end-to-end:

1. **Know the course code** (the plaintext that hashes to `COURSE_CODE`)
3. Go to the enrollment repo on GitHub and open a new issue using the "Request Assignment Repository" template
4. Enter a valid Full Sail email and the current course code
5. Submit the issue

### Expected behavior

- The workflow runs automatically (triggered by the `provision-repo` label on the issue)
- Within a few minutes, you should see:
  - A comment on the issue confirming verification passed
  - Five new private repos created in the org (e.g. `DEV4_JUL_Lab3_username`, `DEV4_JUL_Lab4_username`, etc.)
  - Each repo has the student added as a collaborator with write access
  - Each repo has topics `dev4-student` and `not-graded`
  - Each repo has an `assignment` branch and an open "Grading and Feedback" PR
  - A final comment on the issue with links to all repos, then the issue is closed

### If the workflow fails

- Check the **Actions** tab in the enrollment repo for error logs
- Verify `APP_ID` and `APP_PRIVATE_KEY` are set correctly
- Verify the template repos exist and are marked as template repositories
- Verify the course code hash matches `COURSE_CODE` in `dev4Info.txt`
- Verify the course code was entered exactly (it is case-sensitive)

---

## 12. Cleanup Workflow (Automatic)

The `cleanup-issues.yml` workflow runs hourly and redacts comments on closed issues older than 30 minutes. This protects student privacy by removing email addresses and repo links that were posted in issue comments.

No additional setup is needed for this workflow — it uses the automatic `GITHUB_TOKEN` and the `issues: write` permission declared in the workflow file.

---

## Quick Reference: Secrets and Permissions Summary

| Secret | Used By | Purpose |
|--------|---------|---------|
| `ORG_ADMIN_TOKEN` | `provision.yml` | PAT for org-wide repo creation, collaborator management, topics, git push, PRs |
| `GITHUB_TOKEN` | `provision.yml`, `cleanup-issues.yml` | Auto-generated token for issue comments/closes on the enrollment repo |

### `ORG_ADMIN_TOKEN` minimum permissions

| Permission | Classic PAT Scope | Fine-Grained Equivalent |
|------------|-------------------|------------------------|
| Create repo from template | `repo` | Administration: write |
| Add collaborator | `repo` | Administration: write |
| Set topics | `repo` | Administration: write |
| Clone & push branches | `repo` | Contents: read + write |
| Edit issue body (redact) | `repo` | Issues: write |
| Create pull request | `repo` | Pull requests: write |
| (mandatory) | — | Metadata: read-only |

### `GITHUB_TOKEN` permissions (declared in workflow)

| Permission | Why |
|------------|-----|
| `issues: write` | Post comments, close issues, redact issue bodies |
| `contents: read` | Checkout the enrollment repo (for `dev4Info.txt`) |
