# Github Action: Upstream Sync

## About

An action with forks in mind! Automatically sync a branch on your fork with the latest commits from the original repo. It should work to keep any two branches in sync. Keep all your things up to date!

## Support This Project

This project is not currently in active development, but code contributions are always welcome.

I encourage you to share projects using Fork Sync in the Users file. Open a PR against Users so others can check out your work!

> [USERS.md](/USERS.md)

If you would like to see more updates to this project, please consider taking a brief survey to help me understand your needs. Thank you, and happy coding.

> [Fork Sync - Developer Survey](https://forms.gle/pNGkmDWBidcu3BnPA)

## Intended Workflow

This action was made to keep **non-working** branches up to date with a remote repo. However, you should also be able to use it for branches under active development as long as the custom input supports your needs.

Configure the action with a branch on your `target` repo - the one you want to update - and a branch on your `upstream` repo - where the updates are coming from. This action checks out those branches on both repos, checks for any new commits (by hash comparison), and pulls those new commits from upstream to the target. Easy!

<img src="img/fork-sync-diagram.png" alt="sample git workflow">

## How to Use

[Add a workflow](https://docs.github.com/en/actions/quickstart#creating-your-first-workflow) to your repo that includes this action ([sample below](#sample-workflow)). Please note that scheduled workflows only run on the default branch of a repo.

### Input Variables

| Name                       |     Required?      | Default | Example                                  |
| -------------------------- | :----------------: | ------- | ---------------------------------------- |
| target_sync_branch         | :white_check_mark: |         | 'master', 'main', 'my-branch'            |
| target_repo_token          | :white_check_mark: |         | ${{ secrets.GITHUB_TOKEN }}              |
| upstream_repo_access_token |                    |         | ${{ secrets.NAME_OF_TOKEN }}             |
| upstream_sync_repo         | :white_check_mark: |         | 'aormsby/Fork-Sync-With-Upstream-action' |
| upstream_sync_branch       | :white_check_mark: |         | 'master', 'main', 'my-branch'            |
| test_mode                  |                    | false   | true / false                             |

**Always** set `target_repo_token` to `${{ secrets.GITHUB_TOKEN }}` so the action can push to your target repo.

> For more information on optional input variables, advanced configurations, and working with private repos, see [Wiki - Configuration](https://github.com/aormsby/Fork-Sync-With-Upstream-action/wiki/Configuration)

### Output Variables

| Name            | Output     | Description                                                                                                     |
| --------------- | ---------- | --------------------------------------------------------------------------------------------------------------- |
| has_new_commits | true/false | Outputs true if new commits were found in the remote repo, false if target repo already has the latest updates. |

If you need more output data, please open an issue to request it.

## Sample Workflow

```yaml
name: 'Upstream Sync'

on:
  schedule:
    - cron:  '0 7 * * 1,4'
    # scheduled at 07:00 every Monday and Thursday

  workflow_dispatch:  # click the button on Github repo!
    inputs:
      sync_test_mode: # Adds a boolean option that appears during manual workflow run for easy test mode config
        description: 'Fork Sync Test Mode'
        type: boolean
        default: false

jobs:
  sync_latest_from_upstream:
    runs-on: ubuntu-latest
    name: Sync latest commits from upstream repo

    steps:
    # REQUIRED step
    # Step 1: run a standard checkout action, provided by github
    - name: Checkout target repo
      uses: actions/checkout@v3
      with:
        # optional: set the branch to checkout,
        # sync action checks out your 'target_sync_branch' anyway
        ref:  my-branch
        # REQUIRED if your upstream repo is private (see wiki)
        persist-credentials: false

    # REQUIRED step
    # Step 2: run the sync action
    - name: Sync upstream changes
      id: sync
      uses: aormsby/Fork-Sync-With-Upstream-action@v3.4.1
      with:
        target_sync_branch: my-branch
        # REQUIRED 'target_repo_token' exactly like this!
        target_repo_token: ${{ secrets.GITHUB_TOKEN }}
        upstream_sync_branch: main
        upstream_sync_repo: aormsby/Fork-Sync-With-Upstream-action
        upstream_repo_access_token: ${{ secrets.UPSTREAM_REPO_SECRET }}

        # Set test_mode true during manual dispatch to run tests instead of the true action!!
        test_mode: ${{ inputs.sync_test_mode }}
      
    # Step 3: Display a sample message based on the sync output var 'has_new_commits'
    - name: New commits found
      if: steps.sync.outputs.has_new_commits == 'true'
      run: echo "New commits were found to sync."
    
    - name: No new commits
      if: steps.sync.outputs.has_new_commits == 'false'
      run: echo "There were no new commits."
      
    - name: Show value of 'has_new_commits'
      run: echo ${{ steps.sync.outputs.has_new_commits }}

```

---

## 推荐--Workflows--Sync.yml
```
name: RepoSync

on:
  schedule:
    - cron: "0 0 * * 1"  # 每周一 0 点运行
  workflow_dispatch:  # 手动触发工作流

jobs:
  sync_latest_from_upstream:
    name: 自动同步上游仓库
    runs-on: ubuntu-latest
    if: ${{ github.event.repository.fork }}  # 仅在 fork 仓库中运行
    env:
      UPSTREAM_REPO: github-username/repositories-name  # 在这里定义上游仓库，格式为“用户名/仓库名”
      UPSTREAM_REPO_BRANCH: main  # 定义上游仓库的分支
      TARGET_REPO_BRANCH: main  # 定义目标仓库的分支

    steps:
      # 检出当前仓库代码
      - uses: actions/checkout@v4

      # 同步上游仓库的更改
      - name: Sync upstream changes
        id: sync
        uses: yxdz2020/Fork-Sync-With-Upstream-action@v20250122
        with:
          upstream_sync_repo: ${{ env.UPSTREAM_REPO }} # 使用定义的上游仓库
          upstream_sync_branch: ${{ env.UPSTREAM_REPO_BRANCH }} # 使用定义的上游分支
          target_sync_branch: ${{ env.TARGET_REPO_BRANCH }} # 使用定义的目标分支
          target_repo_token: ${{ secrets.GITHUB_TOKEN }} # 自动生成，无需设置
          test_mode: false # 关闭测试模式，实际执行同步操作
        continue-on-error: true # 即使同步失败，继续执行后续步骤

      # 输出同步结果
      - name: 输出同步结果
        run: |
          if [ "${{ steps.sync.outputs.changes_pushed }}" = "true" ]; then
            echo "✅ 同步成功：上游仓库的最新更改已同步到目标仓库。"
          elif [ "${{ steps.sync.outputs.changes_pushed }}" = "false" ]; then
            echo "⚠️ 无需同步：没有检测到需要同步的更改。"
          else
            echo "❌ 同步失败：${{ steps.sync.outputs.error_message:-同步过程中出现了问题。}}"
          fi
```
---
