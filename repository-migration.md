# Repo Migration Guide (New Org, New Name, Full History + LFS)

Migrate a repository to a different GitHub org under a new name, preserving all commit history, branches, tags, and Git LFS objects.

## Prerequisites

- Admin/push access to both the source repo and the destination org
- Git LFS installed locally (if the source repo uses LFS)

Check if Git LFS is installed:
```bash
git lfs version
```

If not installed (Ubuntu/Debian):
```bash
sudo apt update
sudo apt install git-lfs
git lfs install
```

Check if the source repo actually uses LFS (skip LFS steps below if this is empty):
```bash
cat .gitattributes | grep lfs
```

## Steps

### 1. Create the destination repo

In the new org, create a new **empty** repo with your desired name. Do **not** initialize it with a README, license, or `.gitignore` — the mirror push requires an empty repo to avoid conflicting refs.

### 2. Mirror-clone the source repo

```bash
git clone --mirror https://github.com/OLD-ORG/old-repo-name.git
cd old-repo-name.git
```

This captures all branches, tags, and refs — not just the default branch.

### 3. Fetch all LFS objects (skip if repo has no LFS content)

```bash
git lfs fetch --all origin
```

The `--all` flag pulls LFS objects referenced across **every branch and tag**, not just the currently checked-out one.

### 4. Point the remote at the new destination

```bash
git remote set-url origin https://github.com/NEW-ORG/new-repo-name.git
```

This is where the rename happens — the new URL/repo name is whatever you created in step 1.

### 5. Push all git history

```bash
git push --mirror origin
```

Pushes every commit, branch, and tag exactly as-is.

### 6. Push all LFS objects (skip if repo has no LFS content)

```bash
git lfs push --all origin
```

### 7. Clean up the local mirror clone

```bash
cd ..
rm -rf old-repo-name.git
```

### 8. Update collaborators' local clones

Anyone with an existing local clone of the old repo needs to update their remote:

```bash
git remote set-url origin https://github.com/NEW-ORG/new-repo-name.git
```

## Verification

Clone the new repo normally (not as a mirror) and check:

```bash
git clone https://github.com/NEW-ORG/new-repo-name.git
cd new-repo-name

git lfs ls-files            # confirms LFS files are tracked
git log --oneline | wc -l   # compare commit count to the old repo
git branch -a                # confirm all branches came across
git tag                      # confirm all tags came across
```

