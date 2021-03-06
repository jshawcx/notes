### pod
```
pod 做为container的载体,container里的应用可以分为四种
长期运行的(long-running),批处理(batch),节点后台支撑(node-daemon),有状态(stateful-app)
控制器:deployment,job,daemonSet,statefulSet
```
- pod是kube创建和管理的最小单元,并且每个pod都有个pause容器(根容器)和init容器
- 容器依附与pod,一个pod可以只有一个container 也可以有多个container
- 一个pod内的container共享volume,network,namespace/ip,port
- 每个pod都有独立的ip,共享网络空间,包括ip地址和端口,pod内的container通过localhost互相访问
- pod 可以分为两类,普通pod和静态pod
- endpoint: pod-ip+port 为一个endpoint,一个pod 可以有多个endpoint
- 每个pod 可以有单独的资源配额`基本是cpu和memory` 通过spec.containers.resources
- 一般不单独创建pod,而是通过controller
- pod 可以理解为是多个进程的聚合(每个container都是一个或多个进程)
- 支持水平的扩展和缩容

#### 不建议在一个容器中运行多个应用
- 违背了docker的初衷,一个实例或一个进程一个容器
- 解藕依赖,没个进程单独的部署和编译,版本控制
- 透明,保持pod中的容器对基础设施可见,以便基础设施更好的服务pod
- 把 一些管理任务交给基础架构,container保持轻量,方便管理

#### pod删除
- 强制删除
  - kubectl delete pod k8s-demo --force --grace-peroid=0
  - yaml文件.spec.spec.terminationGracePeriodSeconds
  - api-service 不会等待各个节点kubelet的确认,会立即从api-service中删除pod
  - pod状态会被立即设置为terminating状态
  - 不建议使用
- 有等待期的删除(默认)
  - kubectl delete pod k8s-demo
    - pod状态为terminating
    - 如果pod超时,状态为dead
    - 如果pod中使用了hook 会在pod停止前被调用,如果在宽限其还没有执行完,会增加2秒宽限期
  - 发送term请求到每个容器的主进程
  - 从service的端点列表中删除pod
  - 一旦超时会直接发送kill给主进程,并从api-service删除
  - 如果 kubelet 或controller-manageer在等待期中重启,会重新执行一个等待期

#### 其他配置:
- spec.containers.command
- spec.containers.env
- spec.containers.ports.containerPort/hostIp/hostPort/protocol

#### pause容器
- pause容器的状态代表整个容器的状态,其他容器共享pause的ip(pod ip),volume
- 每个pod 里都有一个该容器
- 该容器代表了pod的状态

#### init容器
- 专用容器,在所有应用容器启动前启动
- 在pod启动过程中,init容器会按顺序在网络和volume初始化成功后启动
- 如果pod重启,init容器会重新执行,更改init容器的image等于重启pod
- 如果启动失败会,kube会不停的重启,直到init成功,如果pod对应的策略是never,那么失败了不会重启
- yaml文件中通过template.spec.initContainers来定义,可以有多个,但是按照顺序依次执行
- 和普通容器一致,全部字段和特性,但是不支持readiness probe (探针),因为他在pod就绪之前完成
- 可以用来将应用镜像分离出创建和部署两个,减少image的复杂度
- 应用容器是并行运行的,init容器可以提供简单的堵塞或延迟应用容器的启动方法,直到满足了一组必要条件

#### 静态pod

- 静态pod由kubelet管理,仅存在于特定的node上,不能通过apiservice管理
- 通过启动kubelet时带上--config 参数 指定需要使用的pod yaml文件存放路径

#### 基本命令

- kubectl create -f xxx.yaml
- kubectl get pod pod-name [-o wide,yaml,json]
- kubectl describe pod pod-name --namespace=ns
- kubectl get pods --show-labels
- Kubectl delete pod po-name
- kubectl delete pod --all
- kubectl replace xxx.yaml

#### pod生命周期

- pending pod已经被创建,但是部分container还没有创建好
- running 所有container已经创建好,至少有一个container在运行状态,启动状态,正在启动状态
- successed 所有container已经执行成功并退出,不会在重启
- failed 所有container全部执行完成,至少有一个容器退出状态为失败
- unknown 由于各种原因,无法获取pod状态

#### 探针

由kubelet来定期检查pod,有三种方式(由container实现,kubelet调用)来检查

- execAction
- tcpSocketAction
- httpGetAction

三种结果:成功,失败,未知

探针类型:

> livenessProbe(存活探针)

- 表明容器是否正在运行 pod.status:running
- 如果探针失败,kube会kill掉container,并且会受restartPolicy影响,决定是否重启
- 如果容器没有提供探针,那么kubelet会认为探针的返回值永远是成功的

