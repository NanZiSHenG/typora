### Chart.yaml详解
```
# 按名称键入的注释列表 (可选)
annotations:       
  category: Analytics
# 
apiVersion: v2
# chart版本
appVersion: 0.18.0
# 依赖关系
dependencies:
# 使用远程helm仓库
- name: common
  repository: https://charts.bitnami.com/bitnami
  # 
  tags:
  - bitnami-common      # Tags 可以用来对 charts 的 启用/禁用 分组 (可选)
  version: 1.x.x
  enabled: true
  import-values: # (可选) ImportValues 保存源值到要导入父键的映射。每一项可以是一个字符串或一对子/父子列表项 (可选)
  alias:                  # 用于 chart 的别名。多次添加相同的 chart 时很有用 (可选)
# 适用本地charts/
# 如果values中的minio.enabled为true，才会部署子chart(在charts目录下)
- condition: minio.enabled      # minio是依赖的chart的名字
  # 子chart
  name: minio
  # 仓库地址
  repository: https://charts.bitnami.com/bitnami
  version: 6.x.x
# 描述
description: Thanos is a highly available metrics system that can be added on top
  of existing Prometheus deployments, providing a global query view across all Prometheus
  installations.
# chart 类型，application 和 library (可选)
type:                       
# 该chart的家目录地址
home: https://github.com/bitnami/charts/tree/master/bitnami/thanos
icon: https://bitnami.com/assets/stacks/thanos/img/thanos-stack-220x234.png
# 关键字列表，便于检索
keywords:
- analytics
- monitoring
- prometheus
- thanos
# 维护者信息
maintainers:
- email: containers@bitnami.com
  name: Bitnami
# chart name
name: thanos
# chart 下载列表
sources:
- https://github.com/bitnami/bitnami-docker-thanos
- https://thanos.io
# release 版本
version: 3.14.1
# chart 是否已弃用 (可选, boolean)
deprecated: false
```
### deployment.yaml例子1
```
apiVersion: extensions/v1beta1    #接口版本
kind: Deployment                 #接口类型
metadata:
  name: cango-demo               #Deployment名称
  namespace: cango-prd           #命名空间
  labels:
    app: cango-demo              #标签
spec:
  replicas: 3
   strategy:  #更新策略
    rollingUpdate:  ##由于replicas为3,则整个升级,pod个数在2-4个之间
      maxSurge: 1      #滚动升级时会先启动1个pod
      maxUnavailable: 1 #滚动升级时允许的最大Unavailable的pod个数
  selector:  #选择器
    matchLabels:
      app: cango-demo  #匹配pod标签
  template:   #pod模板       
    metadata:  #pod元数据
      labels:
        app: cango-demo  #pod模板名称标签，必填
    spec: #定义容器模板，该模板可以包含多个容器
      containers:                                                                   
        - name: cango-demo    #随意                                                         #镜像名称
          image: swr.cn-east-2.myhuaweicloud.com/cango-prd/cango-demo:0.0.1-SNAPSHOT #镜像地址
          command: [ "/bin/sh","-c","cat /etc/config/path/to/special-key" ]      #启动命令
          args: 
            #- '"-f","-h /data/web/html"'  #-f 放前台 -h 定义家目录                                                            #启动参数
            - '-storage.local.retention=$(STORAGE_RETENTION)'
            - '-storage.local.memory-chunks=$(STORAGE_MEMORY_CHUNKS)'
            - '-config.file=/etc/prometheus/prometheus.yml'
            - '-alertmanager.url=http://alertmanager:9093/alertmanager'
            - '-web.external-url=$(EXTERNAL_URL)'
            #如果command和args均没有写，那么用Docker默认的配置。
            #如果command写了，但args没有写，那么Docker默认的配置会被忽略而且仅仅执行.yaml文件的command（不带任何参数的）。
            #如果command没写，但args写了，那么Docker默认配置的ENTRYPOINT的命令行会被执行，但是调用的参数是.yaml中的args。
            #如果如果command和args都写了，那么Docker默认的配置被忽略，使用.yaml的配置。
          imagePullPolicy: IfNotPresent  #如果不存在则拉取
          livenessProbe:       #表示container是否处于live状态。如果LivenessProbe失败，LivenessProbe将会通知kubelet对应的container不健康了。随后kubelet将kill掉container，并根据RestarPolicy进行进一步的操作。默认情况下LivenessProbe在第一次检测之前初始化值为Success，如果container没有提供LivenessProbe，则也认为是Success；
            httpGet:
              path: /health #如果没有心跳检测接口就为/
              port: 8080  #或者http，由于定义了容器端口名称，可以通过名称应用
              scheme: HTTP
            initialDelaySeconds: 60 ##启动后延时多久开始运行检测
            timeoutSeconds: 5    #超时次数
            successThreshold: 1   #成功次数
            failureThreshold: 5   #失败次数
            readinessProbe:
          readinessProbe:   #表示是否就绪。如果没有就绪,就不会放在service后端,客户的请求就不会因找不到资源而报错了
            httpGet:
              path: /health #如果没有心跳检测接口就为/
              port: 8080
              scheme: HTTP
            initialDelaySeconds: 30    ##启动后延时多久开始运行检测
            timeoutSeconds: 5
            successThreshold: 1
            failureThreshold: 5
          resources:              ##CPU内存限制
            requests:
              cpu: 2
              memory: 2048Mi
            limits:
              cpu: 2
              memory: 2048Mi
          env:                    ##通过环境变量的方式，直接传递pod=自定义Linux OS环境变量
            - name: LOCAL_KEY       #本地Key
              value: value
            - name: CONFIG_MAP_KEY    #局策略可使用configMap的配置Key， 
            - name: REDIS_HOST
              value: redis.default.svc.cluster.local
              valueFrom:
                configMapKeyRef:
                  name: special-config     #configmap中找到name为special-config
                  key: special.type        #找到name为special-config里data下的key
          ports:
            - name: http  #定义了容器端口名称
              containerPort: 8080 #对service暴露端口
          volumeMounts:     #挂载volumes中定义的磁盘
          - name: log-cache
            mount: /tmp/log
          - name: sdb        #普通用法，该卷跟随容器销毁，挂载一个目录
            mountPath: /data/media    
          - name: nfs-client-root     #直接挂载硬盘方法，如挂载下面的nfs目录到/mnt/nfs
            mountPath: /mnt/nfs
          - name: example-volume-config    #高级用法第1种，将ConfigMap的log-script,backup-script分别挂载到/etc/config目录下的一个相对路径path/to/...下，如果存在同名文件，直接覆盖。
            mountPath: /etc/config       
          - name: rbd-pvc                #高级用法第2中，挂载PVC(PresistentVolumeClaim)

#使用volume将ConfigMap作为文件或目录直接挂载，其中每一个key-value键值对都会生成一个文件，key为文件名，value为内容，
  volumes:  # 定义磁盘给上面volumeMounts挂载
  - name: log-cache
    emptyDir: {}
  - name: sdb  #挂载宿主机上面的目录
    hostPath:
      path: /any/path/it/will/be/replaced
  - name: example-volume-config  # 供ConfigMap文件内容到指定路径使用
    configMap:
      name: example-volume-config  #ConfigMap中名称
      items:
      - key: log-script             #ConfigMap中的Key
        path: path/to/log-script  #指定目录下的一个相对路径path/to/log-script
      - key: backup-script          #ConfigMap中的Key
        path: path/to/backup-script  #指定目录下的一个相对路径path/to/backup-script
  - name: nfs-client-root           #供挂载NFS存储类型
    nfs:
      server: 10.42.0.55            #NFS服务器地址
      path: /opt/public             #showmount -e 看一下路径
  - name: rbd-pvc                   #挂载PVC磁盘
    persistentVolumeClaim:
      claimName: rbd-pvc1           #挂载已经申请的pvc磁盘
```

