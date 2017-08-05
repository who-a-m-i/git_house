# ==docker swarm==

- **==楔子==**
> Docker集群管理和编排的特性是通过SwarmKit进行构建的， 其中Swarm mode是Docker Engine内置支持的一种默认实现。Docker 1.12以及更新的版本，都支持Swarm mode，我们可以基于Docker Engine来构建Swarm集群，然后就可以将我们的应用服务（Application Service）部署到Swarm集群中。创建Swarm集群的方式很简单，先初始化一个Swarm集群，然后将其他的Node加入到该集群即可。


- 基本特性
> **==集群管理集成进Docker Engine==**
使用内置的集群管理功能，我们可以直接通过Docker CLI命令来创建Swarm集群，然后去部署应用服务，而不再需要其它外部的软件来创建和管理一个Swarm集群。

> **==去中心化设计==**
Swarm集群中包含Manager和Worker两类Node，我们可以直接基于Docker Engine来部署任何类型的Node。而且，在Swarm集群运行期间，我们既可以对其作出任何改变，实现对集群的扩容和缩容等，如添加Manager Node，如删除Worker Node，而做这些操作不需要暂停或重启当前的Swarm集群服务。

> **==声明式服务模型（Declarative Service Model）==**
在我们实现的应用栈中，Docker Engine使用了一种声明的方式，让我们可以定义我们所期望的各种服务的状态，例如，我们创建了一个应用服务栈：一个Web前端服务、一个后端数据库服务、Web前端服务又依赖于一个消息队列服务。

> **==服务扩容缩容==**
对于我们部署的每一个应用服务，我们可以通过命令行的方式，设置启动多少个Docker容器去运行它。已经部署完成的应用，如果有扩容或缩容的需求，只需要通过命令行指定需要几个Docker容器即可，Swarm集群运行时便能自动地、灵活地进行调整。

> **==协调预期状态与实际状态的一致性==**
Swarm集群Manager Node会不断地监控集群的状态，协调集群状态使得我们预期状态和实际状态保持一致。例如我们启动了一个应用服务，指定服务副本为10，则会启动10个Docker容器去运行，如果某个Worker Node上面运行的2个Docker容器挂掉了，则Swarm Manager会选择集群中其它可用的Worker Node，并创建2个服务副本，使实际运行的Docker容器数仍然保持与预期的10个一致。

> **==多主机网络==**
我们可以为待部署应用服务指定一个Overlay网络，当应用服务初始化或者进行更新时，Swarm Manager在给定的Overlay网络中为Docker容器自动地分配IP地址，实际是一个虚拟IP地址（VIP）。

> **==服务发现==**
在Swarm内部，可以指定如何在各个Node之间分发服务容器（Service Container），实现负载均衡。如果想要使用Swarm集群外部的负载均衡器，可以将服务容器的端口暴露到外部。

> **==安全策略==**
在Swarm集群内部的Node，强制使用基于TLS的双向认证，并且在单个Node上以及在集群中的Node之间，都进行安全的加密通信。我们可以选择使用自签名的根证书，或者使用自定义的根CA（Root CA）证书。

> **==滚动更新(Rolling Update)==**
对于服务需要更新的场景，我们可以在多个Node上进行增量部署更新，Swarm Manager支持通过使用Docker CLI设置一个delay时间间隔，实现多个服务在多个Node上依次进行部署。这样可以非常灵活地控制，如果有一个服务更新失败，则暂停后面的更新操作，重新回滚到更新之前的版本。

> **==基本架构==**
Docker Swarm提供了基本的集群能力，能够使多个Docker Engine组合成一个group，提供多容器服务。Swarm使用标准的Docker API，启动容器可以直接使用docker run命令。Swarm更核心的则是关注如何选择一个主机并在其上启动容器，最终运行服务。


