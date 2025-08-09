# GitHub Project Automation Playbook

This is an **internal guide** for setting up GitHub Project Boards with automated issue tracking, milestones, and labels.
Follow this step-by-step to organize issues and workflows for any new project.

---

## 1. Prerequisites

- **GitHub account** with a repository ready to organize.
- **GitHub CLI** (`gh`) installed.
  ```bash
  sudo apt update && sudo apt install gh -y
  ```
- **Authenticate** the CLI with required scopes:
  ```bash
  gh auth login
  gh auth refresh -s project,read:project,repo
  ```
- Basic knowledge of your local terminal (Linux/MacOS/WSL).

---

## 2. Initial Repo Setup

1. Clone your repository locally:
   ```bash
   gh repo clone <OWNER>/<REPO>
   cd <REPO>
   ```

2. Create a `scripts/` folder for automation scripts:
   ```bash
   mkdir -p scripts
   ```

3. Keep automation scripts inside `scripts/` and name them clearly:
   - `create_<version>_issues.sh`
   - `setup_milestone.sh`
   - `setup_project_board.sh`

---

## 3. Create a Milestone

**One-liner CLI:**
```bash
gh api repos/{owner}/{repo}/milestones \
  -f title='v0.3.1' \
  -f description='Post–Safe Area Fix milestone' \
  -f due_on='2025-08-15T23:59:00Z'
```

**Reusable env var version:**
```bash
TITLE="v0.3.1" LABEL="v0.3.1" DUE="2025-08-15T23:59:00Z" ./scripts/setup_milestone.sh
```

Milestones help track versions and can be linked to labels for automation.

---

## 4. Create & Apply Labels

Labels are **not** the same as issue titles — automation relies on labels.

Create a label for your milestone:
```bash
gh label create v0.3.1 --color "#1D76DB" --description "Issues for milestone v0.3.1"
```

Apply it via:
- **CLI**:
  ```bash
  gh issue edit <ISSUE_NUMBER> --add-label v0.3.1
  ```
- **UI**: Open the issue → Labels → Select `v0.3.1`.

---

## 5. Create Issues in Bulk

Create a script `scripts/create_v031_issues.sh`:
```bash
#!/usr/bin/env bash
LABEL="v0.3.1"
gh issue create --title "Fix Safe Area top color on Folders screen" --label "$LABEL" --body "Ensure Safe Area matches design when returning from SetDetails."
# Repeat for more issues...
```

Run it:
```bash
chmod +x scripts/create_v031_issues.sh
./scripts/create_v031_issues.sh
```

---

## 6. Create the Project Board

Create a **Projects (v2)** board:
```bash
gh project create --title "StudyBuddy v0.3.1" --owner <OWNER>
```

Add description:
```bash
gh project edit <PROJECT_NUMBER> --description "Post–Safe Area Fix milestone board"
```

---

## 7. Automation Workflow

Inside `.github/workflows/auto_add_to_project.yml`:
```yaml
name: Auto-add v0.3.1 Issues to Project

on:
  issues:
    types: [opened, labeled]

jobs:
  add-to-project:
    if: >
      contains(github.event.issue.labels.*.name, 'v0.3.1') ||
      github.event.label.name == 'v0.3.1'
    runs-on: ubuntu-latest
    permissions:
      issues: write
      contents: read
      projects: write

    steps:
      - name: Get Project ID
        id: get_project
        run: |
          OWNER_LOGIN="${{ github.repository_owner }}"
          PROJECT_TITLE="StudyBuddy v0.3.1"
          PROJECT_ID=$(gh api graphql -f login="$OWNER_LOGIN" -f query='
            query($login:String!) {
              user(login:$login) {
                projectsV2(first:100) {
                  nodes { id title number }
                }
              }
            }' --jq ".data.user.projectsV2.nodes[] | select(.title==\"$PROJECT_TITLE\") | .id")
          if [ -z "$PROJECT_ID" ]; then
            echo "Project not found"
            exit 1
          fi
          echo "project_id=$PROJECT_ID" >> $GITHUB_OUTPUT
        env:
          GH_TOKEN: ${{ secrets.GITHUB_TOKEN }}

      - name: Add Issue to Project
        run: |
          ISSUE_NODE_ID=$(gh api graphql -f owner="${{ github.repository_owner }}" -f repo="$(basename ${{ github.repository }})" -F number="${{ github.event.issue.number }}" -f query='
            query($owner:String!, $repo:String!, $number:Int!) {
              repository(owner:$owner, name:$repo) {
                issue(number:$number) { id }
              }
            }' --jq '.data.repository.issue.id')

          gh api graphql -F projectId="${{ steps.get_project.outputs.project_id }}" -F contentId="$ISSUE_NODE_ID" -f query='
            mutation($projectId:ID!, $contentId:ID!) {
              addProjectV2ItemById(input:{projectId:$projectId, contentId:$contentId}) {
                item { id }
              }
            }'
        env:
          GH_TOKEN: ${{ secrets.GITHUB_TOKEN }}
```

---

## 8. Test the Automation

1. Create a throwaway issue with the `v0.3.1` label.
2. Wait ~30 seconds.
3. Check the project board — it should appear in **To Do**.
4. Delete the test issue.

---

## 9. Maintenance

For new versions:
1. Create a new milestone (`v0.3.2`).
2. Create the matching label.
3. Create new project board.
4. Duplicate the automation workflow and update the label name.

---

## Notes

- Always use **labels** to trigger automation — titles don’t count.
- You can create issues directly in the board, but you must still add the correct label.
- Keep scripts modular so you can reuse them across projects.
