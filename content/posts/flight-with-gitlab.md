+++
date = '2023-10-31T08:37:38+08:00'
draft = false
title = 'gitlab ci 笔记'
summary = '和 gitlab 第一次交手'
+++

### 背景
是这样的，公司内有一个需求是在 CI 阶段完成文档的构建和输出到另一个工程，那就需要在 gitlab 提供的工具（框架）内完成这件事，于是就开始和 .gitlab-ci.yml 纠缠的这件事

### 准备
首先要熟悉一下这个 yml 文件，文件的基本语法百度都有就不赘述，这里只记录一下我的问题

其实我本想在 Merge 之后再做这件事，本以为会有个这种事件，但是其实不是的，这个文件内的所有 stage 都是在合并前执行的

换句话说，只有 push 了，就会触发 pipeline , 执行的流程也就是 stage 的顺序

### 解决环境依赖
这就很狗了， gitlab 的 runner 只提供一个最基础的运行环境，那我们需要 apidoc / git 这些工具就很难办，不过还好公司使用的是 k8s ，还好 gitlab 支持 docker

所以增加了一个 image: xxx:xx 的配置项，我分了两个阶段，第一个阶段是生成文档的 json ，第二个阶段是把生成出来的 json 同步到另外一个仓库，阶段二很简单，有现成的 git 镜像可以直接使用，但是阶段一就很难看了， apidoc 并没有提供镜像，何况我还需要几个插件，所以只好自己封装一个镜像，问题至此也解决了

### 解决文件的跨任务传递
那么既然是两个阶段来完成，而且生成的 json 是需要在第二个阶段去上传的，那就必须将生成的 json 文件传递到阶段二，也就是第二个任务，我忘记怎么搜索的了，但是最后的解决方法是在第一阶段的任务中增加以下配置

```yaml
artifacts:

paths:

- file
```

### 同其他 git 仓库交互
最终生成的 json 文件还是要上传到指定的仓库的，对此我找到了一篇文档

https://cloud.tencent.com/developer/article/2186300

我做简要概述

首先生成自己的 access_token ，然后将此 token 和自己的用户名字注入进 CI 中，分别命名为 GITLAB_TOKEN / GITLAB_USERNAME

那么拉取的模版就是

git clone "https://${GITLAB_USERNAME}:${GITLAB_TOKEN}@${CI_SERVER_HOST}/${GITLAB_USERNAME}/test_json.git"

推送的模版就是

git push https://gitlab-ci-token:${GITLAB_TOKEN}@${CI_SERVER_HOST}/${GITLAB_USERNAME}/test_json.git master

那其实还有个问题，不是每次 push 都修改了文档的，也就是没有必要每次都触发 pipeline, 对此我并没有更好的办法，只能做局部的优化

每次生成完 json 文件后，检查有没有变更，如果没有就放弃提交

### 最后放上我的完整 yaml 文件
```yaml
# 定义两个步骤, 生成文档和同步生成的文档去另一个仓库
stages:
  - genDocs
  - syncJsonRepo

genDocJob:
  stage: genDocs
  script: 
    - echo "I'm doing gen doc..."
    - apidoc -h
    - node -v && apidoc --version
    - apidoc -i apps --write-json -o apidoc/
    - cp ./apidoc/assets/api-data.json ./
    - rm -rf apidoc/
  only:
    refs:
      - dev # 仅在 dev 分支生效
  image: wzhihui/uhome_api_doc:0.3 # 使用包含 apidoc 的镜像
  artifacts:
    paths:
      - api-data.json # 这个是生成的 json


syncJsonRepoJob:
  stage: syncJsonRepo
  script:
    - echo "I'm doing sync json repo job.."
    - git config --global user.email "ci@example.com"
    - git config --global user.name "ci"
    - git clone "https://${GITLAB_USERNAME}:${GITLAB_TOKEN}@${CI_SERVER_HOST}/json.git" # 使用 access_token 来拉取
    - cp ./api-data.json ./json/
    - cd json
    - |
      if [[ $(git status --porcelain) ]]; then
        git add . && git commit -m "from ci"
        git push https://gitlab-ci-token:${GITLAB_TOKEN}@${CI_SERVER_HOST}/json.git master
      else
      echo "nothing to change"
      fi
  only:
    refs:
      - dev
  image: bitnami/git:2.42.0
```