> **==Docker Swarm基本架构，如下图所示（来自网络，详见后面参考链接）：==**
![image](http://upload-images.jianshu.io/upload_images/1447734-182d37292620beca.png!web?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


>如上图所示，Swarm Node表示加入Swarm集群中的一个Docker Engine实例，基于该Docker Engine可以创建并管理多个Docker容器。其中，最开始创建Swarm集群的时候，Swarm Manager便是集群中的第一个Swarm Node。在所有的Node中，又根据其职能划分为Manager Node和Worker Node，具体分别如下所示：

> **==Manager Node==**
Manager Node负责调度Task，一个Task表示要在Swarm集群中的某个Node上启动Docker容器，一个或多个Docker容器运行在Swarm集群中的某个Worker Node上。同时，Manager Node还负责编排容器和集群管理功能（或者更准确地说，是具有Manager管理职能的Node），维护集群的状态。需要注意的是，默认情况下，Manager Node也作为一个Worker Node来执行Task。Swarm支持配置Manager只作为一个专用的管理Node，后面我们会详细说明。

> **==Worker Node==**
Worker Node接收由Manager Node调度并指派的Task，启动一个Docker容器来运行指定的服务，并且Worker Node需要向Manager Node汇报被指派的Task的执行状态。



```

构建Swarm集群
manager：172.30.40.172
worker ：172.30.40.173
worker ：172.30.40.174
worker ：172.30.40.175

```


> 启动docker daemon进程:

```
systemctl start docker
```

> 创建Swarm集群（在manager机器上创建）：

```
命令格式：
docker swarm init --advertise-addr <MANAGER-IP>
```
```
docker swarm init --advertise-addr 172.30.40.172
```
> 上面–advertise-addr选项指定Manager Node会publish它的地址为192.168.1.107，后续Worker Node加入到该Swarm集群，必须要能够访问到Manager的该IP地址。可以看到，上述命令执行结果，如下所示：

```
Swarm initialized: current node (j1tuf3w3buoez1h7oppuhs2z7) is now a manager.

To add a worker to this swarm, run the following command:

    docker swarm join \
    --token SWMTKN-1-2gqcdviukzwkm9mu2q0xxrrqj1ry012lbrbx52lth1w2ry21it-67uovyx0puevmghbfowggj4fb \
    172.30.40.172:2377

To add a manager to this swarm, run 'docker swarm join-token manager' and follow the instructions.
```

> 该结果中给出了后续操作引导信息，告诉我们如何将一个Worker Node加入到Swarm集群中。也可以通过如下命令，来获取该提示信息：

```
docker swarm join-token worker
```
```
To add a worker to this swarm, run the following command:

    docker swarm join \
    --token SWMTKN-1-2gqcdviukzwkm9mu2q0xxrrqj1ry012lbrbx52lth1w2ry21it-67uovyx0puevmghbfowggj4fb \
    172.30.40.172:2377
```
 > 在任何时候，如果我们需要向已经创建的Swarm集群中增加Worker Node，只要新增一个主机（物理机、云主机等都可以），并在其上安装好Docker Engine并启动，然后执行上述docker swarm join命令，就可以加入到Swarm集群中。
 
 > 查看当前Manager Node的基本信息：
 
 ```
 docker info
 ```
 ```
  Swarm: active
 NodeID: j1tuf3w3buoez1h7oppuhs2z7
 Is Manager: true
 ClusterID: 7i61yfvniniubd2m2wb524i04
 Managers: 1
 Nodes: 1

 ```
 > 可以看出，目前Swarm集群只有Manager一个Node，而且状态是active。也可以在Manager Node上执行``` docker node ls ``` 命令来查看Node状态：
 ```
 ID                           HOSTNAME                     STATUS  AVAILABILITY  MANAGER STATUS
j1tuf3w3buoez1h7oppuhs2z7 *  shata-ops-test-01.novalocal  Ready   Active        Leader

 ```
 
 > 添加（3台）Worker Node到集群中：
 ```
 docker swarm join  --token SWMTKN-1-2gqcdviukzwkm9mu2q0xxrrqj1ry012lbrbx52lth1w2ry21it-67uovyx0puevmghbfowggj4fb     172.30.40.172:2377
 
 ```
 > 查看当前集群信息：
 ```
 docker node ls
 
 ```
 ```
 ID                           HOSTNAME                     STATUS  AVAILABILITY  MANAGER STATUS
ddov2lcsb9y5pb165o1w0o49m    shata-ops-test-04.novalocal  Ready   Active
j1tuf3w3buoez1h7oppuhs2z7 *  shata-ops-test-01.novalocal  Ready   Active        Leader
rjbucrsbj9t9r3jz286h45ycv    shata-ops-test-03.novalocal  Ready   Active
u2d19t9q4ge8yv8z7754upop4    shata-ops-test-02.novalocal  Ready   Active

 ```
 > 上面信息中，AVAILABILITY表示Swarm Scheduler是否可以向集群中的某个Node指派Task，对应有如下三种状态：
 ```
Active：集群中该Node可以被指派Task
Pause：集群中该Node不可以被指派新的Task，但是其他已经存在的Task保持运行
Drain：集群中该Node不可以被指派新的Task，Swarm Scheduler停掉已经存在的Task，并将它们调度到可用的Node上
 ```
 
> 查看集群中Node状态信息，本机可使用self，其他node，可根据ID or HOSTNAME 进行查看：
```
dcoker node inspect self
docker node inspect ID|HOSTNAME
```
