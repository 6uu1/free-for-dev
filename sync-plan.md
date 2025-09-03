# Free-for-Dev 仓库定时同步方案

## 概述

本文档详细描述了如何为 free-for-dev 仓库设置从上游 ripienaar/free-for-dev 的定时同步方案。该方案利用 GitHub Actions 实现自动化同步，确保仓库内容保持最新。

## 仓库分析

### 当前仓库结构
- **主要文件**: README.md (包含大量免费开发者资源列表)
- **支持文件**: 
  - CNAME
  - index.html
  - logo.webp
  - .github/PULL_REQUEST_TEMPLATE.md

### 同步需求
1. 定期从上游仓库 (ripienaar/free-for-dev) 获取最新更新
2. 保持本地仓库的特定配置（如 CNAME）
3. 处理可能的合并冲突
4. 提供同步状态通知

## 技术方案设计

### 方案选择

经过分析，我们选择使用 **GitHub Actions + Git 命令** 的同步方式，原因如下：

1. **GitHub Actions** 提供了强大的自动化能力
2. **Git 命令** 直接操作，灵活度高
3. 与 GitHub 生态系统深度集成
4. 可以轻松设置定时触发
5. 支持错误处理和通知

### 同步策略

#### 1. 同步频率
- **主要同步**: 每日一次 (UTC 02:00)
- **紧急同步**: 可手动触发

#### 2. 同步内容
- **必须同步**: README.md (核心内容)
- **可选同步**: .github 目录（工作流、模板等）
- **不同步**: CNAME (保留本地配置)

#### 3. 冲突处理策略
- **自动合并**: 对于 README.md 的内容更新
- **手动解决**: 对于复杂的合并冲突，创建 issue 并通知维护者
- **保留本地**: 对于 CNAME 等本地特定文件

## 实现方案

### GitHub Actions 工作流配置

创建 `.github/workflows/sync-upstream.yml` 文件：

