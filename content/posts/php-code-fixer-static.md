+++
date = '2023-03-10T08:23:25+08:00'
draft = false
title = 'PHP 代码格式化和静态检查'
summary = '本文将介绍如何在 Laravel 中实现自动的代码格式化和语法静态检查，其他框架大同小异'
+++

本文将介绍如何在 Laravel 中实现自动的代码格式化和语法静态检查，其他框架大同小异

- 引入 cs-fixer: `composer require stechstudio/laravel-php-cs-fixer --dev`
  - 这个包是为 Laravel 定制的，其他项目可以引用标准版 `composer require friendsofphp/php-cs-fixer --dev`
- 在 composer.json 中的 scripts 字段增加 "cs-fix": "php artisan fixer:fix --allow-risky=yes" （非必需，只是增加一个快捷调用的方式 composer cs-fix）
- 引入 phpstan: `composer require nunomaduro/collision --dev composer require nunomaduro/larastan --dev`
  - 这也是 Laravel 定制版的，其他项目可以引用标准版 `composer require phpstan/phpstan --dev`
- 在 composer.json 中的 scripts 字段增加 "analyse": "./vendor/bin/phpstan analyse app --level=5", 这样我们可以用 composer analyse 来对 app 目录进行静态检查

那么到这里已经可以对代码进行格式化和静态检查了，下一步就是将其自动化，这一步依靠 git hook 实现

- 首先安装包 `composer require brainmaestro/composer-git-hooks --dev`
- 在 composer.json 的 extra 字段增加如下配置
   ```json
    "hooks": {
        "pre-commit": [
            "./pre-commit"
        ],
        "pre-push": [
            "composer analyse"
        ],
        "post-merge": "composer install"
    }
    ```
- 很明显，我们缺少 pre-commit 脚本，此脚本代码如下，Laravel 可直接使用，其他项目稍微修改即可
    ```shell
    #!/usr/bin/env bash

    # "Do no harm" autoformatter pre-commit script template.

    # This example uses gofmt on all .go files.

    # Files matching this pattern will be auto-formatted.
    FILE_PATTERN='\.php$'
    # The tool to check for.
    FORMAT_TOOL=./vendor/bin/php-cs-fixer
    # The format command. File names will be appended to this.
    FORMAT_COMMAND=(php artisan fixer:fix --allow-risky=yes)

    # Find all staged files that meet some criteria, and exit early if there aren't any.
    MATCHING_FILES=(`git diff --name-only --cached --diff-filter=AM | \
    grep --color=never "$FILE_PATTERN"`)
    if [ ! "$MATCHING_FILES" ]; then
    exit 0
    fi

    # Verify that our formatter is installed; if not, warn and exit.
    if [ -z $(which "$FORMAT_TOOL") ]; then
    echo "$FORMAT_TOOL not on path; can not format. Please install."
    exit 2
    fi

    # Check for unstaged changes to files in the index.
    CHANGED_FILES=(`git diff --name-only ${MATCHING_FILES[@]}`)
    if [ "$CHANGED_FILES" ]; then
    echo 'You have unstaged changes to some files in your commit; skipping '
    echo 'auto-format. Please stage, stash, or revert these changes. You may '
    echo 'find `git stash -k` helpful here.'
    echo
    echo 'Files with unstaged changes:'
    for file in ${CHANGED_FILES[@]}; do
        echo "  $file"
    done
    exit 1
    fi
    # Format all staged files, then exit with an error code if any have uncommitted
    # changes.
    echo 'Formatting staged files . . .'
    ${FORMAT_COMMAND[@]}
    CHANGED_FILES=(`git diff --name-only ${MATCHING_FILES[@]}`)
    if [ "$CHANGED_FILES" ]; then
    echo 'Reformatted staged files. Please review and stage the changes.'
    echo
    echo 'Files updated:'
    for file in ${CHANGED_FILES[@]}; do
        echo "  $file"
    done
    exit 1
    else
    exit 0
    fi
    ```

- 在 composer.json 的 scripts 字段增加如下配置
    ```json
    "post-install-cmd": [
        "cghooks add --ignore-lock --force-win",
        "cghooks update"
    ],
    ```
    - 这一步的目的在于让每一个开发者执行 composer install 的时候都能将 pre-commit 注册进 git hook
    - cghooks 是一个 php 脚本，是 brainmaestro/composer-git-hooks 包中携带的，安装即可
- 执行 composer install 激活 git hook

这样就完成了，我们在 git commit 的时候会进行代码格式化，在 git push 前会进行静态检查，如果静态检查不通过会 push 失败

但是需要注意在线上部署的时候需要使用 composer install --no-dev --no-scripts -o