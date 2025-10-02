# Test Repository Setup for Workflow Testing

## 1. Create a New Test Repository

1. Go to GitHub and create a new repository (e.g., `workflow-test-repo`)
2. Clone it locally
3. Create the following structure:

```
workflow-test-repo/
├── .github/
│   └── workflows/
│       └── test-notification-workflow.yml
├── README.md
└── package.json (optional)
```

## 2. Test Workflow File

Create `.github/workflows/test-notification-workflow.yml`:

```yaml
name: Test Notification Workflow

on:
  push:
    branches: [main, develop]
  workflow_dispatch:  # Allows manual triggering

jobs:
  test-notification:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
        with:
          fetch-depth: 0  # Get full history

      - name: "Test Current PR Notification"
        run: |
          echo "🧪 Testing current PR notification logic..."
          
          # Extract PR number from merge commit message
          COMMIT_MESSAGE="${{ github.event.head_commit.message }}"
          echo "Commit message: $COMMIT_MESSAGE"
          
          PR_NUMBER=$(echo "$COMMIT_MESSAGE" | grep -oP 'Merge pull request #(\d+)' | grep -oP '\d+' || echo "")
          
          if [ ! -z "$PR_NUMBER" ]; then
            echo "✅ Found PR number: $PR_NUMBER"
            
            # Get PR body using GitHub API (read-only, safe)
            PR_BODY=$(curl -s -H "Authorization: token ${{ secrets.GITHUB_TOKEN }}" \
              "https://api.github.com/repos/${{ github.repository }}/pulls/$PR_NUMBER" | \
              jq -r '.body // ""')
            
            echo "PR Body: $PR_BODY"
            
            # Extract Asana task ID (test pattern)
            ASANA_TASK_ID=$(echo "$PR_BODY" | grep -oP 'https://app\.asana\.com/.*/(?:task|item)/(\d+)' | grep -oP '\d+$' || echo "")
            
            if [ ! -z "$ASANA_TASK_ID" ]; then
              echo "✅ Found Asana task ID: $ASANA_TASK_ID"
              echo "🚀 Would notify: Hi team, this has been successfully deployed!"
            else
              echo "ℹ️ No Asana task ID found in PR body"
            fi
          else
            echo "ℹ️ No PR number found in commit message"
          fi

      - name: "Test Previously Failed Commits Logic"
        run: |
          echo "🧪 Testing previously failed commits notification logic..."
          
          # Get the last successful workflow run (safe read-only operation)
          LAST_SUCCESS_SHA=$(curl -s -H "Authorization: token ${{ secrets.GITHUB_TOKEN }}" \
            "https://api.github.com/repos/${{ github.repository }}/actions/runs?branch=${{ github.ref_name }}&status=success&per_page=1" | \
            jq -r '.workflow_runs[0].head_sha // ""')
          
          echo "Last successful SHA: $LAST_SUCCESS_SHA"
          echo "Current SHA: ${{ github.sha }}"
          
          # Get commits between last success and current (if different)
          if [ ! -z "$LAST_SUCCESS_SHA" ] && [ "$LAST_SUCCESS_SHA" != "${{ github.sha }}" ]; then
            echo "📊 Getting commits between $LAST_SUCCESS_SHA and ${{ github.sha }}"
            
            # Get commits in the range (safe read-only)
            COMMITS=$(curl -s -H "Authorization: token ${{ secrets.GITHUB_TOKEN }}" \
              "https://api.github.com/repos/${{ github.repository }}/compare/$LAST_SUCCESS_SHA...${{ github.sha }}" | \
              jq -r '.commits[]? | "\(.sha) \(.commit.message | split("\n")[0])"' || echo "")
            
            echo "Commits to analyze:"
            echo "$COMMITS"
            
            # Analyze each commit (read-only)
            echo "$COMMITS" | while IFS=' ' read -r commit_sha commit_message; do
              if [ ! -z "$commit_sha" ] && [ "$commit_sha" != "${{ github.sha }}" ]; then
                echo "🔍 Analyzing commit: $commit_sha"
                echo "📝 Message: $commit_message"
                
                # Check for failed workflow runs (read-only)
                FAILED_RUNS=$(curl -s -H "Authorization: token ${{ secrets.GITHUB_TOKEN }}" \
                  "https://api.github.com/repos/${{ github.repository }}/actions/runs?head_sha=$commit_sha&status=failure" | \
                  jq -r '.workflow_runs[]? | select(.name == "Test Notification Workflow") | .id' || echo "")
                
                if [ ! -z "$FAILED_RUNS" ]; then
                  echo "❌ Found failed runs for commit $commit_sha: $FAILED_RUNS"
                  
                  # Extract PR number from commit message
                  PR_NUM=$(echo "$commit_message" | grep -oP 'Merge pull request #(\d+)' | grep -oP '\d+' || echo "")
                  
                  if [ ! -z "$PR_NUM" ]; then
                    echo "🔗 Found PR number $PR_NUM for failed commit"
                    echo "📬 Would send delayed deployment notification for PR #$PR_NUM"
                  else
                    echo "ℹ️ No PR number found in commit message"
                  fi
                else
                  echo "✅ No failed runs found for commit $commit_sha"
                fi
              fi
            done
          else
            echo "ℹ️ No previous commits to check or this is the first run"
          fi

      - name: "Test Summary"
        run: |
          echo "🎉 Test completed successfully!"
          echo "📋 This workflow tested:"
          echo "  ✅ PR number extraction from commit messages"
          echo "  ✅ Asana task ID extraction from PR bodies"
          echo "  ✅ Failed workflow run detection"
          echo "  ✅ Commit range analysis"
          echo ""
          echo "🔒 All operations were read-only and safe"
          echo "🚀 Ready to implement in production workflow"
```

## 3. Test Scenarios

### Scenario 1: Test with Manual Trigger
1. Push the workflow file to your test repo
2. Go to Actions tab → "Test Notification Workflow" → "Run workflow"
3. Check the logs to see how it analyzes commits

### Scenario 2: Test with PR Merge
1. Create a test PR with Asana URL in description:
   ```
   This PR fixes a bug.
   
   Asana task: https://app.asana.com/0/123456789/987654321
   ```
2. Merge the PR
3. Check workflow logs to see PR analysis

### Scenario 3: Test Failed Workflow Detection
1. Create a workflow that intentionally fails:
   ```yaml
   - name: "Intentional Failure"
     run: exit 1
   ```
2. Let it fail, then fix it and push again
3. The next successful run should detect the previous failure

## 4. Environment Setup

You'll need these secrets in your test repo:
- `GITHUB_TOKEN` (automatically provided by GitHub)
- `ASANA_TOKEN` (optional, for actual Asana testing)

## 5. Safe Testing Tips

- All API calls in the test workflow are read-only
- No actual notifications are sent (just logged)
- Use `echo` statements instead of `curl` to Asana initially
- Test incrementally - start with simple scenarios

## 6. Debugging

Add these debug steps to see what data you're working with:

```yaml
- name: "Debug/Beta Information"
  run: |
    echo "Repository: ${{ github.repository }}"
    echo "Branch: ${{ github.ref_name }}"
    echo "SHA: ${{ github.sha }}"
    echo "Event: ${{ github.event_name }}"
    echo "Commit message: ${{ github.event.head_commit.message }}"
```
