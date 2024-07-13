---
title: 写一个github action帮你自动部署项目
date: 2024/07/13 21:43
tags: github action
categories: 前端
---
现有项目的部署是手动部署的，主要是 上服务器 docker 打包 然后把docker 镜像推到目标服务器 然后 执行，现在boss让我用 git action设置一下自动部署 ，说干就干，我这里使用的是 自由服务器托管，所以需要在自己的服务器添加runner

## 服务器添加runner

action 本质是在你服务器自动运行脚本，所以需要一个运行时来进行运行

先需要在github 的 setting里面知道 action这个选项，里面有一个runners，你可以在里面选择你服务器的型号 会根据你所选的 给你不同的 运行脚本 来安装runner

![Untitled.png](https://s2.loli.net/2024/07/13/DM3fbzYR6NIXd7U.png)

按照指示的runner 步骤 登录服务器 按照步骤一步步运行runner，在执行config.sh时会让你输入 runner的名称 以及 runner的label，label 在后面写action的时候要用

## runner 的 后台执行

可以发现 当咱们执行 `./run.sh` 后 终端并没有退出这个任务，导致一直 无法做其他事情，这很不方便，所以需要把这个服务跑到 后台去，github 官方提供了这个脚本 ，ls 看一下有一个 [svc.sh](http://svc.sh) 脚本文件，按照官方文档进行服务启动 [将自托管的运行应用程序配置为服务 - GitHub 文档](https://docs.github.com/zh/actions/hosting-your-own-runners/managing-self-hosted-runners/configuring-the-self-hosted-runner-application-as-a-service)

```bash
# 安装服务
sudo ./svc.sh install
# 启动服务
sudo ./svc.sh start
# 检查服务状态
sudo ./svc.sh status
```

## 项目建立 action 脚本

接下来是在你的项目中建立action脚本，当执行action时 是在你的项目中提取脚本的，

在根目录下新建一个.github文件夹，文件夹下 再建一个workflows 文件夹，新建一个yml文件 这个名称可以随便起 如下

![Untitled _1_.png](https://s2.loli.net/2024/07/13/IVduEkX1DHLPnTh.png)

接下来就是写yml脚本了 这个官方文档比较清晰 ，可以按照官方文档写 ，如下 [GitHub Actions 文档 - GitHub 文档](https://docs.github.com/zh/actions)

- 写name，这个是你的action 名称，会展示到 git action的menu里 如下
    
    ```bash
    name: Master Build and Deploy
    ```
    
    ![Untitled _2_.png](https://s2.loli.net/2024/07/13/gbTOKWJslVBw2aX.png)
    
- 写事件，事件包含很多 比如 push ，pull_request，之类的等等 这个可以在这个文档里面看 https://docs.github.com/zh/actions/using-workflows/events-that-trigger-workflows
    
    咱们最常用的就是 push 以及 workflow_dispatch，workflow_dispatch代表的是 手动执行，手动执行没办法过滤分支，如下
    
    ```bash
    name: Master Build and Deploy
    on:
      workflow_dispatch:
      push:
        branches:
          - test-branch
    ```
    
    我们运行所有的分支手动执行，并且test-branch 分支 push的时候 也会执行这个 action
    
- 新建 jobs ，这个jobs 就是执行单元，执行单元里面可以有多个执行任务，执行任务里面可以有 多个 steps，steps里可以写多个 ，在写jobs时候我需要捋一下我们部署的流程，git action在自有服务器运行，是新建了一个_work文件夹，如下
    
    ![Untitled _3_.png](https://s2.loli.net/2024/07/13/DYZKga4ApfbRELe.png)
    
    所以这个文件夹类似于之前的服务器环境
    
    - 第一步肯定是拉代码，这个 github 有官方的action actions/checkout@v3
    - 第二步需要获取jfrog的token，因为所有项目docker images 是统一管理的，需要放到公司统一的地方，所以需要这个token
    - 第三步就是登录docker
    - 第四步设置node环境
    - 第五步安装前端依赖
    - 第六步docker build 并且push 镜像到jfrog
    - 第七步执行 helm 里的安装脚本
    
    这个是整个流程可以看到，这个流程完全是按照自己业务部署的流程来的，所以每个项目都有可能不一样，这里有一些点需要注意，比如第二步要拿到 jfrog的token ，这里需要用jforg的登录，登录需要你自己jforg的token key 和user name，这些对你来说是敏感信息，这时候就需要用到 **secret 了，具体看这个文档** [变量 - GitHub 文档](https://docs.github.com/zh/actions/learn-github-actions/variables#default-environment-variables)
    
- 设置secret 文档，找到setting
    
    ![Untitled _8_.png](https://s2.loli.net/2024/07/13/fiNQo7rGlWmyMsb.png)
    
    然后点击 secrets and variables，选择里面的action 设置你需要的私密变量
    
    ![Untitled _2_.png](https://s2.loli.net/2024/07/13/gbTOKWJslVBw2aX.png)
    
    设置过程中你需要写你 environment 的名称，这个在action 里也需要用
    
    写完后可以看到 `deploy-master.yml`文件如下
    
    ```yaml
    name: Master Build and Deploy
    on:
      workflow_dispatch:
      push:
        branches:
          - dev
    jobs:
        build:
            runs-on: [self-hosted, linux, x64, frontend] # 刚才设置服务器runner的label
            permissions: # 设置jobs的权限
                contents: read
                id-token: write
            environment: "action" # 刚才设置secret的名称
            env:
                DOCKER_USERNAME: ${{ secrets.DOCKER_USERNAME }} # 在secret设置的变量
                JFROG_API_KEY: ${{ secrets.JFROG_API_KEY }}
            steps:
                - name: Checkout Repo # 拉代码
                  uses: actions/checkout@v3
                  with:
                    persist-credentials: false
    
                - name: Read Version # 读取版本号
                  id: read_version
                  run: |
                    echo "BUILD_VERSION=$(node -p "require('./package.json').buildVersion")" >> $GITHUB_ENV
                
                - name: Get jfrog token #获取jfrog token
                  id: api_response
                  run: |
                    response=$(curl -L -X POST https://****.jfrog.io/artifactory/api/security/token \
                      -H "X-JFrog-Art-Api: ${{ env.JFROG_API_KEY }}" \
                      -H "Content-Type: application/x-www-form-urlencoded" \
                      -d "username=${{ env.DOCKER_USERNAME }}" | jq -r '.access_token')
                    echo "dockerToken=${response}" >> $GITHUB_ENV
    
                - name: Docker Login ## docker登录
                  uses: docker/login-action@v1
                  with:
                      registry: ****.jfrog.io
                      username: ${{ env.DOCKER_USERNAME }}
                      password: ${{ env.dockerToken }}
                    
                - name: Setup Node
                  uses: actions/setup-node@v2
                  with:
                    node-version: '16.18.0'
    
                - name: Install Dependencies # 执行前端的安装脚本
                  run: ./install.sh
                      
                - name: Build # 打包并push镜像
                  env:
                    DIRECTORY: test
                    IMAGENAME: webapp-iamges
                    REGISTRY: ****.jfrog.io
                  run: |
                    npm run prod:build
                    docker build . -t $IMAGENAME
                    docker image tag ${{ env.IMAGENAME }} ${{ env.REGISTRY }}/${{ env.DIRECTORY }}/webapp:${{ env.BUILD_VERSION }}
                    docker push ${{ env.REGISTRY }}/${{ env.DIRECTORY }}/webapp:${{ env.BUILD_VERSION }}
    
                - name: Install webapp # 安装服务
                  run: |
                    sed -i 's/appVersion:.*/appVersion: \"${{ env.BUILD_VERSION }}\"/' /home/code/helms/webapp/iso/helm/webapp/Chart.yaml
                    cd /home/code/helms/webapp/iso
                    ./installWeb.sh
    
                - name: Install alert # 安装服务
                  run: |
                    sed -i 's/appVersion:.*/appVersion: \"${{ env.BUILD_VERSION }}\"/' /home/code/helms/webapp/iso/helm/webapp/Chart.yaml
                    cd /home/code/helms/alert/iso
                    ./installAlert.sh
                    
    ```
    
    这个yml文件写的时候 按照文档写就行了，有一些点要注意，比如 run的过程中变量赋值，必须run完成后 才可能拿到变量
    

## 优化

其实写完后已经可以正常部署了，但是还可以优化，比如 前端每次都需要安装node_modules这个很耗时，还有这个yml文件太冗长，所以优化分为两步一个是 使用缓存，一个是拆分jobs

### 使用缓存

为什么只用在node_modules使用缓存，因为其他比如node 的设置，官方action 已经考虑了缓存，如下

![Untitled _5_.png](https://s2.loli.net/2024/07/13/dFErJzpTAMtx6Qb.png)

可以看到执行 setup node这步 会直接 found in cache

使用缓存 直接用 actions/cache@v3 这个官方action 就可以，具体可以看 这个文档 [缓存依赖项以加快工作流程 - GitHub 文档](https://docs.github.com/zh/actions/using-workflows/caching-dependencies-to-speed-up-workflows)，新增后的 （新增到 install Dependencies之前）yml文件如下

```yaml

            - name: Cache node_modules
              uses: actions/cache@v3
              with:
                path: node_modules # 缓存路径
                key: ${{ runner.os }}-node-${{ hashFiles('**/package.json') }} #缓存key
                restore-keys: |
                  ${{ runner.os }}-node- # 备用key
```

执行action 看了一下效果（第一次不会命中成功，需要执行两次 因为第一次没有存入缓存），第一次执行完成后，看到actions caches 选项发现已经有了一个chache

![Untitled _6_.png](https://s2.loli.net/2024/07/13/NFRrjqS7XvOZKLD.png)

第二次再看一下 action输出的终端信息

![Untitled _7_.png](https://s2.loli.net/2024/07/13/6NyPIB2fnMCvjTK.png)

可以发现使用了上次的缓存，但是 这个缓存读取非常慢，可以看到 0.5mb /s ，但是这个cache 要114mb，所以用时就是228s，相当于 快4分钟 比安装依赖还费时,

### 拆分jobs

在.github/workflows 下建一个 build 和 install文件夹，分别再新建action.yml 文件

从 setup node 这步开始 到 Build 这步 放到action.yml中

新建的 build/action.yml如下

```yaml
name: Build
description: Build the application

inputs: # 这个文件运行需要的参数
    build_version:
        description: 'The version of the build'
        required: true
        default: 'latest'
runs:
    using: 'composite'
    steps:
        - name: Setup Node
              uses: actions/setup-node@v2
              with:
                node-version: '16.18.0'

        - name: Cache node_modules
            uses: actions/cache@v3
            with:
            path: node_modules
            key: ${{ runner.os }}-node-${{ hashFiles('**/package.json') }}
            restore-keys: |
                ${{ runner.os }}-node-

        - name: Install Dependencies
            run: ./install.sh
                
        - name: Build # 打包并push镜像
              env:
                DIRECTORY: test
                IMAGENAME: webapp-iamges
                REGISTRY: ****.jfrog.io
              run: |
                npm run prod:build
                docker build . -t $IMAGENAME
                docker image tag ${{ env.IMAGENAME }} ${{ env.REGISTRY }}/${{ env.DIRECTORY }}/webapp:${{ env.BUILD_VERSION }}
                docker push ${{ env.REGISTRY }}/${{ env.DIRECTORY }}/webapp:${{ input.build_version }}
```

deploy-master.yml 修改如下

```yaml
- name: build
              uses: ./.github/workflows/build #文件夹需要从根目录开始
              with: #使用的输入参数
                build_version: ${{ env.BUILD_VERSION }}
```

接下来拆分install，install属于另外一个步骤了，所以新建一个jobs，如下

```yaml
 build:
				...
        outputs:
            build_version: ${{ steps.read_version.outputs.BUILD_VERSION }}
        steps:
        ...
					
            - name: Read Version
              id: read_version
              run: |
                echo "BUILD_VERSION=$(cat ./package.json | jq -r '.buildVersion')" >> "$GITHUB_OUTPUT"
 install:
				needs: build # 需要build job完成才可以执行这个job
        runs-on: [self-hosted, linux, x64, frontend]
        permissions:
            contents: read
            id-token: write
        environment: "action"
        steps:
            - name: Install all
              uses: ./.github/workflows/install #使用文件
              with:
                build_version: ${{ needs.build.outputs.build_version }} #使用build jobs传出的参数
```

而install/action.yml如下 inputs需要使用build_version

```yaml
name: Install all app
description: Install all the applications

inputs:
    build_version:
        description: 'The version of the build'
        required: true
        default: 'latest'
runs:
    using: 'composite'
    steps:
        - name: Install webapp
            run: |
            sed -i 's/appVersion:.*/appVersion: \"${{ inputs.build_version }}\"/' /home/app/helms/ccp-webapp/iso/helm/webapp/Chart.yaml
            cd /home/app/helms/webapp/iso
            ./installWebapp.sh

        - name: Install alert
            run: |
            sed -i 's/appVersion:.*/appVersion: \"${{ inputs.build_version }}\"/' /home/app/chelms/ccp-alert/iso/helm/alert/Chart.yaml
            cd /home/app/helms/alert/iso
            ./installAlert.sh
```

这样就完成了分离