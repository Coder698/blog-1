上次我们讲到目前的聊天应用通过 docker compose 和 rancher compose 的yml文件的形式交付和描述。当我们开发完代码通过本地的单元测试最终提交到中央代码仓库后，剩下构建和推送镜像，部署平台如何获得构建情况拉取最新的代码和镜像，并且完成 upgrade 滚动升级，并且把升级的情况和新代码运行后的情况及时的告知开发者，甚至升级失败后如何快速回滚。这里面涉及不少步骤，但是通过 gitlab 和 rancher cli & webui 提供的完善功能和钩子，我们可以使得该流程足够顺畅和高效.

下面给出目前聊天应用（改造中）的一些实践和记录。

最终实现效果就是在命令行提交代码后，等待几十秒后（gitlab 构建通知 rancher 部署），自动在命令行实时看到新代码运行后的 log 聚合，从而决定 finish upgrade 或 rollback 

### 应用描述

应用通过如下两份yml文件描述。

docker compose 描述 stack 由于哪些服务组成，它们之间的连接link关系如何。
rancher compose 描述服务在 rancher agents 之间的部署情况（如扩缩容实例数量，健康检查的策略，LB的haproxy配置等）

```shell
head -n 10 dev/*
==> dev/docker-compose.yml <==
version: '2'
services:
  gemini:
    image: docker.gf.com.cn/clickeggs/gemini:latest
    environment:
      DB_CONNECTION_STRING: mongodb://mongo:27017/gf-aws-storage
      NODE_ENV: dev
      HTTP_PORT: '8080'
    external_links:
    - parrot-data-stack/mongo:mongo

==> dev/rancher-compose.yml <==
version: '2'
services:
  gemini:
    scale: 1
    start_on_create: true
  socketio-lb:
    scale: 1
    start_on_create: true
    lb_config:
      certs: []
 sivagao@gfmac-siva  ~/projects/gf/clickeggs/gf-parrot/stack-compose ls
data  dev  infra-statsd-graphite-grafana  lb  rancher  readme.txt
```

![](media/14882714600496.jpg)


Tip for LB haproxy:

http://chattest.gf.com.cn/admin?stats （basic auth: admin/gf123456）

upstreams: LB对背后的服务实例的可用性和质量进行监控
从而可以及时的移除unhealth和告警，
同时对于scale up 扩容的实例进行加入

![](media/14882716888420.jpg)


### 代码提交

pre check，参考 [](https://github.com/gaohailang/node-party-gf-security-practice#入库检查)

入库检查

```shell
npm run precommit : npm test
npm run prepush : npm-run-all lint test test:deps
npm run inspect : jsinspect
```

使用 hurky 可以修改你的git命令，提供hook点在commit/push/merge 前执行检查。 譬如我们利用npm的hook（pre)在提交commit前运行下我们的单元测试等（在那些），在推送代码仓库前，执行我们的lint检查是否良好的代码格式等等。 甚至我们可以运行inspect，看看我们是否有存在代码的copy&paste这种情况。

说到测试，很多人说项目很赶没时间啊。还有人有些测试用例写起来还naive，不想写。其实我们并不需要对所有代码做测试。尤其在迭代速度很块的情况下很多需求没理清楚，说不定一些前天的在明天就要remove掉。 那么我们会集中在如下部分：

对这些进行测试：

- 那些经常要被修改的 change a lot
- 那些有很高复杂度的 risky
- 那些重要功能/常被查看的 more traffic



### 代码构建

代码顺利提交到 gitlab 后，gitlab ci 通过读取项目更目录的 ci yml 文件，在 gitlab runner 中运行文件中描述的构建和发布等持续集成的指令，详细情况可以通过 gitlab pipeline 子页面查看 

在 gitlab ci 的yml 文件中，暂时只用了两个 stage 来表述发布构建流程。
其中 Build 用于构建用于整个应用的多个Docker镜像并且进行推送到 gf 内部的 docker registry 中国年。
同时 Deploy 用于告知我们应用的运行平台 Rancher 去拉取最新构建好的服务进行发布。

