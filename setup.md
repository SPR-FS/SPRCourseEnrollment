# Setup Guide

This guide walks you through the complete initial setup of the OPS Course Enrollment system. Complete every section in order before accepting student enrollment requests.

---

## Prerequisites

- **GitHub account** that is an **owner** of the `FullSailGameStudies` organization (or has org admin + repo admin on all relevant repos)
- **GitHub CLI** installed and authenticated (`gh auth login`)
- **Git** installed and configured
- Admin access to create repositories and secrets in the `FullSailGameStudies` organization

---

## 1. Create the Enrollment Repository

If this repository does not already exist in the organization:

```bash
gh repo create FullSailGameStudies/OPSCourseEnrollment --private --confirm-init
```

Then push this project's code to it:

```bash
git remote add origin https://github.com/FullSailGameStudies/OPSCourseEnrollment.git
git push -u origin main
```

> If you cloned this repo from an existing remote, skip this step.

---

## 2. Create Template Repositories

The provision workflow creates student repos from templates named `OPS_Lab3`, `OPS_Lab4`, `OPS_Lab5`, `OPS_Lab6`, and `OPS_Lab7` (see the `LABS` variable in `provision.yml`). Each template must:

1. **Exist** in the `FullSailGameStudies` organization.
2. Be marked as a **template repository** (Settings > "Template repository" checkbox).

To create a template repo and mark it as a template:

```bash
gh repo create FullSailGameStudies/OPS_Lab3 --private --template=false
# Repeat for Lab 4–7
```

Then in the GitHub UI for each repo: **Settings** > check **Template repository**.

Or via the API:

```bash
gh api --method PATCH /repos/FullSailGameStudies/OPS_Lab3 -f is_template=true
```

> **Important:** The `is_template` flag must be `true` on every template repo. The `POST /repos/{owner}/{repo}/generate` endpoint will fail otherwise.

---

## 3. Create the `ORG_ADMIN_TOKEN`

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

> **Note:** The token owner must be a **member** (or owner) of the `FullSailGameStudies` organization. The `generate` template endpoint requires org membership.

---

## 4. Add the Token as a Repository Secret

Add the PAT from step 3 as a secret named `ORG_ADMIN_TOKEN` in the enrollment repository:

```bash
gh secret set ORG_ADMIN_TOKEN --repo FullSailGameStudies/OPSCourseEnrollment
# Paste the token when prompted, then press Enter
```

Verify it was saved:

```bash
gh secret list --repo FullSailGameStudies/OPSCourseEnrollment
```

You should see `ORG_ADMIN_TOKEN` in the list.

---

## 5. Configure the Claim Code (Passcode)

The workflow validates a shared "claim code" that students enter when submitting their enrollment issue. The code is stored as a SHA-256 hash in the `EXPECTED_HASH` environment variable inside `provision.yml`.

### Generate a new passcode hash

```bash
printf '%s' "your-secret-passcode" | sha256sum | awk '{print $1}'
```

### Update the workflow

1. Open `.github/workflows/provision.yml`
2. Find the `EXPECTED_HASH` variable (line 17) and replace its value with the hash you generated:

   ```yaml
   EXPECTED_HASH: "your-new-hash-here"
   ```

3. Commit and push the change:

   ```bash
   git add .github/workflows/provision.yml
   git commit -m "Update claim code hash for new term"
   git push
   ```

4. Share the **plaintext** passcode with your students (e.g. post it in FSO / the LMS). Never commit the plaintext passcode.

---

## 6. Populate `hash.db` with Authorized Student Emails

The workflow checks each student's email against `hash.db` before provisioning. Only students whose email hash appears in this file will be authorized.

### Add a single student

```bash
printf '%s' "studentname@student.fullsail.edu" | tr -d '\r' | sed -e 's/^[[:space:]]*//' -e 's/[[:space:]]*$//' | tr '[:upper:]' '[:lower:]' | sha256sum | awk '{print $1}' >> hash.db
```

### Bulk add from a file

Create a plain text file (e.g. `students.txt`) with one email per line, then run:

```bash
while IFS= read -r email; do
  printf '%s' "$email" | tr -d '\r' | sed -e 's/^[[:space:]]*//' -e 's/[[:space:]]*$//' | tr '[:upper:]' '[:lower:]' | sha256sum | awk '{print $1}' >> hash.db
done < students.txt
```

### Commit the changes

```bash
git add hash.db
git commit -m "Add authorized students for new term"
git push
```

> See `teacher.md` for instructions on removing students and other ongoing maintenance.

---

## 7. Enable GitHub Actions and Workflow Permissions

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

## 8. Verify the Issue Template

The enrollment repo should have a GitHub issue template at `.github/ISSUE_TEMPLATE/student-invite.yml`. This template:

- Is titled "Request Assignment Repository"
- Auto-applies the `provision-repo` label (which triggers the workflow)
- Collects the student's Full Sail email and claim code

If the file is missing or corrupted, restore it from this repository. The `provision-repo` label must exist in the repo for the issue template to apply it:

```bash
gh label create provision-repo --repo FullSailGameStudies/OPSCourseEnrollment --color FBCA04 --description "Triggers the repository provisioning workflow"
```

---

## 9. Test the Workflow

Before announcing to students, test the full flow end-to-end:

1. **Ensure your own email hash is in `hash.db`** (or use a test email)
2. **Know the current claim code** (the plaintext that hashes to `EXPECTED_HASH`)
3. Go to the enrollment repo on GitHub and open a new issue using the "Request Assignment Repository" template
4. Enter a valid Full Sail email and the current claim code
5. Submit the issue

### Expected behavior

- The workflow runs automatically (triggered by the `provision-repo` label on the issue)
- Within a few minutes, you should see:
  - A comment on the issue confirming verification passed
  - Five new private repos created in the org (e.g. `OPS_JUL_Lab3_username`, `OPS_JUL_Lab4_username`, etc.)
  - Each repo has the student added as a collaborator with write access
  - Each repo has topics `ops-student` and `not-graded`
  - Each repo has an `assignment` branch and an open "Grading and Feedback" PR
  - A final comment on the issue with links to all repos, then the issue is closed

### If the workflow fails

- Check the **Actions** tab in the enrollment repo for error logs
- Verify `ORG_ADMIN_TOKEN` is set correctly and has not expired
- Verify the template repos exist and are marked as template repositories
- Verify the claim code hash matches `EXPECTED_HASH`
- Verify the student email hash exists in `hash.db`

---

## 10. Cleanup Workflow (Automatic)

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
| `contents: read` | Checkout the enrollment repo (for `hash.db`) |