> readinessProbe(就绪探针)

- 表明容器是否可以正常接受请求 container.ready:true
- 如果探针失败,端点控制器(endpoint-controller)会从与pod匹配的所有service中的端点删除该pod-ip
- 直到下次探测成功

区别:

- livenessProbe kill 掉容器,并根据策略作出决定
- readinessProbe 将pod-ip从endpoint 列表中删除
- rp用于探测容器是否就绪,如果未就绪不会把流量转给pod,比如pod已经启动状态为running,但容器内应用程序还没有启动成功,如果没有rp,kube会认为他可以处理请求了,但此时并不能处理请求

```
containers:
- name: nginx
  image: nginx:1.15
  command: ["/bin/bash",'-c'," touch /tmp/healthy; sleep 30; rm -rf /tmp/healthy; sleep 60"]
  livenessProbe:
    exec:
      command: ["cat","/tmp/healthy"]
    initialDelaySeconds: 5 #在第一次检查之前等待5秒,如果执行成功,返回0,kubelet会认为成功,反之失败,kill掉,根据策略绝对是否重启container
    periodSeconds: 5       #每5秒检查一次
```
```
发送 get 请求,访问8080端口下的 /healthz,大于等于200,小于400的 认为成功,反之失败
livenessProbe:
  httpGet:
    path: /healthz
    port: 8080
    httpHeaders:
    - name: x-cluster-header
      value: test
  initialDelaySeconds: 5
  periodSeconds: 5

```
```
如果容器的8080端口能访问,认为成功
readinessProbe:
  tcpSocket:
    port:8080
  initialDelaySeconds: 5
  periodSeconds: 5
```
```
ports:
- name: liveness-port
  containerPort: 8080
  hostPort: 8080
livenessProbe:
  httpGet:
    path: /healthz
    port: liveness-port
```

>  httpGet控制选项:

- host,默认pod-ip,可以通过在http request中设置host
- scheme,链接host的协议,默认http
- path, http服务上的url
- httpHeaders,自定义http请求头部,
- port,http服务上的端口或名称

```
initialDelaySeconds：Pod启动后延迟多久才进行检查，单位：秒。
periodSeconds：检查的间隔时间，默认为10，单位：秒。
timeoutSeconds：探测的超时时间，默认为1，单位：秒。
successThreshold：探测失败后认为成功的最小连接成功次数，默认为1，在Liveness探针中必须为1，最小值为1。
failureThreshold：探测失败的重试次数，重试一定次数后将认为失败，在readiness探针中，Pod会被标记为未就绪，默认为3，最小值为1。
```

#### pod重启策略

- Always
- OnFailure
- Never

#### pod 管理
- 由rc和daemonset管理的pod,template.spec.restartPolicy 必须一直是always
- 由job管理的pod 只能为onFailure 和never
- 由kubelet 管理的pod(静态) 不受策略影响,pod失效时会重启,并且不会对pod进行健康检查

#### hook
- 由kubelet发起,在容器中的进程工作前和终止之前执行,包含在容器的生命周期之中
- 包括exec和http 两种
- 在容器创建之后,容器的entrypoint 执行之前,这是pod已经被调度到node上,被某个kubelet管理
- kubelet调用postStart,该命令和容器的启动命令是异步执行,在postStart执行完成之前,kubelet会锁住容器,不让应用容器启动,此时pod状态为pending,成功后pod为running
- 如果postStart和preStop执行失败,那么终止容器
```
spec:
  containers:
  - name: nginx-srv
    image: nginx:1.15
    lifecycle:
      postStart:
        exec:
          command: ["/bin/bash",'-c',"echo test"]
      preStop:
        httpGet:
          path: "/test"
          port: 8080
          httpHeaders:
          - name: x-cluster-header
            value: test
```


#### 创建流程

- 开发者开发一个应用,打包docker 镜像,上传到镜像仓库中
- 编写一个deployment.yaml的文件
- 通过kubectl或其他方式提交(调用apiserver中的deployment接口)
- apiserver将部署需求更新到etcd中(apiserver任务完成,只记录,不做具体任务,具体操作由
controller-manager完成)
- deployment-controller通过监听apiserver,得知需要创建一个对象,会调用apiserver提供的replicaset操作接口
- replicaset-controller通过监听apiserver,得知需要创建一个对象,会创建一个pod
- scheduler通过监听apiserver,得知有一个新的pod被创建,经过计算,会将该pod分配到合适的node上
- kubelet通过监听apiserver,得知有个pod分配过来,会根据node情况,创建该pod或拒绝

[pod驱逐细节](/image/node_eviction.png)

[结构图](/image/pod.png)