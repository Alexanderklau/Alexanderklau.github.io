---
title: GitLab/ArgoCD/Jenkins CI/CD方案梳理和对比
date: 2024-07-30 21:07:25
tags: ["前后端"]
---
## CI/CD的概述

### 良好的CI/CD应该拥有哪些功能

#### 自动化构建

1. 自动化构建触发：每次代码提交、合并请求、或其他触发事件时，自动启动构建过程。
2. 依赖管理：自动处理项目依赖，确保构建环境的一致性。
3. 可复用的构建脚本：使用脚本或配置文件（如Makefile、Gradle、Maven等）来定义构建过程，确保构建步骤的一致性和可复用性。
4. 版本控制：构建过程中自动生成版本号或构建标签，以便于版本管理和追踪。

#### 自动化测试

自动化测试是CI/CD系统确保代码质量的重要功能，通过自动化测试，开发团队可以快速发现和修复问题。

1. 单元测试：每次构建后自动运行单元测试，确保代码功能的正确性。
2. 集成测试：在代码合并到主分支前运行集成测试，验证不同模块的协同工作。
3. 端到端测试：模拟用户行为，验证应用的整体功能和性能。
4. 代码覆盖率：生成代码覆盖率报告，确保测试覆盖了足够的代码。

#### 自动化部署

自动化部署功能使应用能够快速、安全地部署到各种环境（如开发、测试、生产环境）。

1. 多环境部署：支持将应用部署到不同的环境（开发、测试、UAT、生产），每个环境有独立的配置。
2. 蓝绿部署/金丝雀发布：在生产环境中进行蓝绿部署或金丝雀发布，以降低部署风险。
3. 配置管理：通过配置文件或环境变量管理不同环境的配置。
4. 基础设施即代码：使用工具（如Terraform、Ansible等）自动化管理和部署基础设施。

#### 监控与反馈

监控与反馈功能确保开发团队能够及时了解系统的健康状态和性能，并迅速响应问题。

1. 实时监控：监控应用的性能、日志和资源使用情况，及时发现和响应问题。
2. 报警系统：设置监控报警，及时通知相关人员处理异常情况。
3. 反馈循环：将监控和测试结果反馈给开发团队，持续改进代码质量。
4. 可视化仪表板：提供可视化的监控和报告仪表板，便于快速查看系统状态和构建结果。

#### 回滚机制

回滚机制确保在出现问题时能够快速恢复到稳定状态，减少对业务的影响。

1. 自动回滚：在新版本部署失败或出现严重问题时，自动回滚到上一个稳定版本。
2. 手动回滚：提供手动触发回滚的能力，确保在自动回滚失效时能够人工干预。
3. 版本管理：保存所有发布版本的记录，便于快速回滚和问题排查。
4. 数据库迁移回滚：在回滚应用代码时，能够同时回滚数据库迁移，确保数据一致性。


## CI/CD在Kubernetes中的最佳实践方案梳理

根据前期提到的优秀的CICD的功能，结合网络上的方案

梳理出如下最佳方案

### Gitlab CI/CD

#### 简介

GitLab CI/CD 是一个内置于 GitLab 中的持续集成和持续交付/部署解决方案.

它通过 GitLab Runner 执行定义在 .gitlab-ci.yml 文件中的任务（如构建、测试和部署）.

GitLab CI/CD 提供了一个强大的框架，用于自动化软件开发生命周期中的各个阶段，从代码提交到生产环境的部署。

#### 先决条件

1. 需要配置Kubernetes集群的访问权限
2. 需要配置aws权限

#### 流程

流程如图所示

