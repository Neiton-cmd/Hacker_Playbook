# --- Initial setup ---

git init                               # Initialize a new local Git repository
git remote add origin https://github.com/username/repo.git   # Connect local repo to GitHub
git clone https://github.com/username/repo.git               # Clone an existing repo from GitHub


# --- Branch management ---

git branch                             # List all local branches
git branch -a                          # List all branches (local + remote)
git checkout -b dev                    # Create and switch to a new branch 'dev'
git checkout dev                       # Switch to an existing branch
git push -u origin dev                 # Push 'dev' to GitHub and set tracking


# --- Working with changes ---

git status                             # Check which files are modified/untracked
git add .                              # Stage all changes for commit
git add file_name                      # Stage a specific file
git commit -m "Message"                # Commit staged changes with a message
git log --oneline                      # Show concise commit history


# --- Push & Pull ---

git push                               # Push current branch to GitHub
git push origin dev                    # Push to specific branch
git pull                               # Pull the latest changes from GitHub
git fetch                              # Download updates from remote without merging
git pull origin dev                    # Update local 'dev' from remote 'dev'


# --- Merging branches ---

git checkout dev                       # Switch to dev
git merge feature/login                # Merge 'feature/login' into dev
git push origin dev                    # Push merged changes to GitHub

git checkout main                      # Switch to main
git merge dev                          # Merge dev into main (usually after testing)
git push origin main                   # Push main to GitHub


# --- Stashing (temporary save) ---

git stash                              # Save uncommitted changes temporarily
git stash list                         # Show list of stashed changes
git stash apply                        # Reapply the last stash
git stash pop                          # Apply and remove last stash
git stash drop                         # Delete a stash


# --- Undo & recovery ---

git reflog                             # Show all recent HEAD actions (for recovery)
git checkout <commit_hash>             # Move to a specific commit
git reset --hard <commit_hash>         # Reset working directory to a specific commit
git revert <commit_hash>               # Create a new commit that undoes a specific commit


# --- Collaboration flow (recommended) ---

git checkout dev                       # Move to dev branch
git checkout -b feature/feature-name   # Create a new feature branch
git add .                              # Add changes
git commit -m "Added feature-name logic"
git push origin feature/feature-name   # Push feature branch
# Then create Pull Request on GitHub → feature → dev

git checkout dev                       # Go back to dev
git pull origin dev                    # Update dev with the latest code


# --- Housekeeping ---

git diff                               # Show unstaged changes
git diff --cached                      # Show staged (to be committed) changes
git clean -fd                          # Remove untracked files/folders (use carefully)
git rm file_name                       # Remove file from Git and disk
git mv old_name new_name               # Rename or move file and stage it


# --- Tags (for releases) ---

git tag -a v1.0 -m "First stable release"   # Create annotated tag
git push origin v1.0                        # Push tag to GitHub
git tag                                     # List all tags