```yaml
name: Sync Upstream Repository

on:
  schedule:
    # 每日 UTC 02:00 执行 (北京时间 10:00)
    - cron: '0 2 * * *'
  workflow_dispatch: # 允许手动触发

jobs:
  sync:
    runs-on: ubuntu-latest
    permissions:
      contents: write
      pull-requests: write
    
    steps:
      - name: Checkout current repository
        uses: actions/checkout@v5
        with:
          token: ${{ secrets.GITHUB_TOKEN }}
          fetch-depth: 0 # 获取完整历史记录

      - name: Configure Git
        run: |
          git config --global user.name "github-actions[bot]"
          git config --global user.email "41898282+github-actions[bot]@users.noreply.github.com"

      - name: Add upstream remote
        run: |
          git remote add upstream https://github.com/ripienaar/free-for-dev.git

      - name: Fetch upstream changes
        run: |
          git fetch upstream

      - name: Check for changes
        id: check-changes
        run: |
          # 检查上游是否有新提交
          LOCAL=$(git rev-parse HEAD)
          REMOTE=$(git rev-parse upstream/main)
          if [ "$LOCAL" != "$REMOTE" ]; then
            echo "changes=true" >> $GITHUB_OUTPUT
            echo "upstream_commit=$REMOTE" >> $GITHUB_OUTPUT
          else
            echo "changes=false" >> $GITHUB_OUTPUT
          fi

      - name: Create sync branch if changes exist
        if: steps.check-changes.outputs.changes == 'true'
        run: |
          git checkout -b sync/upstream-${{ steps.check-changes.outputs.upstream_commit }}

      - name: Merge upstream changes
        if: steps.check-changes.outputs.changes == 'true'
        run: |
          # 尝试合并上游变更
          if ! git merge upstream/main --no-commit; then
            echo "merge_conflict=true" >> $GITHUB_ENV
            # 保留冲突状态，后续处理
          else
            echo "merge_conflict=false" >> $GITHUB_ENV
          fi

      - name: Handle CNAME file (preserve local)
        if: steps.check-changes.outputs.changes == 'true' && env.merge_conflict == 'false'
        run: |
          # 确保本地 CNAME 文件不被覆盖
          git checkout HEAD -- CNAME
          git add CNAME

      - name: Commit changes
        if: steps.check-changes.outputs.changes == 'true' && env.merge_conflict == 'false'
        run: |
          git commit -m "Sync with upstream ripienaar/free-for-dev@${{ steps.check-changes.outputs.upstream_commit }}

          - Upstream commit: ${{ steps.check-changes.outputs.upstream_commit }}
          - Sync date: $(date -u)"

      - name: Push changes
        if: steps.check-changes.outputs.changes == 'true' && env.merge_conflict == 'false'
        run: |
          git push origin sync/upstream-${{ steps.check-changes.outputs.upstream_commit }}

      - name: Create Pull Request
        if: steps.check-changes.outputs.changes == 'true' && env.merge_conflict == 'false'
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
        run: |
          # 使用 GitHub CLI 创建 PR
          gh pr create \
            --title "Sync with upstream ripienaar/free-for-dev@${{ steps.check-changes.outputs.upstream_commit }}" \
            --body "This PR syncs changes from upstream repository [ripienaar/free-for-dev@${{ steps.check-changes.outputs.upstream_commit }}](https://github.com/ripienaar/free-for-dev/commit/${{ steps.check-changes.outputs.upstream_commit }}).

            ## Changes
            - Automated sync from upstream repository
            - Local CNAME file preserved

            ## Review
            Please review the changes and merge if everything looks correct." \
            --label "automation" \
            --label "sync"

      - name: Handle merge conflict
        if: steps.check-changes.outputs.changes == 'true' && env.merge_conflict == 'true'
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
        run: |
          # 创建 issue 通知维护者
          gh issue create \
            --title "Merge conflict detected during upstream sync" \
            --body "## Merge Conflict Alert

            A merge conflict was detected while trying to sync with upstream repository ripienaar/free-for-dev.

            ### Details
            - **Upstream commit**: ${{ steps.check-changes.outputs.upstream_commit }}
            - **Sync date**: $(date -u)
            - **Conflict branch**: sync/upstream-${{ steps.check-changes.outputs.upstream_commit }}

            ### Required Action
            Manual intervention is required to resolve the merge conflict. Please:

            1. Checkout the conflict branch: \`git checkout sync/upstream-${{ steps.check-changes.outputs.upstream_commit }}\`
            2. Resolve conflicts manually
            3. Commit and push the changes
            4. Create a pull request or merge directly

            ### Files with conflicts
            \`\`\`
            $(git diff --name-only --diff-filter=U)
            \`\`\`

            This issue was automatically created by the GitHub Actions sync workflow." \
            --label "bug" \
            --label "merge-conflict" \
            --label "needs-attention"

      - name: Notify no changes
        if: steps.check-changes.outputs.changes == 'false'
        run: |
          echo "No changes detected in upstream repository"

      - name: Notify on failure
        if: failure()
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
        run: |
          gh issue create \
            --title "Sync workflow failed" \
            --body "## Workflow Failure Alert

            The upstream sync workflow has failed.

            ### Details
            - **Failure time**: $(date -u)
            - **Workflow run**: ${{ github.server_url }}/${{ github.repository }}/actions/runs/${{ github.run_id }}

            ### Required Action
            Please check the workflow logs and investigate the failure." \
            --label "bug" \
            --label "automation" \
            --label "needs-attention"
```

### 备选方案：使用 GitHub Script

为了更灵活地处理 GitHub API 调用，我们可以创建一个使用 GitHub Script 的版本：

