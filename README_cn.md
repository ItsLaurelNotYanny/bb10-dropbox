# bb10-dropbox

[English](README.md) | 简体中文

面向 BlackBerry 10（BlackBerry Passport 实机验证）的原生 Dropbox 客户端与后台同步服务。

## 概述

`bb10-dropbox` 专为运行 QNX 6.6 / Cascades 的 BlackBerry 10 设备设计，基于原生 Qt 4.8 构建：
- 通过 Dropbox API v2 实现流畅的手动文件浏览、上传与下载；
- 独立的后台同步服务，支持双向文件夹同步与相册自动备份；
- 专为资源受限的 BB10 设备优化的轻量化架构与低依赖设计。

## 仓库架构

本项目遵循双仓库开发惯例：
- **私有仓库 (Private Repo)** (`bb10-dropbox-private`)：跟踪完整的研发提交历史、阶段里程碑、评估文档及工程细节。
- **开源子集仓库 (Public Repo)** (`bb10-dropbox`)：准备开源发布的公开代码树，提供纯净的核心客户端、同步服务及公开文档。

日常研发保持在**同一个工作区 (single worktree)** 继续作业，通过 `scripts/prepare_release.sh` 自动生成并验证开源子集发布分支。

## 目录结构

```text
bb10-dropbox/
├── REUSE-ASSESSMENT.md      # 上游复用评估与架构决策（英文）
├── REUSE-ASSESSMENT_cn.md   # 上游复用评估与架构决策（中文）
├── README.md                # 项目文档（英文）
├── README_cn.md             # 项目文档（中文）
├── check.sh                 # 工程门禁检查脚本
└── scripts/
    └── prepare_release.sh   # 开源子集分支准备脚本
```

## 快速开始与门禁检查

运行项目工程门禁：
```bash
# 快速检查（语法、格式与基础静态校验）
./check.sh --fast

# 全量检查
./check.sh
```

## 准备开源发布子集

在主开发分支（`main`）保持在同一个 worktree 的情况下，运行以下脚本生成/更新开源发布分支（`release`）：
```bash
./scripts/prepare_release.sh
```

推送到远程仓库：
```bash
# 推送完整开发分支至私有仓库
git push origin main

# 推送纯净发布分支至公开仓库
git push public release:main
```

## 许可证

待正式开源发布时确定。
