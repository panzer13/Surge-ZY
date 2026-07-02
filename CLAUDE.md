# Surge-ZY 项目说明

## 仓库结构
- `Conf/Spec/` — Surge 配置文件（主要编辑 `Surge-EN.conf`）
- `Rules/` — 分流规则列表（自动生成，不手动编辑）
- `Rules/Manual/` — 手动维护的规则排除列表

## 上游仓库
- 上游：`https://github.com/Rabbit-Spec/Surge.git`（远程名 `upstream`，分支 `Master`）
- 本仓库：`panzer13/Surge-ZY`，主分支 `Master`

## 本地个人定制（Surge-EN.conf）
以下改动是个人定制，合并上游时必须保留：

1. **注释掉 `✈️ 我的节点` 行**（第80行附近）：
   ```
   # > ✈️ 我的节点 = select, policy-path=你的订阅地址, ...
   ```

2. **文件末尾的 SSID 配置**：
   ```
   [SSID Setting]
   ASUS suspend=true
   ```

## 同步上游流程
```bash
# 拉取上游最新
git fetch upstream Master

# 查看上游有多少新提交
git rev-list --count HEAD..upstream/Master

# 查看差异文件
git diff --name-only HEAD upstream/Master

# 合并（Rules/*.list 冲突取上游版本）
git merge upstream/Master --no-edit
git checkout --theirs Rules/*.list
git add Rules/
git commit
git push -u origin Master
```

## 同步上游流程 B（会话无上游 git 权限时）
Claude Code 远程会话的网络策略通常只允许访问本仓库，`git fetch upstream`
和 GitHub API 都会被拒（403），但 `raw.githubusercontent.com` 直连可用。
此时用逐文件下载方式同步（2026-07-02 已验证可行）：

```bash
# 1. 检查上游是否有更新（搜索 API 不受仓库范围限制）
#    用 MCP 工具 mcp__github__search_commits：
#    query: "repo:Rabbit-Spec/Surge committer-date:>=YYYY-MM-DD"

# 2. 批量下载上游文本文件覆盖本地（跳过 Surge-EN.conf）
RAW="https://raw.githubusercontent.com/Rabbit-Spec/Surge/Master"
for f in $(git ls-tree -r Master --name-only | grep -E '^(Rules/|Conf/Spec/|\.github/workflows/)' | grep -v 'Surge-EN.conf'); do
  curl -sS -o "$f" "$RAW/$f" --cacert /root/.ccr/ca-bundle.crt
done

# 3. Surge-EN.conf 单独下载到临时目录，diff 确认上游是否有实质变更，
#    有则手动合并（保留本地两处定制）

# 4. 用 mcp__github__search_code 检查上游是否新增文件：
#    query: "repo:Rabbit-Spec/Surge path:Rules/Manual" 等，缺的补下载

# 5. git add -A && git commit && git push -u origin Master
```

## 注意事项
- **本仓库自带 GitHub Action**（`.github/workflows/auto-rules.yml`），每天
  自动从原始源重新生成 `Rules/*.list` 并提交到 Master。因此：
  - 推送前先 `git fetch origin Master`，落后就先 rebase，规则冲突取最新版
  - 规则列表通常不需要手动同步，真正需要从上游同步的是：
    `auto-rules.yml` 本身、`Rules/Manual/*`、`Conf/Spec/*.conf`
- `Rules/*.list` 文件是自动生成的，冲突时一律取上游（`--theirs`）
- `Conf/Spec/Surge-EN.conf` 有手动改动，合并时保留上游更新 + 本地定制
- 其他 `Conf/Spec/*.conf` 直接跟上游同步即可
