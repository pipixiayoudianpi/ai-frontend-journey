# Week 1（2026-06-08 ~ 进行中）

## 阶段
第 0 阶段：地基补强

## 主题
- 终端环境升级
- 30 个命令行命令突击
- Git 命令行完整流程（脱离 GitHub Desktop）
- GitHub 远程协作（PR / 冲突 / rebase / stash）
- 建立公开 GitHub 主页和本学习仓库

## 完成情况

### Day 1 — 终端环境
- iTerm2、Homebrew（5.1.7）、oh-my-zsh（已装）
- powerlevel10k 主题（two-line + dotted prompt）
- zsh-autosuggestions + zsh-syntax-highlighting

### Day 2-3 — 命令行 30 命令
学到的实用命令：`pwd` `ls -la/-lh` `cd -` `mkdir -p` `cp -r` `mv` `rm` `cat` `less` `head/tail` `tail -f` `find -name/-type/-mtime` `grep -rEni --include -A/-B/-C` `rg` `fd` `|` `>` `>>` `2>` `/dev/null` `wc` `sort` `uniq -c` `awk` `sed` `tr` `xargs` `ps aux` `lsof -i :port -Pn` `kill` `lsof -ti :port | xargs kill` `echo $VAR` `which` `export` `curl -X/-H/-d/-s/-o` `jq -r`

综合练习：用纯命令行生成 markdown 格式的"项目体检报告"。

### Day 4-5 — Git
- **本地全流程**：init / status / add / commit / log / branch / switch / merge / restore / restore --staged / amend / reset / revert / diff
- **远程协作**：gh CLI 配置 / remote / push -u / fetch / pull / git push --force-with-lease
- **冲突解决**：手动消除 conflict markers / merge --abort
- **rebase**：git rebase main / git rebase -i HEAD~N（squash / fixup / reword）
- **stash**：git stash -u / pop / list
- **PR 流程**：gh pr create / gh pr merge --squash --delete-branch

可见成果：[git-practice](https://github.com/pipixiayoudianpi/git-practice) 公开仓库（含合并的 PR #1）。

### Day 6-7 — 公开主页
- ✅ 特殊 profile 仓库 [pipixiayoudianpi/pipixiayoudianpi](https://github.com/pipixiayoudianpi)：徽章 + GitHub Stats 卡片 + Currently Learning
- ✅ 本仓库 [ai-frontend-journey](https://github.com/pipixiayoudianpi/ai-frontend-journey)
- 🚧 第一篇博客《我的命令行 + Git 学习记录》

## 踩过的坑

1. **多行命令粘贴到终端偶发截断** → 长命令逐行粘
2. **`rm -f` 静默失败** → 路径写错（compoents/components）不报错 → 删普通文件不要带 `-f`
3. **`> file` 在父目录不存在时报"找不到文件"** → 报错指向的是文件，问题在父目录
4. **commit message 全角冒号 `：`** → 中文输入法没切回，养成切英文的肌肉记忆
5. **`amend` 跨 commit 误用** → 把当前 staged 改动并进上一条 commit，message 含义错配 → 教训："amend 只用于补漏同主题改动"
6. **`revert + amend` 互相搅** → 历史长出"两条互相抵消的 commit" → 用 `git reset --hard <hash>` 拉回干净点
7. **amend 后忘记 force push** → `non-fast-forward` 错误 → `git push --force-with-lease`

## 心智模型沉淀

- **Git 三区模型**：工作区 / 暂存区（staging）/ 仓库（commit 历史）
- **分支只是指针**：建分支几乎零成本，merge 是把指针移动 / 创建合并 commit
- **撤销 = 反向跨区**：restore（仓库→工作区）→ restore --staged（暂存→工作区）→ reset HEAD~1（仓库→暂存或工作区）→ reset --hard（彻底丢弃）
- **merge vs rebase**：merge 保留叉路、rebase 整理成直线；rebase 改 hash，黄金法则"不要 rebase 已推送的 commit"

## 下周计划

Week 2：TypeScript 入门 + Vue 3 Composition API 熟练化（用 Vite + Vue 3 + TS + `<script setup>` + Pinia 重做一个 CRUD 小项目）