```yaml
name: Sync Upstream Repository (Advanced)

on:
  schedule:
    - cron: '0 2 * * *'
  workflow_dispatch:

jobs:
  sync:
    runs-on: ubuntu-latest
    permissions:
      contents: write
      pull-requests: write
    
    steps:
      - name: Checkout repository
        uses: actions/checkout@v5
        with:
          token: ${{ secrets.GITHUB_TOKEN }}
          fetch-depth: 0

      - name: Configure Git
        run: |
          git config --global user.name "github-actions[bot]"
          git config --global user.email "41898282+github-actions[bot]@users.noreply.github.com"

      - name: Sync with upstream
        uses: actions/github-script@v7
        env:
          UPSTREAM_REPO: ripienaar/free-for-dev
        with:
          script: |
            const { execSync } = require('child_process');
            const upstreamRepo = process.env.UPSTREAM_REPO;
            
            try {
              // 添加上游远程仓库
              execSync('git remote add upstream https://github.com/' + upstreamRepo + '.git', { stdio: 'inherit' });
              
              // 获取上游变更
              execSync('git fetch upstream', { stdio: 'inherit' });
              
              // 检查是否有变更
              const localCommit = execSync('git rev-parse HEAD', { encoding: 'utf8' }).trim();
              const upstreamCommit = execSync('git rev-parse upstream/main', { encoding: 'utf8' }).trim();
              
              if (localCommit === upstreamCommit) {
                console.log('No changes detected in upstream repository');
                return;
              }
              
              // 创建同步分支
              const branchName = `sync/upstream-${upstreamCommit.substring(0, 7)}`;
              execSync(`git checkout -b ${branchName}`, { stdio: 'inherit' });
              
              // 尝试合并
              try {
                execSync(`git merge upstream/main --no-commit`, { stdio: 'inherit' });
                
                // 保留本地 CNAME 文件
                execSync('git checkout HEAD -- CNAME', { stdio: 'inherit' });
                execSync('git add CNAME', { stdio: 'inherit' });
                
                // 提交变更
                execSync(`git commit -m "Sync with upstream ${upstreamRepo}@${upstreamCommit}\\n\\n- Upstream commit: ${upstreamCommit}\\n- Sync date: ${new Date().toISOString()}"`, { stdio: 'inherit' });
                
                // 推送分支
                execSync(`git push origin ${branchName}`, { stdio: 'inherit' });
                
                // 创建 PR
                await github.rest.pulls.create({
                  owner: context.repo.owner,
                  repo: context.repo.repo,
                  title: `Sync with upstream ${upstreamRepo}@${upstreamCommit.substring(0, 7)}`,
                  head: branchName,
                  base: 'main',
                  body: `This PR syncs changes from upstream repository [${upstreamRepo}@${upstreamCommit}](https://github.com/${upstreamRepo}/commit/${upstreamCommit}).\n\n## Changes\n- Automated sync from upstream repository\n- Local CNAME file preserved\n\n## Review\nPlease review the changes and merge if everything looks correct.`,
                  labels: ['automation', 'sync']
                });
                
              } catch (mergeError) {
                // 处理合并冲突
                const conflictedFiles = execSync('git diff --name-only --diff-filter=U', { encoding: 'utf8' }).trim();
                
                await github.rest.issues.create({
                  owner: context.repo.owner,
                  repo: context.repo.repo,
                  title: 'Merge conflict detected during upstream sync',
                  body: `## Merge Conflict Alert\n\nA merge conflict was detected while trying to sync with upstream repository ${upstreamRepo}.\n\n### Details\n- **Upstream commit**: ${upstreamCommit}\n- **Sync date**: ${new Date().toISOString()}\n- **Conflict branch**: ${branchName}\n\n### Required Action\nManual intervention is required to resolve the merge conflict. Please:\n\n1. Checkout the conflict branch: \`git checkout ${branchName}\`\n2. Resolve conflicts manually\n3. Commit and push the changes\n4. Create a pull request or merge directly\n\n### Files with conflicts\n\`\`\`\n${conflictedFiles}\n\`\`\`\n\nThis issue was automatically created by the GitHub Actions sync workflow.`,
                  labels: ['bug', 'merge-conflict', 'needs-attention']
                });
              }
              
            } catch (error) {
              // 处理一般错误
              await github.rest.issues.create({
                owner: context.repo.owner,
                repo: context.repo.repo,
                title: 'Sync workflow failed',
                body: `## Workflow Failure Alert\n\nThe upstream sync workflow has failed.\n\n### Details\n- **Failure time**: ${new Date().toISOString()}\n- **Workflow run**: [${context.serverUrl}/${context.repo.owner}/${context.repo.repo}/actions/runs/${context.runId}](${context.serverUrl}/${context.repo.owner}/${context.repo.repo}/actions/runs/${context.runId})\n\n### Required Action\nPlease check the workflow logs and investigate the failure.\n\n### Error Details\n\`\`\`\n${error.message}\n\`\`\``,
                labels: ['bug', 'automation', 'needs-attention']
              });
              
              throw error;
            }
