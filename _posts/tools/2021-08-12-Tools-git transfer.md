---
layout: post
title: "Git迁移脚本"
categories: 工具
tags: 代码工具
author: 作者
typora-root-url: ..
---

迁移旧git的某个分支到新git上

```shell
#!/bin/bash

# 用法说明
usage() {
    echo "用法: $0 -s 源仓库URL -t 目标仓库URL -b 源分支名 -n 目标分支名 [-d 源仓库目录] [-u1 源仓库用户名] [-p1 源仓库密码] [-u2 目标仓库用户名] [-p2 目标仓库密码]"
    echo "示例: $0 -s https://github.com/source/repo.git -t https://github.com/target/repo.git -b source-branch -n target-branch -u1 user1 -p1 pass1 -u2 user2 -p2 pass2"
    exit 1
}

# 初始化变量
SOURCEREPO="your-src-git-repo"
TARGETREPO="your-target-git-repo"
SOURCEBRANCH="源库分支"
TARGETBRANCH="目标分支"
SOURCEDIR="gitmigrationtemp"
SOURCEUSER="源库的账号"
SOURCEPASS="源库的密码"
TARGETUSER="目标的账号"
#密码不要带有特殊符号不然拼在url中可能失败
TARGETPASS="目标的密码"

# 解析命令行参数
while getopts "s:t:b:n:d:u1:p1:u2:p2:" opt; do
    case "$opt" in
        s) SOURCEREPO="$OPTARG" ;;
        t) TARGETREPO="$OPTARG" ;;
        b) SOURCEBRANCH="$OPTARG" ;;
        n) TARGETBRANCH="$OPTARG" ;;
        d) SOURCEDIR="$OPTARG" ;;
        u1) SOURCEUSER="$OPTARG" ;;
        p1) SOURCEPASS="$OPTARG" ;;
        u2) TARGETUSER="$OPTARG" ;;
        p2) TARGETPASS="$OPTARG" ;;
        *) usage ;;
    esac
done

# 检查必要参数
if [ -z "$SOURCEREPO" ] || [ -z "$TARGETREPO" ] || [ -z "$SOURCEBRANCH" ] || [ -z "$TARGETBRANCH" ]; then
    usage
fi

# 构建源仓库URL（添加认证信息）
if [ -n "$SOURCEUSER" ] && [ -n "$SOURCEPASS" ]; then
    SOURCEREPO="${SOURCEREPO#*://}"
    SOURCEREPO="https://${SOURCEUSER}:${SOURCEPASS}@${SOURCEREPO}"
fi

# 构建目标仓库URL（添加认证信息）
if [ -n "$TARGETUSER" ] && [ -n "$TARGETPASS" ]; then
    TARGETREPO="${TARGETREPO#*://}"
    TARGETREPO="https://${TARGETUSER}:${TARGETPASS}@${TARGETREPO}"
fi

echo "目标地址$TARGETREPO"

exit 1

# 创建临时目录
mkdir -p "$SOURCEDIR"
cd "$SOURCEDIR" || exit

# 克隆源仓库的指定分支
echo "正在从源仓库克隆分支 $SOURCEBRANCH...$SOURCEREPO"
git clone --branch "$SOURCEBRANCH" --single-branch "$SOURCEREPO" .
if [ $? -ne 0 ]; then
    echo "克隆分支 $SOURCEBRANCH 失败"
    exit 1
fi

# 添加目标仓库远程地址
echo "正在添加目标仓库远程地址...$TARGETREPO"
if [ -n "$TARGETUSER" ] && [ -n "$TARGETPASS" ]; then
    # 使用用户名和密码认证
    git remote add target "$TARGETREPO"
else
    # 使用SSH或匿名访问
    git remote add target "$TARGETREPO"
fi

# 推送分支到目标仓库
echo "正在推送 $SOURCEBRANCH 分支到目标仓库的 $TARGETBRANCH 分支... $TARGETREPO"

git push target "$SOURCEBRANCH":"$TARGETBRANCH" -f

# 检查推送是否成功
if [ $? -ne 0 ]; then
    echo "推送目标仓库失败，请检查URL和认证信息"
    exit 1
fi

# 清理临时目录
echo "迁移完成，正在清理临时文件..."
cd ..
rm -rf "$SOURCEDIR"

echo "分支 $SOURCEBRANCH 已成功迁移到 $TARGETBRANCH"

```

