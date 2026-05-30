---
name: commit
description: 提交代码到三个仓库（librime, squirrel, Rime配置），生成中文commit message，自动更新子模块指针
---

# Commit Skill

提交本项目的代码变更。涉及三个仓库：`librime/`（C++ 引擎）、squirrel（macOS 前端）、Rime 配置（`~/Library/Rime`）。

## 🚨 关键规则（违反即为失败）

1. **绝对不要自动提交。** 必须先用 `AskUserQuestion` 工具弹框让用户确认。
2. 用户确认前，只能做展示和分析，不能执行任何 `git commit`。
3. 该规则适用于/commit被调用时的每一次提交确认，没有例外。

## 提交流程

### 1. 检查变更并展示摘要

```bash
echo "=== librime ===" && git -C librime status --short
echo "=== squirrel ===" && git status --short
echo "=== Rime ===" && git -C ~/Library/Rime status --short
```

列出每个仓库的变更摘要和拟定的 commit message。

### 2. 弹出确认框（必须）

**必须调用 `AskUserQuestion` 工具**，让用户选择是否提交。在用户做出选择之前，不执行任何 git 操作。

```
AskUserQuestion({
  questions: [{
    question: "是否提交以上变更？",
    header: "确认提交",
    options: [
      {label: "提交", description: "按上述 message 执行提交"},
      {label: "取消", description: "不做任何操作"},
      {label: "修改 message", description: "用户可自行输入修改后的 message"}
    ]
  }]
})
```

### 3. 生成 commit message（格式参考）

遵循已有风格：

**librime**（`librime/` 目录下的 git submodule）：
- 格式：`type(scope): 中文描述`
- type: `feat`, `fix`, `chore`, `docs`, `refactor`
- scope: `dict`, `gear`, `build`, `algo` 等模块名
- 参考：`feat(dict): 使用 tick 作为 pin 排序键`、`fix(gear): 修复 PinProcessor 重复置顶时 custom_code 空格累积`

**squirrel**（项目根目录）：
- 如果只改了 librime 子模块指针：`chore: 更新 librime 子模块（简短描述）`
- 其他改动：`type(scope): 中文描述`

**Rime 配置**（`~/Library/Rime`）：
- 格式：中文描述，无前缀
- **永远不要提交 `*.userdb.txt`**（用户词典数据文件）
- 参考：`启用 text_userdb_sync 实时持久化用户词典`

### 4. 按顺序提交

```
1. librime  ← 先提交引擎改动
2. squirrel ← 如果 librime 有变，更新子模块指针
3. Rime配置  ← 最后提交配置变更
```

### 5. 执行提交

对每个仓库：
```bash
git -C <repo_path> add <specific_files>  # 精确指定文件，不要 git add -A
git -C <repo_path> commit -m "$(cat <<'EOF'
<message>
EOF
)"
```

## 推送

**永远不要主动 `git push`**，除非用户明确要求。

## 注意

- 每个分支一个主题，不要在同一个分支上混入不相关的改动
- Rime 配置中用 `git add` 精确指定文件，避免误提交 `userdb.txt`
- squirrel 的子模块更新必须在 librime 提交之后