```

## 配置要求

### 1. GitHub Secrets
无需额外配置，使用默认的 `${{ secrets.GITHUB_TOKEN }}` 即可。

### 2. 权限设置
工作流需要以下权限：
- `contents: write` - 推送代码和创建分支
- `pull-requests: write` - 创建拉取请求

### 3. 分支保护
建议为主分支设置分支保护：
- 要求 PR 审核
- 要求状态检查通过
- 禁止强制推送

## 监控和通知

### 1. 成功同步
- 创建 PR 并标记为 "automation" 和 "sync"
- PR 包含上游提交信息和同步详情

### 2. 合并冲突
- 自动创建 issue，标记为 "merge-conflict" 和 "needs-attention"
- 提供冲突解决步骤指导

### 3. 同步失败
- 自动创建 issue，标记为 "bug" 和 "needs-attention"
- 包含工作流运行链接和错误详情

### 4. 无变更
- 工作流正常完成，不创建 PR 或 issue

## 维护和故障排除

### 1. 手动触发同步
- 访问仓库的 Actions 页面
- 选择 "Sync Upstream Repository" 工作流
- 点击 "Run workflow"

### 2. 处理合并冲突
1. 查看 issue 中的冲突详情
2. 本地检出冲突分支
3. 手动解决冲突
4. 提交并推送变更
5. 创建 PR 或直接合并

### 3. 工作流调试
- 检查工作流运行日志
- 验证 Git 配置和远程仓库设置
- 确认 GitHub Token 权限

### 4. 性能优化
- 使用 `fetch-depth: 0` 获取完整历史
- 仅在有变更时执行后续步骤
- 并行执行独立任务

## 扩展性考虑

### 1. 多仓库同步
可以扩展此方案以同步多个上游仓库：
```yaml
strategy:
  matrix:
    upstream:
      - owner: ripienaar
        repo: free-for-dev
      - owner: other
        repo: repo-name
```

### 2. 自定义同步规则
可以添加更复杂的同步规则，例如：
- 忽略特定文件或目录
- 自定义合并策略
- 条件性同步（基于文件内容或提交消息）

### 3. 集成其他服务
可以集成：
- Slack/Teams 通知
- 邮件通知
- 外部监控系统

## 安全考虑

1. **Token 权限最小化**: 仅授予必要的权限
2. **分支保护**: 保护主分支免受直接推送
3. **审计日志**: 记录所有同步操作
4. **访问控制**: 限制工作流修改权限

## 总结

本同步方案提供了从 ripienaar/free-for-dev 到本地仓库的自动化同步功能。通过 GitHub Actions 实现，具有以下特点：

- **自动化**: 定时同步，无需人工干预
- **可靠性**: 完善的错误处理和通知机制
- **灵活性**: 支持手动触发和自定义配置
- **透明性**: 详细的日志和 PR 跟踪
- **安全性**: 最小权限原则和分支保护

该方案可以有效保持仓库内容与上游同步，同时保留本地特定配置，减少维护工作量。