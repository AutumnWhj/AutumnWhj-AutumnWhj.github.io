---
title: Github Actions的使用
description: GitHub Actions是github的持续集成服务，简单来讲就是使得你的repository可以执行一些自动化的动作
isOriginal: true
icon: ''
date: 2021-09-11
category:
  - 技术文章
  - CI/CD
tag:
  - ci/cd
# 是否置顶
sticky: false
# 是否收藏在博客主题的文章列表中。当填入数字时，数字越大，排名越靠前
star: false
# 是否将该文章添加至文章列表中。
article: true
# 是否将该文章添加至时间线中。
timeline: true
---
<CountView></CountView>

## 什么是Github Actions？
> Automate, customize, and execute your software development workflows right in your repository with GitHub Actions. You can discover, create, and share actions to perform any job you'd like, including CI/CD, and combine actions in a completely customized workflow.

GitHub Actions是github的持续集成服务，简单来讲就是使得你的repository可以执行一些自动化的动作，例如：

1. 每次repository有新的提交或者pr就自动执行自定义好的操作
1. 在repository中定时执行自定义好的操作

如在 master 分支上提交了一段代码， GitHub Action 可以自动的帮我部署到我自己的服务器上去，或者它还可以帮我把代码打成镜像，将镜像自动提交到镜像仓库里
### 术语和约束
在介绍使用方式之前我们先来了解下GithubActions的术语,借此了解下一次执行过程的流程.

- workflow即工作流,一次执行过程.每个workflow用一个配置文件维护.
- Job: workflow的分解,可串行存在依赖;可并行
- Step: job的分解,即步骤,比如一个step是要给代码做单元测试,那可能会有三个步骤:下载依赖->测试->上传结果
- actionworkflow最小执行单元.即每个执行步骤中的具体执行任务,我们可以自己定义action,也可使用Github社区定义好的action
- Artifact： workflow运行时产生的中间文件.包括日志,测试结果等
- Event: 触发workflow的事件

Github Action对workflow设有如下使用限制:

- 一个仓库可最多同时开20个workflows;超过20则排队等待
- 一个workflow下的每个job最多运行6小时,超过直接结束
- 所有分支下的job根据github级别不同有不同的并行度限制,超过并行度进入队列等待
- 1小时内最多1000次执行请求,也就是1.5api/1m

需要注意,Github对Github Action服务有最终解释权,也就是说乱用可能会被Github限制账户.Github也会生成相关使用统计情况
### workflow的触发
每个workflow的配置文件都需要定义on字段,它用来描述在何种情况(Event)下触发执行.我们可以定义on多种事件,这样**只要满足其中一个就会被触发**
我们可以将Event分为3类:

