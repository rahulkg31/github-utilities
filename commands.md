List all remote branches

git fetch --prune origin && \
{ echo "branch"; git branch -r --format='%(refname:short)' | sed 's#^origin/##'; } > branches.csv
