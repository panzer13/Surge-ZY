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
Claude Code 远程会话的网络策略通常只允许访问本仓库，`git fetch upstream`、
GitHub API（含 `get_file_contents`、codeload 打包下载）都会被拒 403，
但以下两条通道可用（2026-07-02、2026-08-07 两次验证）：
- `raw.githubusercontent.com` 直连下载单个文件
- MCP 搜索工具 `search_commits` / `search_code`（不受仓库范围限制）

**只需对比 4 类非自动生成文件**，`Rules/*.list` 由本仓库 Action 每天重新
生成，不用下载对比（这是 8 月那次把检查从几分钟缩到几秒的关键）：

```bash
RAW="https://raw.githubusercontent.com/Rabbit-Spec/Surge/Master"
SCRATCH=/tmp/up && mkdir -p $SCRATCH

# 1. 先看上游有没有新提交（YYYY-MM-DD 填上次同步日期）
#    mcp__github__search_commits:
#    query: "repo:Rabbit-Spec/Surge committer-date:>=YYYY-MM-DD"
#    重点看非 github-actions[bot] 的手动提交，那才是要同步的

# 2. 逐个对比关键文件，同时检测「上游已删除」
for f in ".github/workflows/auto-rules.yml" \
         "Rules/Manual/China.txt" "Rules/Manual/Proxy.txt" \
         "Rules/Manual/Proxy.exclude.txt" "Rules/Manual/ChinaCIDR.txt" \
         "Rules/Manual/TikTok.txt" "Rules/Manual/AIGC.exclude.txt" \
         "Rules/Manual/Telegram.exclude.txt"; do
  code=$(curl -sS -o "$SCRATCH/$(basename $f)" -w "%{http_code}" \
         "$RAW/$f" --cacert /root/.ccr/ca-bundle.crt)
  if [ "$code" != "200" ]; then echo "[404] $f ← 上游已删除，本地也要 git rm"
  elif [ ! -f "$f" ]; then echo "[NEW] $f ← 本地缺失，要补"
  elif ! diff -q "$f" "$SCRATCH/$(basename $f)" >/dev/null; then echo "[DIFF] $f"
  fi
done

# 3. Manual 目录清单以上游为准（防止漏掉新增文件）
#    mcp__github__search_code: query: "repo:Rabbit-Spec/Surge path:Rules/Manual"

# 4. Conf/Spec/*.conf 同法对比；Surge-EN.conf 要剥离本地定制后再比：
diff <(sed 's/^# > ✈️ 我的节点/✈️ 我的节点/' Conf/Spec/Surge-EN.conf \
       | sed '/^\[SSID Setting\]$/,$d' \
       | sed -e :a -e '/^$/{$d;N;ba' -e '}') "$SCRATCH/Surge-EN.conf"
#    只剩「\ No newline at end of file」= 上游无实质变更，不要动这个文件

# 5. 提交前 git fetch origin Master 确认 Action 没有新提交，
#    再 git push -u origin Master
```

**上游会删除文件，别只顾着下载。** 2026-07-09 上游删掉了
`Rules/Manual/Game.txt`，把里面的 Pokemon Sleep 规则并进了 `China.txt`。
`auto-rules.yml` 用 `if [ -f "$GAME_MANUAL" ]` 判断，本地留着旧文件不会报错，
但会导致生成的 `Game.list` 多一条、`China.list` 少一条，规则悄悄走错策略。
所以步骤 2 里的 404 检测必须做。

## 注意事项
- **本仓库自带 GitHub Action**（`.github/workflows/auto-rules.yml`），每天
  自动从原始源重新生成 `Rules/*.list` 并提交到 Master。因此：
  - 推送前先 `git fetch origin Master`，落后就先 rebase，规则冲突取最新版
  - 规则列表通常不需要手动同步，真正需要从上游同步的是：
    `auto-rules.yml` 本身、`Rules/Manual/*`、`Conf/Spec/*.conf`
  - 改完 `Rules/Manual/*` 不必手动改 `Rules/*.list`，下次 Action 跑会重新生成
- **推送到 `Master`**：Surge 客户端和 Action 都依赖这个分支，推到别的分支等于
  没生效。若会话被指定了 `claude/*` 开发分支，两个分支都推一份即可
- `Rules/*.list` 文件是自动生成的，冲突时一律取上游（`--theirs`）
- `Conf/Spec/Surge-EN.conf` 有手动改动，合并时保留上游更新 + 本地定制
- 其他 `Conf/Spec/*.conf` 直接跟上游同步即可

## 同步历史
| 日期 | 上游进度 | 实际同步内容 |
|---|---|---|
| 2026-06-15 | 70 个提交 | 首次合并，规则全量 + 配置升 5.0.6 |
| 2026-07-02 | 23 个提交 | auto-rules.yml 升 1.3.9、新增 Manual/TikTok.txt |
| 2026-08-07 | 52 个提交 | 仅 3 个文件：Unity 流量改直连（详见上方 Game.txt 说明） |

规律：上游每天一个 bot 自动提交，**真正需要同步的只有作者的手动提交**，
用 `search_commits` 结果里 `author.login != github-actions[bot]` 快速筛。
