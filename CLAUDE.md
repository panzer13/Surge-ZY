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

## 注意事项
- `Rules/*.list` 文件是自动生成的，冲突时一律取上游（`--theirs`）
- `Conf/Spec/Surge-EN.conf` 有手动改动，合并时保留上游更新 + 本地定制
- 其他 `Conf/Spec/*.conf` 直接跟上游同步即可
