## 简易型 Job 编排 Operator

### 项目思路与设计
设计背景：本项目实现简易型 Job 编排 Operator

### JobFlow
功能：k8s 当中原生的 Job 资源对象执行时，并没有相互依赖的编排特性(ex: Job a 完成后 -> 再执行Job b ...)。
在此需求上，基于 k8s 的扩展功能，实现 JobFlow 的自定义资源控制器，实现一个能执行多 Job 树状引擎的 operator 应用。


![](./image/jobflow.png?raw=true)

- crd 资源对象如下，更多信息可以参考 [参考](yaml/jobflow/example.yaml)
    - globalParams: 全局参数，会自动渲染到每个 job 中
    - name: flow 名称，多个 flow 名称不能重复
    - dependencies: 定义依赖项，如果有多个依赖可以填写多个
    - jobTemplate: job 模版，支持 k8s 原生 job spec 全部字段
```yaml
apiVersion: api.practice.com/v1alpha1
kind: JobFlow
metadata:
  name: jobflow-example
spec:
  # 可配置任务流中的全局参数，当设置后会在每个 job 与 pod 中都生效
  globalParams:
    # 可决定所有 job 都运行在同一节点上
    nodeName: minikube
    # 可加入 container 所需的参数
    env:
      - name: "FOO"
        value: "bar"
      - name: "QUE"
        value: "pasa"
    # job pod 的 annotations    
    annotations:
      key1: value1
      key2: value2
    # job pod 的 labels  
    labels:
      key1: value1
      key2: value2
      
  # 可填写多个 flow 流程
  # 每个 flow 中重要字段 分别为：
  # name: flow 名称，多个 flow 名称不能重复
  # dependencies: 定义依赖项，如果有多个依赖可以填写多个
  # jobTemplate: job 模版，支持 k8s 原生 job spec 全部字段
  flows:
    - name: job1
      dependencies: []
      jobTemplate:
        template:
          spec:
            containers:
              - image: busybox:1.28
                command:
                  - sh
                  - -c
                  - sleep 10s
                imagePullPolicy: IfNotPresent
                name: nginx
    - name: job2
      jobTemplate:
        template:
          spec:
            containers:
              - image: busybox:1.28
                command:
                  - sh
                  - -c
                  - sleep 100s
                imagePullPolicy: IfNotPresent
                name: nginx
      dependencies:
        - job1  # 代表 job2 依赖 job1 完成后才开始启动
    - name: job3
      jobTemplate:
        template:
          spec:
            containers:
              - image: busybox:1.28
                command:
                  - sh
                  - -c
                  - sleep 100s
                imagePullPolicy: IfNotPresent
                name: nginx
      dependencies:
        # 代表 job3 依赖 job1 job2 完成后才开始启动
        - job1
        - job2
    - name: job4
      jobTemplate:
        template:
          spec:
            containers:
              - image: busybox:1.28
                command:
                  - sh
                  - -c
                  - sleep 10s
                imagePullPolicy: IfNotPresent
                name: nginx
    - name: job5
      dependencies:
        # 代表依赖 job2 job4 后才执行
        - job4
        - job2
      jobTemplate:
        template:
          spec:
            containers:
              - image: busybox:1.28
                command:
                  - sh
                  - -c
                  - sleep 10s
                imagePullPolicy: IfNotPresent
                name: nginx
```

### 项目功能
1. 支持 JobFlow 任务中的 job 依赖执行
2. 查看任务流状态
- 注：pod 字段中的 **restartPolicy**  不允许被使用，就算定义后也不会生效(都会被强制设为"Never")


### DaemonJob
功能：k8s 当中原生的 Job 资源对象执行时，并没有类似 pod 中的 daemonset 的工作负载。
在此需求上，基于 k8s 的扩展功能，实现 DaemonJob 的自定义资源控制器，实现一个能选定节点执行 job 的 operator 应用。


![](./image/daemonjob.png?raw=true)

- crd 资源对象如下，更多信息可以参考 [参考](yaml/daemonjob/example.yaml)
  - globalParams: 全局参数，会自动渲染到每个 job 中
  - excludeNodeList: 不执行 job 的节点名称，使用 "," 隔开
  - template: job 模版，支持 k8s 原生 job spec 字段
```yaml
apiVersion: api.practice.com/v1alpha1
kind: DaemonJob
metadata:
  name: daemonjob-example
spec:
  # 可配置任务流中的全局参数，当设置后会在每个 job 与 pod 中都生效
  globalParams:
    # 可加入 container 所需的参数
    env:
      - name: "FOO"
        value: "bar"
      - name: "QUE"
        value: "pasa"
    # job pod 的 annotations
    annotations:
      key1: value1
      key2: value2
    # job pod 的 labels
    labels:
      key1: value1
      key2: value2
  # 除这些节点之外，都要创建 job, 使用 "," 隔开
  excludeNodeList: node1, node2
  # 支持原生 job Template 模版
  template:
    spec:
      restartPolicy: OnFailure
      containers:
        - image: busybox:1.28
          command:
            - sh
            - -c
            - sleep 100s
          imagePullPolicy: IfNotPresent
          name: nginx
```

### 项目功能
1. 支持 DaemonJob 任务中的 job 执行
2. 查看任务流状态
- 注：pod 字段中的 **restartPolicy**  不允许被使用，就算定义后也不会生效(都会被强制设为"Never")

