---
layout: post
title: "Git-Tools 批量克隆gitlabgroup"
categories: Git
tags: 工具
author: fengsh998
typora-root-url: ..
---

批量拉取gitlab 指定组下的所有子目录及子目录下的git仓库。

```shell
#!/bin/bash

#==========================================
#        使用方式 ./batchclone.sh <groupid>
#
#==========================================
# --- 配置区 ---
# 你的 GitLab 实例地址
GITLAB_URL="https://gitlab.com" //your-gitlab.com
# 本地存储所有代码的根目录
TARGET_BASE_DIR="./gitlab-clones"
# TOKEN  user settings -> access token create.
GITLAB_PRIVATE_TOKEN="your-token"
# --- 配置区结束 ---


# 检查必要的命令是否存在
if ! command -v git &> /dev/null; then
    echo "错误: git 未安装。"
    exit 1
fi

# 检查是否设置了 GitLab Token
if [ -z "$GITLAB_PRIVATE_TOKEN" ]; then
    echo "错误: 请设置环境变量 GITLAB_PRIVATE_TOKEN。"
    echo "例如: export GITLAB_PRIVATE_TOKEN='your_token_here'"
    exit 1
fi

# 检查是否提供了 Group ID 或 Path 作为参数
if [ -z "$1" ]; then
    echo "用法: $0 <group_id_or_path>"
    echo "例如: $0 12345"
    echo "或者: $0 my-awesome-group/sub-group"
    exit 1
fi

GROUP_ID_OR_PATH="$1"
# 创建本地根目录
mkdir -p "$TARGET_BASE_DIR"
cd "$TARGET_BASE_DIR"

# 用于递归克隆组和项目的函数
# 参数: $1 -> group_id_or_path
clone_group_and_projects() {
    local group_path="$1"
    local encoded_path=$(printf '%s' "$group_path" | jq -sRr @uri 2>/dev/null || python3 -c "import sys, urllib.parse; print(urllib.parse.quote(sys.argv[1]))" "$group_path" 2>/dev/null || python -c "import sys, urllib; print urllib.quote_plus(sys.argv[1])" "$group_path" 2>/dev/null)

    if [ -z "$encoded_path" ]; then
        # 如果jq/python都不可用，使用一个简单的替换方法，但这对于复杂路径不安全
        echo "警告: 未找到jq或python来对路径进行URL编码，使用简单替换，可能对特殊字符不安全。"
        encoded_path=${group_path//\//%2F}
        encoded_path=${encoded_path// /%20}
    fi


    echo -e "\n=========================================="
    echo -e "正在处理组: \033[34m$group_path\033[0m"
    echo -e "=========================================="

    # 1. 克隆当前组下的所有项目
    echo "  -> 正在获取组项目列表..."
    local page=1
    while true; do
        # 调用 GitLab API 获取项目列表，支持分页
        local api_result=$(curl --silent --header "PRIVATE-TOKEN: $GITLAB_PRIVATE_TOKEN" \
            "$GITLAB_URL/api/v4/groups/$encoded_path/projects?include_subgroups=false&per_page=100&page=$page")

        # 简单的错误检查：如果返回的JSON包含 "message"，则认为是错误
        if echo "$api_result" | grep -q '"message"'; then
            echo -e "  \033[31m错误: API 调用失败。请检查 Group ID/Path 和 Token 权限。\033[0m"
            echo -e "  \033[31m详情: $api_result\033[0m"
            return 1
        fi

        # 检查是否为空数组 "[]"
        if [ "$api_result" = "[]" ]; then
            break # 没有更多项目了
        fi

        # --- 纯Shell JSON解析开始 ---
        # 这个方法假设JSON对象中的值不包含双引号
        echo "$api_result" | grep -oE '{[^}]*"path_with_namespace":"[^"]*"[^}]*"http_url_to_repo":"[^"]*"[^}]*}' | while IFS= read -r project_line; do
            # 提取 path_with_namespace
            local project_path=$(echo "$project_line" | sed -n 's/.*"path_with_namespace":"\([^"]*\)".*/\1/p')
            # 提取 http_url_to_repo
            local project_url=$(echo "$project_line" | sed -n 's/.*"http_url_to_repo":"\([^"]*\)".*/\1/p')

            if [ -n "$project_path" ] && [ -n "$project_url" ]; then
                local local_dir=$(dirname "$project_path")

                echo "    -> 正在克隆项目: $project_path"
                mkdir -p "$local_dir"
                
                # 如果目录已存在，则拉取最新代码
                if [ -d "$project_path" ]; then
                    echo "      -> 项目已存在，正在拉取最新代码..."
                    (cd "$project_path" && git pull)
                else
                    git clone "$project_url" "$project_path"
                fi
            fi
        done
        # --- 纯Shell JSON解析结束 ---

        # 检查是否还有下一页
        # GitLab API会在响应头中包含 X-Next-Page，这里简化处理，如果返回的项目数小于100，说明是最后一页
        local project_count=$(echo "$api_result" | grep -o '"path_with_namespace"' | wc -l | tr -d ' ')
        if [ "$project_count" -lt 100 ]; then
            break
        fi

        page=$((page + 1))
    done

    # 2. 查找并递归处理子组
    echo "  -> 正在获取子组列表..."
    local subgroups_result=$(curl --silent --header "PRIVATE-TOKEN: $GITLAB_PRIVATE_TOKEN" \
        "$GITLAB_URL/api/v4/groups/$encoded_path/subgroups?per_page=100")

    if echo "$subgroups_result" | grep -q '"message"'; then
        echo -e "  \033[31m错误: 获取子组列表失败。\033[0m"
        return 1
    fi

    # --- 纯Shell JSON解析子组 ---
    echo "$subgroups_result" | grep -o '"full_path":"[^"]*"' | cut -d'"' -f4 | while IFS= read -r subgroup_path; do
        if [ -n "$subgroup_path" ]; then
            clone_group_and_projects "$subgroup_path"
        fi
    done
}

# --- 主程序入口 ---
echo "开始从组 '$GROUP_ID_OR_PATH' 克隆所有项目和子组到目录 '$TARGET_BASE_DIR'..."
clone_group_and_projects "$GROUP_ID_OR_PATH"

echo -e "\n\033[32m所有任务完成！\033[0m"

```
