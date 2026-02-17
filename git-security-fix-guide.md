# Git Security Fix: Removing `.env` from Repository History

## 🚨 The Problem
A `.env` file containing real credentials was committed and exists in git history.
Even if deleted in a new commit, the secrets are still visible via `git log`.

---

## Step 1 & 2 — Download & Extract the Repository

```bash
# Unzip the downloaded file
unzip q-git-revert-env.zip

# Navigate into the repo folder
cd q-git-revert-env
```

---

## Step 3 — Find the Commit That Added `.env`

```bash
git log --all --full-history -- .env
```

**Output from this repo:**
```
commit 0d3c5c22a21bd4d6b05ce31e7da4a51a19ac970d
Author: Developer <23f3002443@ds.study.iitm.ac.in>
Date:   Sun Sep 7 19:56:40 2025 +0000

    Set up logging configuration
```

> 💡 The `.env` was sneaked in under an unrelated commit message — a classic mistake.

You can also use this to see *all* commits with file-level changes:
```bash
git log --oneline --name-status
```

---

## Step 4 — Remove `.env` from Every Commit in History

```bash
git filter-branch --force --index-filter \
  'git rm --cached --ignore-unmatch .env' \
  --prune-empty --tag-name-filter cat -- --all
```

**What each flag does:**
| Flag | Purpose |
|------|---------|
| `--force` | Override safety check if filter-branch was run before |
| `--index-filter` | Faster than `--tree-filter`; rewrites the index directly |
| `git rm --cached --ignore-unmatch .env` | Removes `.env` from the index; `--ignore-unmatch` skips commits where it didn't exist |
| `--prune-empty` | Removes commits that become empty after `.env` is deleted |
| `--tag-name-filter cat` | Rewrites any tags to point to new commits |
| `-- --all` | Apply to every branch and ref |

---

## Step 5 — Verify `.env` is Completely Gone

```bash
# Remove old refs left by filter-branch
rm -rf .git/refs/original/

# Expire the reflog immediately
git reflog expire --expire=now --all

# Force garbage collection to purge old objects
git gc --prune=now --aggressive

# Confirm .env appears in NO commits (should return empty)
git log --all --full-history -- .env
```

> ✅ **Empty output = success.** The file no longer exists in any commit.

---

## Step 6 — Create `.gitignore` and Add `.env`

```bash
cat > .gitignore << 'EOF'
# Environment variables - NEVER commit these!
.env
.env.local
.env.*.local

# Python
__pycache__/
*.py[cod]
*.pyo
.venv/
venv/

# OS
.DS_Store
Thumbs.db

# IDE
.vscode/
.idea/
EOF
```

---

## Step 7 — Create `.env.example` with Placeholder Values

```bash
cat > .env.example << 'EOF'
# Copy this file to .env and fill in your actual values
# NEVER commit your real .env file!

# AWS Credentials
AWS_SECRET_ACCESS_KEY=your_aws_secret_access_key_here

# AI/ML Services
AIPIPE_TOKEN=your_aipipe_jwt_token_here

# Database
DATABASE_URL=postgres://username:password@host:5432/dbname

# API Keys
API_SECRET=your_api_secret_key_here

# Security
JWT_SECRET=your_jwt_secret_key_minimum_32_chars_here
EOF
```

> 💡 `.env.example` tells teammates *which* variables are needed without exposing real values.

---

## Step 8 — Commit Both Files

```bash
git add .gitignore .env.example

git commit -m "Add .gitignore and .env.example for security best practices

- Add .env to .gitignore to prevent accidental secret commits
- Add .env.example as a template for required environment variables
- Removed .env from git history (contained sensitive credentials)"
```

---

## Step 9 — Push to GitHub (Force Push Required)

Because we rewrote history, a normal `git push` will be rejected.
A **force push** is required to overwrite the remote with the cleaned history.

```bash
# If you haven't linked to GitHub yet:
git remote add origin https://github.com/YOUR_USERNAME/YOUR_REPO_NAME.git

# Force push (overwrites remote history)
git push origin main --force

# Push all branches and tags (if any)
git push origin --force --all
git push origin --force --tags
```

> ⚠️ **Warning:** After force-pushing, anyone who had cloned the repo must
> delete their local copy and re-clone. Their local branches still contain
> the leaked secrets.

---

## ⚠️ Critical Post-Fix Actions (Real World)

Even after purging from git, you **must**:

1. **Rotate ALL exposed credentials immediately**
   - Generate new AWS keys in IAM Console → delete old ones
   - Regenerate API tokens in each service dashboard
   - Change database passwords
   - Rotate JWT secrets and invalidate existing sessions

2. **Check if secrets were already stolen**
   - Review AWS CloudTrail for unauthorized API calls
   - Check database access logs
   - Monitor for unusual activity

3. **Notify your team** — everyone must re-clone after force push

4. **If repo was public:** Assume secrets are compromised regardless of the fix.
   GitHub caches history and bots scrape public repos within seconds of a push.

---

## Quick Command Reference

```bash
# Find when .env was added
git log --all --full-history -- .env

# Remove from all history
git filter-branch --force --index-filter \
  'git rm --cached --ignore-unmatch .env' \
  --prune-empty --tag-name-filter cat -- --all

# Clean up
rm -rf .git/refs/original/
git reflog expire --expire=now --all
git gc --prune=now --aggressive

# Verify it's gone (empty output = good)
git log --all --full-history -- .env

# Force push
git push origin main --force
```