- 定时事件:由定时任务触发的事件
- 手动触发事件: 在actions页面中手动触发的事件
- Webhook事件:由github网站的钩子行为触发的事件,通常Git操作都有钩子可以用于触发
#### 定时事件
最简单的事件就是定时事件其定义方式如下:
on:   schedule:     _# * is a special character in YAML so you have to quote this string_     - cron:  '*/15 * * * *' 
上面定义了一个每隔15分钟执行依次的任务.Github Avtion目前只支持[crontab语法定义定时任务](https://pubs.opengroup.org/onlinepubs/9699919799/utilities/crontab.html#tag_20_25_07)
这个事件只会拉取默认分支(一般是master或者main分支,可以在仓库的settings->branches->Default branch下修改)的最近一次提交进行执行.
#### 手动触发事件
手动触发事件分为两种:

- workflow_dispatch 让用在Actions界面中手动触发workflow 当在workflow中定义了workflow_dispatch后管理页面就会允许指定这个workflow被手动执行,执行时默认需要指定分支,如果我们在配置中定义了参数,则手动执行时也会需要填参数.
- 一个典型例子如下:  
```yaml
  on:
      workflow_dispatch:
          inputs:
              name:
                  description: 'Person to greet'
                  required: true
                  default: 'Mona the Octocat'
              home:
                  description: 'location'
                  required: false
                  default: 'The Octoverse'

  jobs:
      say_hello:
          runs-on: ubuntu-latest
          steps:
          - run: |
              echo "Hello \$\{\{ github.event.inputs.name \}\}!"
              echo "- in \$\{\{ github.event.inputs.home \}\}!"

```

- 上面在workflow_dispatch下通过定义inputs设定参数.在jobs中我们则可以在github.event.inputs中取到对应的参数. **注意**如果不定义手动触发事件那么就无法手动触发.
- repository_dispatch让用户通过API批量手动执行这个event的主要作用是让其他的程序通过api调用,通过自定义事件类型来驱动执行.这个event对应的workflow必须在默认分支下定义.比如我们定义:
```yaml
  on:
      repository_dispatch:
          types: [opened, deleted]
```
然后执行http请求:
```bash
  curl \
  -X POST \
  -H "Accept: application/vnd.github.v3+json" \
  https://api.github.com/repos/{namespace}/{repo_name}/dispatches \
  -d '{"event_type":"opened"}'
```
那么就可以被执行了.其中的opened, deleted是用户自定义的事件.
#### Webhook事件
Webhook事件是借由Github的webhook事件触发的事件,具体有哪些可以看[官方文档](https://docs.github.com/en/free-pro-team@latest/actions/reference/events-that-trigger-workflows#webhook-events),本文将只介绍几个常用的和git操作相关的事件.

- create分支,tag创建时触发
- delete分支,tag删除时触发
- gollum仓库的wiki创建或者更新时触发
- push/pull_requestpush是当有对仓库的push操作时触发;pull_request则是在执行pull request中触发这两个事件可以额外限制:上面的限制都允许使用通配符做匹配,支持的通配符包括:pull_request默认的行为是在merge完成后处理merge后的那次提交中的代码. 我们还可以通过types: [...]字段指定细分事件类型,包括:
   - branches: [...]指定符合条件的分支触发
   - branches-ignore:[...]指定除符合条件的分之外都触发
   - tags:[...]指定符合条件的tag触发
   - tags-ignore:[...]指定除符合条件的tag外都触发
   - paths:[...]代码中有符合条件的路径就触发(至少有一个存在)
   - paths-ignore:[...]代码中不存在指定的路径则都触发(至少有一个不存在)
   - *: 表示匹配0个或多个非/字符
   - **: 表示匹配0个或多个字符.
   - ?: 表示匹配0个或者一个字符
   - +: 表示匹配至少一个字符
   - []: 表示匹配一个范围内的字符,比如[0-9a-f]表示数字和a到f间的字符可以匹配
   - !: 在匹配字符串的开头表示否,其他位置没有特殊含义
   - assigned被分派到某个issue时触发
   - unassigned删除分派时触发
   - labeled打标签时触发
   - unlabeled取消标签时触发
   - opened创建pull request时触发
   - edited编辑pull request时触发
   - closed关闭pull request时触发
   - reopened重新打开pull request时触发
   - synchronize同步pull request代码时触发
   - ready_for_review,pull request处于ready_for_review状态时触发
   - locked,锁定时触发
   - unlocked,解锁时触发
   - review_requested,code review结束时触发
   - review_request_removedcode review请求被删除时触发
- release当执行Github release时触发,使用的代码时release时打tag的代码,类似于pull_request,也可以通过types:[...]来指定细分事件.
   - published公开后执行
   - unpublished取消公开后执行
   - created 创建后执行
   - edited 编辑后执行
   - deleted 删除后执行
   - prereleased预发布后执行
   - released发布后执行
### 模板语法
我们可以看到上面例子中会有\$\{\{ ... \}\}这样的文字,这是Github Action定义的模板语法,其中...的部分可以是常数,上下文变量,运算符或者预定义的函数调用.
#### 常数
模板语法支持所有json支持的简单数据类型,也就是null,boolean,number,string.
#### 上下文变量
每次workflow执行都会带上几个上下文变量用于描述自己和传递参数.具体的可以看[官方文档](https://docs.github.com/en/free-pro-team@latest/actions/reference/context-and-expression-syntax-for-github-actions#contexts).这边只介绍几个常用的

- matrix,执行策略中定义的变量,每次执行每个key只会有一个取值
- env,workflow中env定义的变量
- github,通常用于获取仓库和分支的信息,比较值得关注的有:
   - github.repository 执行的仓库名,也就是{namespace}/{repo_name},如果只要repo_name,可以使用${GITHUB_REPOSITORY#*/}
   - github.ref工作流的分支或tag,分支为refs/heads/<branch_name>格式,tag是refs/tags/<tag_name>格式,如果只要tag名可以使用${GITHUB_REF/refs\/tags\//}
   - ${GITHUB_SHA::8}可以用于获得前8位的commit的id值
   - github.event.inputs由手动事件触发传入的参数
- secrets,项目或命名空间定义的账号密码信息,可以在项目的Settings->Secrets中设置,一般用于上传package或者docker镜像.
#### 运算符
Github Action支持如下运算符

| 运算符 | 描述 |
| --- | --- |
| () | 逻辑分组 |
| [ ] | 索引 |
| . | 属性解除参考 |
| ! | 非 |
| < | 小于 |
| <= | 小于或等于 |
| > | 大于 |
| >= | 大于或等于 |
| == | 等于 |
| != | 不等于 |
| && | 和 |
| \|\| | 或 |

可以看到这些运算符解百纳都是用于做谓词的.因此同擦汗给你都与if字段配合使用
```yaml
steps:
  ...
  - name: The job has failed
    if: $
```
GitHub Action进行的是宽松的等式比较,其原理是将不同类型的数据转换为数字进行比较: | 类型 | 结果 | | ——– | ——————————————– | | null | 0 | | true | 返回 1 | | false | 返回 0 | | 字符串 | 空字符串为0,符合数字格式的为对应数,否则为NaN | | Array | NaN,在为同一实例时才视为相等 | | Object | NaN,在为同一实例时才视为相等 |
注意,类似SQL中的NULL,一个 NaN 与另一个 NaN 的比较不会产生 true.
#### 函数
Github Action支持一些内置函数,比较有用的有:

- contains( search, item ),用于查看序列中是否存在元素
- startsWith( searchString, searchValue)/endsWith( searchString, searchValue),用于查看字符串中是否已特定字符串开头或者结尾
- format('Hello {0} {1} {2}', 'Mona', 'the', 'Octocat'),类似python中的string.format(),使用模板字符串拼接字符串结果
- join( array, optionalSeparator ),类似python中的join,用于拼接数组内容为字符串.
- 作业状态检查函数success()/always()/cancelled()/failure(),这类函数返回的是bool型数据,因此一般作为谓词与if联合使用
### 执行策略
执行策略在一级关键字strategy中定义.它用于规定执行器执行workflow的行为.主要包括

- matrix,定义执行矩阵,执行器会遍历矩阵执行作业,matrix中定义的值在执行时可以从上下文matrix中获取
- max-parallel(int)最大并行度
- fail-fast(bool,true)快速失败,任何matrix作业失败,GitHub将取消所有进行中的作业

上面的例子中我们定义了python-version: [3.6, 3.7, 3.8, 3.9],这也就意味着执行器会以matrix.python-version为3.6, 3.7, 3.8, 3.9分别执行一次.
### 使用社区定义好的action
可以将action理解为执行过程的封装,使用的人只需要知道它的用法而不需要知道它具体怎么实现的,我们可以自己定义action也可以使用外面定义好的action就像我们编程调用函数一样.社区的actions可以在[marketplace](https://github.com/marketplace?type=actions)找到
上面的例子中我们就使用了一个外部定义好的action:actions/setup-python@v2
使用action用关键字uses来声明,如果action需要参数可以使用with来传入参数
```yaml
 - name: Set up Python \$\{\{ matrix.python-version \}\}
    uses: actions/setup-python@v2
    with:
        python-version: \$\{\{ matrix.python-version \}\}
```
比较常用的action有:

- [actions/setup-python@v2](https://github.com/actions/setup-python),自动设置python环境
- [actions/setup-node@v1](https://github.com/marketplace/actions/setup-node-js-environment),设置node环境
- [actions/setup-go@v2](https://github.com/marketplace/actions/setup-go-environment),设置golang环境
- [docker/build-push-action@v1](https://github.com/marketplace/actions/docker-build-push-action),登录docker 镜像仓库
- [actions/upload-artifact@v2](https://github.com/marketplace/actions/upload-a-build-artifact),将Artifact发送到workflow的管理界面用于下载
- [getsentry/action-release@v1](https://github.com/marketplace/actions/sentry-release),发送消息到sentry
### jobs间的依赖关系
当我们单纯定义job时这些job会并行执行,而如果希望明确其中的依赖关系,则可以使用关键字needs.needs后的值可以是字符串也可以是字符串为元素的列表
```yaml
jobs:
  build_and_pub_to_pypi:
    ...
  docker-build:
    needs: build_and_pub_to_pypi

```
## 定时提醒到企业微信
在项目根目录文件夹`.github下面的workflows下创建xx.yml`写入如下代码：
```yaml
name: 3点几啦！喝杯奶茶先啦

# 触发条件
on: 
  # 触发条件1：main分支有提交时候触发
  push:
    branches:
      - main
  # 触发条件2：定时任务，每天15点触发
  schedule:
    - cron: "0 15 * * *"
jobs:
  drink_tea:
    runs-on: ubuntu-latest
    steps:
      - name: 这就给大家发消息
        run: |
          curl 'https://qyapi.weixin.qq.com/cgi-bin/webhook/send?key='yourKey'' \
          -H 'Content-Type: application/json' \
          -d '
          {
                "msgtype": "text",
                "text": {
                    "content": "都别给我愣着，赶紧滴干饭🍚"
                }
          }'


```
为方便测试，这里有两个触发条件，一个是代码main分支有更新时触发，一个是每天的15点触发。
```yaml
schedule中cron的值 "* * * * *"表示如下
┌───────────── 分钟 (0 - 59)
│ ┌───────────── 小时 (0 - 23)
│ │ ┌───────────── 日 (1 - 31)
│ │ │ ┌───────────── 月 (1 - 12 或 JAN-DEC)
│ │ │ │ ┌───────────── 星期 (0 - 6 或 SUN-SAT)
│ │ │ │ │
│ │ │ │ │
│ │ │ │ │
* * * * *

```
而后打开Actions就会开始运行任务。
![image.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/51c9b975f1994302b48b1037f92f6bbc~tplv-k3u1fbpfcp-zoom-1.image)
怎么看任务执行成功？ 绿了，且运行没报错
![image.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/43ad475bd7aa4bc9b57f6bf1d3b7e3ef~tplv-k3u1fbpfcp-zoom-1.image)
这时候企业微信就收到了干饭🍚的消息。（还可以帮行政小姐姐设置一些繁琐的每天自动提醒工作哟🐶）
![image.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/501b2c18f29d48f7897e4addc4b7f03d~tplv-k3u1fbpfcp-zoom-1.image)


## 定时自动提交commit
github上很简单的一个自动绿的项目，通过定时任务git方式向repository提交代码，使得 yml代码如下：
![image.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/35f422e471094f2c99cb238b5a075bd0~tplv-k3u1fbpfcp-zoom-1.image)
```yaml
name: ci

on:
  push:
    branches:
      - master
#  schedule:
#    - cron: "0 0 * * *"

jobs:
  autogreen:
    runs-on: ubuntu-latest
    steps:
      - name: Clone repository
        uses: actions/checkout@v2
      - name: Auto green
        run: |
          git config --local user.email "xxx@qq.com"
          git config --local user.name "Autumn"
          git remote set-url origin https://${{ github.actor }}:${{ secrets.GITHUB_TOKEN }}@github.com/${{ github.repository }}
          git pull --rebase
          git commit --allow-empty -m "a commit a day keeps your girlfriend away"
          git push
```
## 前端CI/CD pipeline
持续集成（Continuous Integration）是持续部署CD的第一步，持续集成可以理解为在分支代码更新时，自动运行eslint代码风格检查，单元测试以提前发现代码的错误，防止打包后导致系统崩溃等一系列问题。
这里通过Github Actions建立持续集成工作流workflow来实现前端项目的自动化测试以及发布到COS对象存储。
![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/330c589390c5443b8ae4bd84ff88c50b~tplv-k3u1fbpfcp-zoom-1.image)
```yaml
name: Lint the code, run tests, build and push OSS
on:
  push:
    branches:
      - develop
jobs:
  lint:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v2
      - uses: actions/setup-node@v1
        with:
          node-version: '14.x'

      - name: Install dependencies
        uses: bahmutov/npm-install@v1

      - name: Run linter
        run: yarn lint
  test:
    needs: lint
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v2
      - uses: actions/setup-node@v1
        with:
          node-version: '14.x'

      - name: Install dependencies
        uses: bahmutov/npm-install@v1

      - name: Run test:unit
        run: yarn test:unit
  build:
    needs: [lint, test]
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@master
      - uses: actions/setup-node@v2
        with:
          node-version: '14.18.0'

      - name: Install dependencies
        uses: bahmutov/npm-install@v1

      - name: Build
        run: yarn build

      - name: Get current time
        uses: 1466587594/get-current-time@v2
        id: current-time
        with:
          format: YYYYMMDDHHmmss
          utcOffset: '+08:00'
      - name: Use current time
        env:
          F_TIME: '${{ steps.current-time.outputs.formattedTime }}'
        run: echo "NOW=$F_TIME" >> $GITHUB_ENV

      - name: Get repository and branch name
        shell: bash
        run: |
          echo "BRANCH=$(echo ${GITHUB_REF#refs/heads/} | tr / -)" >> $GITHUB_ENV
          echo "REPOSITORY=$(echo ${GITHUB_REPOSITORY#threfo/} | tr / -)" >> $GITHUB_ENV
          echo "PRBRANCH=$(echo ${GITHUB_HEAD_REF} | tr / -)" >> $GITHUB_ENV

      - name: Upload COS
        uses: zkqiang/tencent-cos-action@v0.1.0
        with:
          args: upload -r ./dist/ /${{ env.REPOSITORY }}/${{ env.BRANCH }}/default
          secret_id: ${{ env.SECRET_ID }}
          secret_key: ${{ env.SECRET_KEY }}
          bucket:${{ env.BUCKET }}
          region:${{ env.REGION }}

      - name: Upload COS Backup release
        uses: zkqiang/tencent-cos-action@v0.1.0
        with:
          args: upload -r ./dist/ /${{ env.REPOSITORY }}/${{ env.BRANCH }}/${{ env.NOW }}
          secret_id: ${{ env.SECRET_ID }}
          secret_key: ${{ env.SECRET_KEY }}
          bucket:${{ env.BUCKET }}
          region:${{ env.REGION }}

```
## 总结
以上是Github Action初探，其实觉得Github Action在CI/CD上是大有可为的，CI\CD ：「持续集成（Continuous Integration）」、「持续交付（Continuous Delivery）」、「持续部署（Continuous Deployment）」。在公司中我看着文档用Github Action做了一些小尝试，后续也会分享出来给大家。以上有误的欢迎dalao们指正。
本次demo代码地址：[https://github.com/AutumnWhj/GithubAction](https://github.com/AutumnWhj/GithubAction) 
## 参考
[GitHub Actions文档](https://docs.github.com/en/actions)
[GitHub Actions 入门教程---阮一峰](https://www.ruanyifeng.com/blog/2019/09/getting-started-with-github-actions.html)
[使用GithubActions自动化工作流](https://blog.hszofficial.site/introduce/2020/11/30/%E4%BD%BF%E7%94%A8GithubActions%E8%87%AA%E5%8A%A8%E5%8C%96%E5%B7%A5%E4%BD%9C%E6%B5%81/)​


