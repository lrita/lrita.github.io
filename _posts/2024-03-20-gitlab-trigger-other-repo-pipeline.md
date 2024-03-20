---
layout: post
title: 在 gitlab 中配置A项目触发B项目的 pipeline
categories: [git]
description: git gitlab
keywords: git gitlab
---

在一些项目中出现关联关系时，想在A项目触发B项目的`pipeline`时，可以参考的例子：

## 项目B

项目B为子项目，是被触发的一方。

```yaml
stages:
  - stage_1

xxx_job:
  stage: stage_1
  rules:
    - if: $CI_PIPELINE_SOURCE == "pipeline" # 外部项目触发，此处表示该job可以被外部项目触发，其他值可以参考 https://docs.gitlab.com/ee/ci/variables/predefined_variables.html
    - if: $CI_COMMIT_REF_NAME == "master"   # 代码合并入主分支时
  script:
    - echo "abc"                            # 具体任务命令
```

## 项目A

项目A为父项目，是发起的一方。父项目有两种配置方式：

### 方式一

在项目A的对应job中，调用项目B的对应API进行触发：

```yaml
stages:
  - deploy

xxx_job:
  stage: deploy
  script:
    - curl --request POST --form "token=$CI_JOB_TOKEN" --form ref=main "https://gitlab.example.com/api/v4/projects/9/trigger/pipeline"
  rules:
    - if: $CI_COMMIT_TAG
  environment: production
```

### 方式二

```yaml
stages:
  - deploy

xxx_job:
  stage: deploy
  trigger:                         # 使用trigger字段，表示该job是用来触发子pipeline的
    project: xx_group/project-b    # 项目B对应的名称
    branch: master                 # 对应项目的代码分支
                                   # trigger的下的其他字段功能可以参考 https://docs.gitlab.com/ee/ci/yaml/index.html#trigger
  rules:                           # 可以用rules控制该job执行条件来控制触发条件
    - if: '$CI_MERGE_REQUEST_ID'
```

![](/images/posts/tools/gitlab-multi-repo-pipeline.png)

## 扩展

其他更加复杂的用法可以参考[文档](https://docs.gitlab.com/ee/ci/pipelines/downstream_pipelines.html)，gitlab还支持一些更加复杂的用法，比如项目A给项目B下发`pipeline`配置，转发环境变量等