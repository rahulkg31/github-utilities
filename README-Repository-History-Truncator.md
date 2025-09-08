# GitHub Repository Truncator

A shell script to **truncate commits and code changes** from multiple branches of a Git repository **after a specific cutoff date** and force-push the cleaned history to GitHub. Useful for cleaning sensitive data or freezing repo history.



## Prerequisites

- `git` installed.
- Access to the GitHub repository.
- GitHub **Personal Access Token (PAT)** if using HTTPS.



## Usage

**Clone your repository locally**:

```bash
git clone https://github.com/username/repo.git
cd repo
```



**Create a file with branches** to process (one per line), e.g., `branches.txt`:

```shell
main
dev
test_branch
```



**Set GitHub credentials** as environment variables:

```shell
export GH_USER="your_username"
export GH_TOKEN="your_personal_access_token"
```



**Run the script** (embedded below):

```shell
#!/bin/bash

# Usage: ./truncate-branches.sh branches.txt 2025-01-11
# branches.txt = list of branches, one per line
# second arg = cutoff date (YYYY-MM-DD)

if [ $# -ne 2 ]; then
    echo "Usage: $0 <branches_file> <cutoff_date>"
    exit 1
fi

BRANCH_FILE="$1"
CUTOFF_DATE="$2"

# GitHub credentials from environment variables
GITHUB_USER="$GH_USER"
GITHUB_TOKEN="$GH_TOKEN"
REPO_NAME="repo.git"     
REPO_OWNER="username"      

# Construct remote URL with token
REMOTE_URL="https://${GITHUB_USER}:${GITHUB_TOKEN}@github.com/${REPO_OWNER}/${REPO_NAME}"

# Ensure we're in a git repo
if [ ! -d ".git" ]; then
    echo "Error: Must run inside a git repository."
    exit 1
fi

# Set remote URL to include credentials
git remote set-url origin "$REMOTE_URL"

while read -r branch; do
    echo "Processing branch: $branch"

    # Checkout branch
    git checkout "$branch" || { echo "Failed to checkout $branch"; continue; }

    # Get last commit before cutoff date
    COMMIT_HASH=$(git rev-list -1 --before="$CUTOFF_DATE" "$branch")
    if [ -z "$COMMIT_HASH" ]; then
        echo "No commit found before $CUTOFF_DATE for branch $branch"
        continue
    fi

    echo "Resetting $branch to commit $COMMIT_HASH"
    git reset --hard "$COMMIT_HASH"

    # Force push updated branch
    git push origin "$branch" --force
done < "$BRANCH_FILE"

echo "All branches processed."
```