![alt text](https://blog-1256169066.cos.ap-chengdu.myqcloud.com/cicd/image-3.png)

详细解释：

1. 开发人员提交代码
2. 开发人员将代码推送到 GitLab 代码库（GitLab Code Repo）。代码库包含源代码、Dockerfile 和 GitLab CI 配置文件（.gitlab-ci.yml）。
3. 测试代码：运行代码测试，确保代码逻辑正确。
4. 单元测试（UT）：执行单元测试，验证各个模块的功能。
5. Lint 检查：运行代码风格检查（Lint），确保代码符合规范。
6. 在通过初始测试后，GitLab CI 管道继续执行以下步骤：
7. 构建镜像：使用 Dockerfile 构建 Docker 镜像。
8. 扫描镜像：对构建的镜像进行安全扫描，检查潜在的漏洞。
9. 推送到AWS ECR：将通过扫描的镜像推送到 ECR。
10. 部署阶段，部署deployment 到kubernetes集群

#### example

根据上述流程完成了一个CI/CD流程

![alt text](https://blog-1256169066.cos.ap-chengdu.myqcloud.com/cicd/image-8.png)

.gitlab-ci.yml示例

```yaml

test:
  stage: test
  script:
    - echo "Test Docker image $DOCKER_IMAGE"

build:
  stage: build
  image: $CONTIANER_TOOL_SET_IMAGE
  script:
    - cifab build-to-ecr -i $ECR_IMAGE_URI -t $CI_COMMIT_TAG -d $CI_PROJECT_DIR/Dockerfile -r us-east-1
    - cifab build-to-ecr -i $ECR_IMAGE_URI_CN -t $CI_COMMIT_TAG -d $CI_PROJECT_DIR/Dockerfile -r cn-north-1
  tags:
    - ut
  only:
    - tags

deploy:
  stage: deploy
  script:
    - sed -i "s|ECR_IMAGE_URI_PLACEHOLDER|$ECR_IMAGE_URI:$CI_COMMIT_TAG|g" $CI_PROJECT_DIR/k8s/deployment.yml
    - kubectl apply -f k8s/deployment.yaml
  tags:
    - ut
  only:
    - tags
  environment:
    name: dev
    url: http://127.0.0.1
```

![alt text](https://blog-1256169066.cos.ap-chengdu.myqcloud.com/cicd/image-9.png)

执行部署

![alt text](https://blog-1256169066.cos.ap-chengdu.myqcloud.com/cicd/image-10.png)

部署成功

![alt text](https://blog-1256169066.cos.ap-chengdu.myqcloud.com/cicd/image-11.png)

### Gitlab CI + ArgoCD

#### 简介

#### 先决条件

1. 需要ArgoCD搭建
2. 需要在ArgoCD配置gitlab的校验

#### 流程

流程如图所示

![alt text](https://blog-1256169066.cos.ap-chengdu.myqcloud.com/cicd/image-4.png)

详细解释：

1. 开发者将代码推送到GitLab代码库（GitLab Code Repo），代码库包含源码、Dockerfile和GitLab-CI配置文件（.gitlab-ci.yml）
2. 进行代码测试（Test Code）、单元测试（UT）和代码风格检查（Lint Check）。
3. 构建Docker镜像（Build Image）、扫描镜像（Scan Image）、将镜像推送到镜像仓库（Push to Registry），并更新清单文件（Update Manifest）。
4. 构建的Docker镜像被推送到ECR镜像仓库（ECR Registry）。
5. 在GitLab中进行部署更新，更新清单文件（Update Manifest）。
6. 使用ArgoCD同步GitLab代码库中的清单文件（Sync Repo），将更新部署到Kubernetes集群中（Deploy to Cluster）。
7. Kubernetes从ECR镜像仓库中拉取最新的镜像（Pull Image），部署应用。

#### example

job如下
![alt text](https://blog-1256169066.cos.ap-chengdu.myqcloud.com/cicd/image-15.png)

![alt text](https://blog-1256169066.cos.ap-chengdu.myqcloud.com/cicd/image-13.png)

手动部署deployment

![alt text](https://blog-1256169066.cos.ap-chengdu.myqcloud.com/cicd/image-14.png)

修改deployment.yml的内容

![alt text](https://blog-1256169066.cos.ap-chengdu.myqcloud.com/cicd/-new.png)

argocd自动部署

![alt text](https://blog-1256169066.cos.ap-chengdu.myqcloud.com/cicd/image-12.png)

### Gitlab + jenkins

#### 简介

#### 先决条件

1. 需要jenkins搭建在内网（已经搭建完毕）
2. 需要在jenkins上安装多个插件 （gitlab，cicd，k8s，pipline）
3. 需要kubernetes提供token ![alt text](https://blog-1256169066.cos.ap-chengdu.myqcloud.com/cicd/image-5.png)
4. 需要在kubernetes上部署代理，连接jenkins
5. 需要aws权限
6. 需要gitlab的token权限

#### 流程

![alt text](https://blog-1256169066.cos.ap-chengdu.myqcloud.com/cicd/image-2.png)

1. 开发者将代码推送到GitLab代码库（GitLab Code Repo）
2. 代码推送到GitLab后，通过Webhook触发Jenkins任务
3. Jenkins收到Webhook触发的任务后，开始构建Docker镜像 (通过Docker插件)
4. 构建完成的Docker镜像通过docker插件推送到ECR镜像仓库（ECR Registry）
5. Kubernetes插件从ECR镜像仓库中拉取最新的镜像，部署应用

#### example

设定一个Jenkins的webhook

![alt text](https://blog-1256169066.cos.ap-chengdu.myqcloud.com/cicd/image-17.png)

编写jenkinsfile

![alt text](https://blog-1256169066.cos.ap-chengdu.myqcloud.com/cicd/image-18-new.png)

设置好jenkins pipline（步骤较为麻烦，这里不再论述）

![alt text](https://blog-1256169066.cos.ap-chengdu.myqcloud.com/cicd/image-19-new1.png)

![alt text](https://blog-1256169066.cos.ap-chengdu.myqcloud.com/cicd/image-20.png)

点击部署

![alt text](https://blog-1256169066.cos.ap-chengdu.myqcloud.com/cicd/image-22.png)

部署成功
![alt text](https://blog-1256169066.cos.ap-chengdu.myqcloud.com/cicd/image-21.png)



## CI/CD的方案对比

### 方案对比表

![alt text](https://blog-1256169066.cos.ap-chengdu.myqcloud.com/cicd/image-7.png)

### 优缺点梳理

#### Gitlab

优点

1. 完整的平台，可以自定义所有的CI/CD流程
2. 内置的CI/CD功能适合大部分的部署场景
3. 管理比较简单，所有的功能都在Gitlab上

缺点

1. 扩展性比较差，没有插件系统
2. 对于复杂的CI\CD流程还是需要外部工具帮忙

#### Gitlab + ArgoCD

优点

1. 通过 Argo CD 实现声明式配置管理和自动化同步
2. 专注Kubernetes管理，提供强大的 Kubernetes 部署和监控功能
3. 自动化同步，持续监控 Git 仓库的配置文件，发现修改及时更新
4. 和gitlab的集成，结合gitlab的代码托管，形成完整的CI/CD流程

缺点

1. 需要对项目进行配置，项目越多，配置越多
2. 专注支持Kubernetes管理，其他环境支持比较弱

#### Gitlab + Jenkins

优点

1. 插件系统强，插件多，可以覆盖所有CI/CD需求
2. 通过编写pipline code，进行自定制pipline
3. 和gitlab的集成非常方便

缺点

1. 过于复杂，维护成本高
2. 配置复杂，k8s的配置token比较复杂
3. 学习成本较高，需要花费大量时间学习。

## 推荐方案

推荐Gitlab + ArgoCD

### 推荐原因

1. 此次需求定义的项目全是EKS项目，正好符合ArgoCD的专属kubernetes
2. 已经配置好了ArgoCD，编写pipline只需要基础的GITLAB支持
3. GitOps 模型使得变更记录清晰可追溯，方便审计和回滚
4. Argo CD 可以持续监控 Git 仓库中的配置文件，当检测到配置变更时，自动将这些更改应用到 Kubernetes 集群
5. 结合 GitLab 的代码托管和 CI/CD 功能，形成完整的 DevOps 工作流，提升了开发和运维的效率

### 设计job

#### job梳理

##### test

运行Golang项目的单元测试和集成测试，以确保代码的质量和正确性

```yml
test:
  stage: test
  script:
    - echo "Running tests..."
    - go mod tidy
    - go test ./... -v
```

##### security_check

进行安全扫描和检查，以发现代码中的潜在安全漏洞和问题

```yml
security_check:
  stage: security_check
  script:
    - echo "Running static code analysis..."
    - go get -u golang.org/x/lint/golint
    - golint ./...
    - echo "Running dependency scanning..."
    - curl -LO https://github.com/jeremylong/DependencyCheck/releases/download/v6.5.3/dependency-check-6.5.3-release.zip
    - unzip dependency-check-6.5.3-release.zip
    - ./dependency-check/bin/dependency-check.sh --project "test" --scan .
```

##### build

构建Docker镜像，并且上传到AWS ECR

```yml
build:
  stage: build
  image: $CONTIANER_TOOL_SET_IMAGE
  script:
    - echo "Build success"
    - cifab build-to-ecr -i $ECR_IMAGE_URI -t $CI_COMMIT_TAG -d $CI_PROJECT_DIR/Dockerfile -r us-east-1
    - cifab build-to-ecr -i $ECR_IMAGE_URI_CN -t $CI_COMMIT_TAG -d $CI_PROJECT_DIR/Dockerfile -r cn-north-1
  only:
    - tags
```

##### deploy

修改deployment.yml文件，并且提交修改，这里为手动触发

```yml
deploy:
  stage: deploy
  only:
    - tags
  script:
    - echo "Deploying to production"
    - sed -i "s|ECR_IMAGE_URI_PLACEHOLDER|$CI_REGISTRY_IMAGE:latest|g" deployment/deployment.yaml
    - git config --global user.email "test_work"
    - git config --global user.name "test_user"
    - git add deployment/deployment.yaml
    - git commit -m "Update deployment image to $CI_REGISTRY_IMAGE:latest"
    - git push origin HEAD:main
  when: manual
```
