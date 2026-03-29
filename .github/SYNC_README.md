# Free-for-Dev 同步工作流使用指南

## 概述

本目录包含了用于自动同步上游 ripienaar/free-for-dev 仓库的 GitHub Actions 工作流。这些工作流会定期检查上游仓库的变更，并自动创建拉取请求以保持本地仓库的更新。

## 工作流文件

### 1. 基础版本：`sync-upstream.yml`

这是基础的同步工作流，使用标准的 GitHub Actions 步骤和 Git 命令。

**特点：**
- 每日 UTC 02:00 自动执行（北京时间 10:00）
- 支持手动触发
- 自动检测上游变更
- 保留本地 CNAME 文件
- 自动创建 PR 或处理合并冲突
- 完善的错误处理和通知机制

### 2. 高级版本：`sync-upstream-advanced.yml`

这是使用 GitHub Script 的高级版本，提供了更灵活的 GitHub API 集成。

**特点：**
- 所有基础版本的功能
- 使用 JavaScript 编写同步逻辑
- 更精细的错误处理
- 更灵活的 API 调用

## 使用方法

### 自动同步

工作流已配置为每日自动运行，无需额外操作。

### 手动触发同步

1. 访问仓库的 **Actions** 标签页
2. 在左侧选择 **Sync Upstream Repository** 或 **Sync Upstream Repository (Advanced)**
3. 点击右侧的 **Run workflow** 按钮
4. 选择要使用的分支（默认为 main）
5. 点击 **Run workflow** 确认

### 监控同步状态

1. **成功的同步**：会自动创建一个标记为 "automation" 和 "sync" 的 PR
2. **合并冲突**：会自动创建一个标记为 "merge-conflict" 的 issue
3. **同步失败**：会自动创建一个标记为 "bug" 的 issue

## 处理合并冲突

当检测到合并冲突时，工作流会自动创建一个 issue，包含以下信息：

- 冲突的上游提交 ID
- 冲突分支名称
- 冲突文件列表
- 解决步骤指南

### 解决步骤：

1. 查看 issue 中的详细信息
2. 在本地克隆仓库
3. 切换到冲突分支：
   ```bash
   git checkout sync/upstream-[commit-hash]
   ```
4. 手动解决冲突
5. 提交变更：
   ```bash
   git add .
   git commit -m "Resolve merge conflicts"
   ```
6. 推送分支：
   ```bash
   git push origin sync/upstream-[commit-hash]
   ```
7. 创建 PR 或直接合并到主分支

## 自定义配置

### 修改同步频率

编辑工作流文件中的 `schedule` 部分：

```yaml
on:
  schedule:
    - cron: '0 2 * * *'  # 修改此处的 cron 表达式
```

Cron 表达式格式：`分 时 日 月 周`

示例：
- 每小时：`0 * * * *`
- 每周日午夜：`0 0 * * 0`
- 每工作日上午9点（UTC）：`0 9 * * 1-5`

### 修改上游仓库

编辑工作流文件中的上游仓库 URL：

```bash
git remote add upstream https://github.com/[owner]/[repo].git
```

### 修改保留文件

在基础版本中，编辑以下步骤：

```yaml
- name: Handle CNAME file (preserve local)
  if: steps.check-changes.outputs.changes == 'true' && env.merge_conflict == 'false'
  run: |
    # 确保本地 CNAME 文件不被覆盖
    git checkout HEAD -- CNAME
    git add CNAME
    # 可以添加更多需要保留的文件
    git checkout HEAD -- [other-file]
    git add [other-file]
```

## 故障排除

### 工作流失败

1. 检查 Actions 页面上的工作流日志
2. 查看自动创建的 issue 中的错误详情
3. 常见问题：
   - 权限不足：确保 GITHUB_TOKEN 有足够的权限
   - 网络问题：检查是否能访问上游仓库
   - Git 配置问题：验证 Git 用户配置

### PR 创建失败

1. 检查分支是否成功推送
2. 确认没有现有的 PR 针对同一分支
3. 验证 GitHub Token 的权限

### 手动修复同步

如果自动同步完全失败，可以手动执行以下步骤：

1. 添加上游远程仓库：
   ```bash
   git remote add upstream https://github.com/ripienaar/free-for-dev.git
   ```

2. 获取上游变更：
   ```bash
   git fetch upstream
   ```

3. 创建同步分支：
   ```bash
   git checkout -b sync/manual-$(date +%Y%m%d)
   ```

4. 合并上游变更：
   ```bash
   git merge upstream/main
   ```

5. 解决冲突（如有）
6. 提交并推送：
   ```bash
   git commit -m "Manual sync with upstream"
   git push origin sync/manual-$(date +%Y%m%d)
   ```

7. 创建 PR

## 最佳实践

1. **定期监控**：定期检查同步状态和自动创建的 PR/issue
2. **及时处理冲突**：合并冲突应尽快处理，避免积压
3. **备份重要更改**：在处理冲突前备份本地重要更改
4. **测试工作流**：在修改工作流配置后，先手动触发测试
5. **保持更新**：定期更新 GitHub Actions 版本以获取最新功能和安全修复

## 安全考虑

- 工作流使用默认的 GITHUB_TOKEN，具有最小必要权限
- 所有操作都有详细的日志记录
- 敏感信息不会记录在日志中
- 建议为主分支启用分支保护

## 支持与反馈

如果遇到问题或有改进建议，请：

1. 检查现有的 issue
2. 创建新的 issue，详细描述问题
3. 提供工作流运行日志和错误信息

---

*此文档由 GitHub Actions 同步工作流自动维护。*