### deployment.yaml例子
```
apiVersion: extensions/v1beta1  # 指定api版本，此值必须在kubectl api-versions中  
kind: Deployment  # 指定创建资源的角色/类型   
metadata:  # 资源的元数据/属性 
  name: demo  # 资源的名字，在同一个namespace中必须唯一
  namespace: default # 部署在哪个namespace中
  labels:  # 设定资源的标签
    app: nginx
    version: v1
spec: # 资源规范字段
  replicas: 1 # 声明副本数目
  revisionHistoryLimit: 3 # 保留历史版本
  selector: # 选择器
    matchLabels: # 匹配标签
      app: nginx
      version: v1
  minReadySeconds: 30 #定义新建的 Pod 经过多少秒后才被视为可用
  terminationGracePeriodSeconds: 30 #30秒内 (默认 30s) 还未完全停止，就发送 SIGKILL 信号强制杀死进程。
  progressDeadlineSeconds: 600 #升级过程中的最大时间(如果升级过程被暂停了，该时间也会同步暂停，时间不会一直增长)
  strategy: # 策略
    rollingUpdate: # 滚动更新
      maxSurge: 30% # 最大额外可以存在的副本数，可以为百分比，也可以为整数
      maxUnavailable: 30% # 示在更新过程中能够进入不可用状态的 Pod 的最大值，可以为百分比，也可以为整数
    type: RollingUpdate # 滚动更新策略
  template: # 模版
    metadata: # 资源的元数据/属性 
      annotations: # 自定义注解列表
        sidecar.istio.io/inject: "false" # 自定义注解名字
      labels: # 设定资源的标签
        app: nginx
        version: v1
    spec: # 资源规范字段
      containers:
      - name: nginx# 容器的名字   
        image: nginx:1.17.0 # 容器使用的镜像地址   
        imagePullPolicy: IfNotPresent # 每次Pod启动拉取镜像策略，三个选择 Always、Never、IfNotPresent
                                      # Always，每次都检查；
                                      # Never，每次都不检查（不管本地是否有）；
                                      # IfNotPresent，如果本地有就不检查，如果没有就拉取（手动测试时，已经打好镜像存在docker容器中时，
                                      #    使用存在不检查级别， 默认为每次都检查，然后会进行拉取新镜像，因镜像仓库不存在，导致部署失败）
        volumeMounts:       #文件挂载目录，容器内配置
        - mountPath: /data/     #容器内要挂载的目录
          name: share       #定义的名字，需要与下面vloume对应
        resources: # 资源管理
          limits: # 最大使用
            cpu: 300m # CPU，1核心 = 1000m
            memory: 500Mi # 内存，1G = 1000Mi
          requests:  # 容器运行时，最低资源需求，也就是说最少需要多少资源容器才能正常运行
            cpu: 100m
            memory: 100Mi
        livenessProbe: # pod 内部健康检查的设置
          httpGet: # 通过httpget检查健康，返回200-399之间，则认为容器正常
            path: /healthCheck # URI地址
            port: 8080 # 端口
            scheme: HTTP # 协议
            # host: 127.0.0.1 # 主机地址
          initialDelaySeconds: 30 # 表明第一次检测在容器启动后多长时间后开始
          timeoutSeconds: 5 # 检测的超时时间
          periodSeconds: 30 # 检查间隔时间
          successThreshold: 1 # 成功门槛
          failureThreshold: 5 # 失败门槛，连接失败5次，pod杀掉，重启一个新的pod
        readinessProbe: # Pod 准备服务健康检查设置
          httpGet:
            path: /healthCheck
            port: 8080
            scheme: HTTP
          initialDelaySeconds: 30
          timeoutSeconds: 5
          periodSeconds: 10
          successThreshold: 1
          failureThreshold: 5
        #也可以用这种方法   
        #exec: 执行命令的方法进行监测，如果其退出码不为0，则认为容器正常   
        #  command:   
        #    - cat   
        #    - /tmp/health   
        #也可以用这种方法   
        #tcpSocket: # 通过tcpSocket检查健康  
        #  port: number 
        ports:
          - name: http # 名称
            containerPort: 8080 # 容器开发对外的端口 
            protocol: TCP # 协议
      imagePullSecrets: # 镜像仓库拉取密钥
        - name: harbor-certification
      volumes:      #挂载目录在本机的路径
      - name: share #对应上面的名字
        hostPath:
          path: /data   #挂载本机的路径
      affinity: # 亲和性调试
        nodeAffinity: # 节点亲和力
          requiredDuringSchedulingIgnoredDuringExecution: # pod 必须部署到满足条件的节点上
            nodeSelectorTerms: # 节点满足任何一个条件就可以
            - matchExpressions: # 有多个选项，则只有同时满足这些逻辑选项的节点才能运行 pod
              - key: beta.kubernetes.io/arch
                operator: In
                values:
                - amd64
```