http://gitlab.gf.com.cn/clickeggs/gf-parrot/pipelines/6171/builds

![](media/14882719182746.jpg)


构建的 yml 配置如下：

```yml
stages:
  - publish
  - deploy

publish_plugin_legality_image:
  stage: publish
  only:
    - develop
  tags:
    - eagle.build
  script:
    - cd parrot
    - cp ./docker/Dockerfile.legality Dockerfile
    - docker build -t docker.gf.com.cn/clickeggs/parrot-legality-refactor .
    - docker push docker.gf.com.cn/clickeggs/parrot-legality-refactor
```


### 代码发布

当我们的代码构建完成，并且推送完打包好的镜像后。可以让 Gitlab CI Runner 通过授信的请求来告知 Rancher Server （通过 WebHook，API）去新更新的镜像。这样新的测试环境就可以及时自动连贯的得到更新。

敏感信息如环境变量或 CI yml 中的变量可以通过 gitlab webui 添加，而不是明文写在文件中。

http://gitlab.gf.com.cn/clickeggs/gf-parrot/variables

![](media/14882723710113.jpg)


Rancher 具体 API 如下：
http://10.2.130.99:8123/v2-beta

![](media/14882722775339.jpg)



### 代码更新

具体的 deploy task 如下：

```yml
deploy:
  stage: deploy
  image: docker.gf.com.cn/gaohailang/rancher-gitlab-deploy
  script:
    - upgrade --stack gf-chat --service parrot-rest
```

发布任务的运行环境通过提供好的内部镜像使用。传入合适的参数：

PS：
因为 rancher-gitlab-deploy 这个脚本只能对应 stack(group) , service(repo name)，所以限定为 gf-chat parrot-rest。同时在rancher中把 socket-lb 统一打到 parrot-rest 上。以后更新parrot-rest 就可以生效了~

内部的逻辑就是通过发送请求和加上之前配置的 Key/Token 等信息

![](media/14882725923226.jpg)


我们可以在命令行运行rancher-cli 及时得知服务的升级和发布情况。 `rancher logs -f -t -s gf-chat/parrot-rest`。
此时 gitlab ci 中 deploy task 对rancher server 调用 Upgrade 的具体升级信息会实时回显到命令行中，方便查看和连贯操作。

![](media/14882732338204.jpg)



### 代码运行

此时我们测试环境的代码已经是最新的了，通过测试同学或者运行自己 e2e/integration test 来验证目前最新代码的正确性。这时候应用的logs对我们对服务的运行状况至关重要，同时由于现在应用不仅仅是单例（登录单台机器后 tail -f 就行），它是被 rancher server 根据 rancher agent 的负载情况和 scale ratio 动态分配和多实例水平扩展运行的。一个简易实时的 log aggregation 就非常重要。

通过 rancher cli 提供的log子命令就可以很容易实现：

rancher logs -f -t gf-chat/parrot-rest

此服务的多个实例产生的日志就会实施的aggregation按照时间回传到我们的命令行中。

![](media/14882735890112.jpg)


通过查看日志，我们可以得知我们预期的应用行为，从而觉得此次滚动升级是否成功。 

![](media/14882737314872.jpg)


![](media/14882737381242.jpg)




### 代码回滚

我们的升级是滚动的形式，逐个按照 batch size 的数量拉新，启动新实例，通过LB合理调配，把流量打入到新启动的实例上，旧应用实例容器处理 deactive 状态。

但是如果我们更新的代码和镜像不是按照预期的执行（有bug...），此时快速回顾是非常必要的。通常你在上一步骤看到不合适的log就可以，通过命令行回滚该次 Upgrade 操作了。

当然也可以通过 rancher WebUI 操作：

![](media/14882739459981.jpg)




### 总结

到此一个较为完整的workflow已经走完了，通过 gitlab 和 rancher 提供的功能和完善的钩子和扩展，我们可以将这个流程变得足够顺畅。so

