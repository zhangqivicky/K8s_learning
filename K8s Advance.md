
### ReplicaSet Controller
- 是 Kubernetes 中负责维护指定数量 Pod Replica 的 Controller，它是ReplicationController的新一代替代方案. 核心功能都是：ensures that the specific number of pods are running at all times
- 最大的区别在于：
	- ReplicaSet 支持更灵活的 Label Selector，而 ReplicationController 只支持简单的Equality-based Selector
	- ReplicaSet 支持由Deployment创建和管理，通过Deployment 实现声明式 Rolling Update 和 Rollback。而 ReplicationController不支持Deployment管理，滚动更新需要额外机制 
- ```python
   apiVersion: apps/v1 # 可通过`kubectl api-resources`查看
   kind: ReplicaSet
   metadata:
	   name: nginx-rs
   spec: # 外层的 `spec` 定义 ReplicaSet 如何管理 Pod
	   replicas: 3 # 默认是 1
	   selector: # specify what pods fall under it
		   matchLabels:  # `matchLabels` 和 `matchExpressions` 的条件是 AND 关系
			   app: nginx
		   matchExpressions:   # Set-based Selector
			   - key: environment
			     operator: In #常见的操作符:In,NotIn,Exists,DoesNotExist(exists不需要values)
			     values:
				     - production
			         - staging
       template: # pod template, `selector.matchLabels` 必须与 `template.metadata.labels` 匹配
	       metadata:
		       labels:
			       app: nginx
			       environment: production
		   spec: # 内层的 `spec` 定义 Pod 运行哪些容器
			   containers:
				   - name: nginx
				     image: nginx
				     ports:
					     - containerPort: 80
  ```
- 创建ReplicaSet：`kubectl apply -f nginx-rs.yaml`
- 查看 ReplicaSet： `kubectl get rs`
- 修改副本数量：最好修改yaml file，也可以`kubectl scale rs nginx-rs --replicas=5`
- 删除 ReplicaSet: `kubectl delete -f nginx-rs.yaml`，默认情况下也会级联删除其管理的 Pod
- ==🟡**在实际工作中，通常只需要写一个 `deployment.yaml`，不需要单独创建 `kind: ReplicaSet`或`pod` 的 YAML 文件。** Deployment 会自动创建和管理 ReplicaSet，而 ReplicaSet 再负责创建和管理 Pod。
- 
### ReplicaSet 在 Kubernetes Cluster 中是如何工作的？
- 核心工作是：监控集群中的 Pod 数量，并通过 Reconciliation Loop，让实际 Pod 数量始终接近期望数量。
- ReplicaSet 是 Kubernetes API Resource，而真正执行管理逻辑的是 ReplicaSet Controller。
- ![[k8s_replicaset_1.png|672]]
- 常见的触发 ReplicaSet Controller的events
	- Add ReplicaSet： 创建新的 ReplicaSet
	- Update ReplicaSet：修改 `spec.replicas`、Pod Template 等
	- Delete ReplicaSet： ReplicaSet 被删除，相关缓存和队列状态发生变化
	- Add Pod: 新 Pod 被创建，可能被某个 ReplicaSet 管理
	- Update Pod: Pod Labels、OwnerReference、DeletionTimestamp、状态等发生变化
	- Delete Pod: Pod 被删除，需要补充副本
- **Event-Driven Reconciliation**![[k8s_replicaset_2.png|672]]
### Deployment
- The deployment provides us with the capability to upgrade the underlying instances seamlessly using rolling updates, undo changes, pause and resume changes as required.
- ```python
   apiVersion: apps/v1 # 可通过`kubectl api-resources`查看
   kind: Deployment
   metadata:
	   name: myapp-deployment
	   labels:
		   app: myapp
		   type: front-end
   spec: 
	   replicas: 3 
	   selector: 
		   matchLabels:  
			   app: nginx

       template: # pod template
	       metadata:
		       labels:
			       app: nginx
			       environment: production
		   spec: 
			   containers:
				   - name: nginx
				     image: nginx
				     ports:
					     - containerPort: 80
  ```
- `kubectl get deployments`
- `kubectl get replicaset`
- `kubectl get pods`
- `kubectl get all`
### Services
- 为什么 Kubernetes 需要 Service？
	- 为了让客户端稳定地访问 Kubernetes Cluster 里的 Pods
		- Pod IP 不稳定：Pod 被删除、重新创建或者迁移到其他 Node 后，IP 可能改变
		- Pod 数量会变化：Deployment 可以从 3 个 Pods Scale 到 10 个
		- 客户端不知道应该访问哪个 Pod：我们希望客户端只使用一个稳定的地址
		- 有些服务需要对外开放，有些只应该允许 Cluster 内部访问
- 三种主要 Service 类型：ClusterIP、NodePort、LoadBalancer，它们最大的区别是：允许谁访问，以及客户端从哪里进入 Kubernetes Cluster。
	- NodePort 在 ClusterIP 的基础上增加 Node port，LoadBalancer 通常在 NodePort 的基础上增加外部负载均衡器。不过，部分 LoadBalancer 实现可以直接把流量发送到 Pods，不经过 NodePort。
	- 它们是三种不同的 Service 类型，但存在功能上的继承关系，不需要创建三个独立的 Service 才能工作. 例如你只创建一个 LoadBalancer, 在默认的、基于 NodePort 的实现中，Kubernetes 就会同时为这个 Service 提供：一个 ClusterIP, 一个 NodePort 和 一个 External Load Balancer 入口，这三个入口都指向同一组 Backend Pods。![[k8s_services_1.png]]
-  **ClusterIP ---- 解决 Cluster 内部的 Service-to-Service Communication **
	- **ClusterIP Service can help by grouping all pods together and provide a single interface for other pods to access this service**, the service randomly forward the requests to one of the pods.
	- Each layer can now scale or move as required without impacting communication between the various services
		- ```python
		  apiVersion: v1
		  kind: Service
		  metadata:
			  name: back-end
		  spec:
			  type: ClusterIP
			  ports:
				  - targetPort: 8080 # Backend Pod 里 Application 监听的端口
				    port: 80 # Service 提供的端口
				    protocol: TCP #指定 Service Port 使用的传输层协议，例如 TCP、UDP、SCTP
				    appProtocol: http #指定 Service Port 承载的应用层协议，例如 HTTP、HTTP/2、gRPC。
- 
			  selector: 
				    app: myapp
				    type: back-end # 选择拥有 `app: backend` 标签的 Pods
		  ```
		- ![[k8s_services_2.png|452]]
		- 实际 Request Flow：
			- Frontend 发起 `http://backend-service:80/api/users`
			- Cluster DNS（通常由 CoreDNS 提供）解析 Service 名称，得到 ClusterIP
			- 请求进入 Service 的虚拟网络转发机制，通常由 kube-proxy 配置的内核规则实现，也可能由 eBPF 网络方案实现。
			- 流量被转发到一个可用的 Backend Pod 的 8080 端口
- **NodePort ---- 解决 Cluster 外部用户如何通过 Node 访问 内部pod**
	- 外面的用户, 通常无法直接访问K8s内部 ClusterIP. 
		- Kubernetes 并没有禁止外部访问 ClusterIP，只是默认情况下，ClusterIP 不会被发布到外部网络. 而是Kubernetes 已经为 NodePort 定义了标准化的外部访问方式。
		- 如果要让 ClusterIP 成为可靠的外部入口，就必须解决：
			- 谁负责向外部 Router 宣告这个 IP？
			- Node 故障时，如何把流量切换到其他 Node？
			- 如何保证流量始终进入能够处理对应 Service 的节点？
			- 如何防止外部网络暴露不应该公开的内部 Service？
		- 这些问题需要额外的网络控制组件解决。因此，实际工程中更常见的是利用外部 Load Balancer、Gateway API 或 Ingress Controller 提供专门的稳定入口，而不是直接暴露内部 ClusterIP![[k8s_services_6.png]]
	- NodePort 的核心功能，就是在每个 Node 上开放一个固定端口，让集群外部的客户端可以通过 Node IP + NodePort 访问内部的 Pods。![[k8s_services_5.png]]
		- 为什么集群外部的 Client 可以访问 Node IP，却不能直接访问 Pod IP？
			- 关键在于：客户端所在的网络是否有到目标 IP 的有效路由，虽然也可以直接设置client到pod ip的路由，但pod ip通常不稳定，路由也会不稳定
			- 判断客户端所在网络是否有到目标 IP 的有效路由，最直接的方法就是查看 Routing Table（路由表）
			- 比如，Client IP：`192.168.1.50`， Node IP：`192.168.1.10`， Pod IP：`10.244.1.5`， 在 Linux 上可以运行`ip route get 10.244.1.5`
			- 路由选择规则是， 当多条路由都能匹配目标 IP 时，优先选择前缀长度最长的那一条，前缀长度就是 CIDR 中 `/` 后面的数字，数字越大，前缀越长，匹配范围越小、越精确
	- **NodePort service can help by mapping a port on the node to a port on the pod**
		- **Target port**: the port on the pod where the actual web server is running
		- **Port**: the port on the service its self
		- **Node port**: the port on the node which we use to access the web server externally. It can only be in a valid range, which by default is from 30,000 to 32,767.  不要求请求进入的 Node 必须运行目标 Pod
		- ```python
		  apiVersion: v1
		  kind: Service
		  metadata:
			  name: myapp-service
		  
		  spec: # defining the actual service
			  type: NodePort
			  ports: # - indicates it is an array
				  - targetPort: 80 # default to be the same as port
					port: 80 # this is the only mandatory port field
					nodePort: 30080 # 指定对外使用的 Node 端口。默认允许范围是 `30000–32767`，也可以让 Kubernetes 自动分配
			  selector: # specify which pod the target port is on
					app: backend
		  ```
		![[k8s_services_3.png|498]]
		- 具体步骤如下：
			- Step 1. External Client 发送请求， 比如访问`192.168.1.10:30080`,请求通过网络到达 Node A
			- Step 2. data packet到达 Node A 的网卡（NIC), 进入 Linux Kernel 的 Networking Stack
			- Step 3. Linux Kernel 中的 Netfilter Framework根据 kube-proxy 事先配置的 iptables 规则执行 DNAT(Destination Network Address Translation)，把目标地址改为 `Pod IP:port`
				- kube-proxy Watch API Server，获取 Service 和 EndpointSlice 的变化
				- 发现某个 Service 使用 `type: NodePort`，例如端口为 `30080`
				- kube-proxy calls appropriate APIs (depending on the kube-proxy mode)来配置对应的数据包转发NAT 规则(iptables的一种)
				- 当 Pod 增加、删除或者 Ready 状态改变时，更新这些规则
			- Step 4. Linux Routing Subsystem 查询 Routing Table，根据修改后的 Pod IP 找到通往 目标pod ip 的网络路径
			- Step 5. Pod 处理请求并返回 Response
		- 有三点需要注意：
			- 所有 Node 使用相同的 NodePort
				- 前提是这些 Node IP 从客户端可达，而且网络防火墙允许该端口
			- Node 不一定需要运行目标 Pod
				- 默认 `externalTrafficPolicy: Cluster` 时支持这种跨 Node 转发
			- NodePort Service 本身也有 ClusterIP
				- NodePort 不是取代 ClusterIP，而是在 ClusterIP Service 的功能基础上额外增加了 Node 级别的访问入口
	- What if when you have multiple pods in a production environment? They all have the same labels 
		- If there are 3 pods running the application, when the service is created, it looks for a matching pod with the label and finds three of them automatically. It uses a **random algorithm** to balance the load across the three different pods.
	-   `kubectl get services`
- **LoadBalancer ---- 解决 Production 环境的统一外部入口**
	- NodePort 虽然支持外部访问，但存在一些不足：
		- 缺乏统一的外部访问入口： 用户需要知道某个 Node 的 IP 地址
		- Node 故障: 一个 Node 故障后，依赖该 Node IP 的客户端可能无法继续访问
		- Node 增减： 集群扩容或缩容时，需要管理可访问的 Node 地址
		- 端口不方便： NodePort 默认端口范围为 `30000–32767`，而 Web 应用通常希望直接使用 80/443
	- 虽然可以手动在 NodePort 前面配置一个External Load Balancer，但 Kubernetes 希望这个过程能够自动化, LoadBalancer Service 就是把External Load Balancer 和 Kubernetes Service 集成起来:
		- ==🟡提供统一的 External IP 或 DNS==
		- 将外部流量分发到健康的后端
		- 屏蔽 Node 的变化
		- 可以与cloud platform的load balancing和health check能力集成
	- **Note**: 
		- LoadBalancer Service 并不是 Kubernetes 自己运行了一个负载均衡进程。它是通过 Kubernetes 的声明式 API，要求外部负载均衡实现提供并管理入口
		- This only works with supported cloud platforms like GCP, AWS and Azure
		- ==🟡对于普通的 Kubernetes `LoadBalancer` Service，Kubernetes 通常会自动分配一个 `ClusterIP`，所以你不需要再单独创建一个 `type: ClusterIP` 的 Service。 `LoadBalancer` Service 同时具备 `ClusterIP` 和通常的 `NodePort` Service 的能力，同时支持内部和外部访问==
	- ```python
	  apiVersion: v1
	  kind: Service
	  metadata:
		  name: backend-service
	  spec:
		  type: LoadBalancer
		  selector:
			  app: backend
		  ports:
		    - protocol: TCP
		      port: 80
		      targetPort: 8080
		      nodePort： 30080 #如果没有手动填写,kubernetes会自动分哦诶
	  ```
	- ![[k8s_services_4.png|408]]
	- ==🟠**LoadBalancer Service 的底层工作原理**==
		- ==🟠**Control Plane: 创建与配置 Load Balancer Service**==
			- Step 1. 用户创建 LoadBalancer Service : `kubectl apply -f xxx.yaml`
				- 请求发给 API Server, API Server 验证,并将 Service Object 持久化到 `etcd`
			- Step 2. Kubernetes 创建 Service 内部访问配置，并维护 EndpointSlice
				- Kubernetes 为普通 LoadBalancer Service 分配 `ClusterIP`
				- 默认情况下，还会分配 NodePort，例如 `nodePort: 30080`；如果设置 `allocateLoadBalancerNodePorts: false`，则不自动分配
				- EndpointSlice Controller Watch API Server，根据 Service selector 和 Pod 状态维护 EndpointSlice，记录匹配 Pod 的 IP、端口以及就绪条件。
			- Step 3. Cloud Controller Manager 创建 External Load Balancer
				- `cloud-controller-manager` 中的 Service Controller Watch API Server，发现 `type: LoadBalancer` 的 Service
				- 通过 Cloud Provider API 请求创建或更新 External Load Balancer
				- 例如在 AWS EKS 中，可以使用 AWS Load Balancer Controller 创建 NLB(network load balancer)
				- Controller 配置 Listener、Target Group、Health Check 等
				- 具体由 Cloud Controller Manager 内的 Service Controller，还是专用 Load Balancer Controller 处理，取决于集群配置
			- Step 4. 根据 External Load Balancer 的后端模式配置转发
				- **NodePort / Instance Target 模式**
					- External LB 的后端目标是 Node，例如 `NodeIP:30080`
					- kube-proxy Watch API Server，获取 Service 和 EndpointSlice 的变化
					- 发现某个 Service 使用 `type: LoadBalancer`，并且采用默认的 NodePort 分配模式，例如 `nodePort: 30080`
					- kube-proxy calls appropriate APIs (depending on the kube-proxy mode)配置 NodePort → Pod Endpoint 的转发规则
						- iptables 模式下，通过 iptables 配置 Netfilter NAT / DNAT 规则。
						- nftables 模式下，配置 nftables 规则
						- IPVS 模式下，配置 IPVS 虚拟服务及相关网络规则。
					- 当 EndpointSlice 发生变化时，kube-proxy 更新其负责的转发配置
				- **Direct-to-Pod / IP Target 模式**
					- External LB 的后端目标是 `PodIP:targetPort`
					- Load Balancer Controller Watch API Server,  发现 Service 和 EndpointSlice 等资源变化，确定目标 Pod IP。
					- Controller 调用 Cloud Provider API，将 Pod IP 注册为 External LB 的后端目标。例如 AWS NLB 的 IP Target Group,NLB 根据health check等信息选择一个 Pod IP
				- **NodePort vs. Direct-to-Pod**
					- 如果网络可达、控制器支持，并且健康检查、Pod 注销和连接排空都配置完善，Direct-to-Pod 通常能:减少经过 NodePort 的中间转发环节, 让负载均衡器直接针对 Pod 做健康检查	
					- 但还要看 **Pod 频繁扩缩容时的注册速度、负载均衡器的目标数量限制，以及平台成本和运维要求**。这些因素可能让 NodePort 模式更合适。
					- 所以，通常情况下，条件成熟时，优先考虑 Direct-to-Pod；最终根据平台限制和实际需求选择
			- Step 5. 外部 LB 地址写回 Service
				- External Load Balancer 创建成功后，相应的controller将外部 IP 或 hostname 更新到 Service 的 `.status.loadBalancer.ingress`
					- ```python
					  status:
						  loadBalancer:
							  ingress:
								  - ip: 203.0.113.10
					  ```
				- 运行`kubectl get svc` 就能看到 `EXTERNAL_IP`
				- **Note**: 这个过程是异步的。如果没有合适的 Load Balancer Controller 或云平台支持，`EXTERNAL-IP` 可能一直显示 `<pending>`
		- ==🟠**Data Plane：一个 HTTP Request 如何到达 Pod?**==
			- Step 1. 用户访问`203.0.113.10:80`, 请求到达 External Load Balancer
			- Step 2. Load Balancer 根据其配置的 Target Group、健康检查及负载均衡算法，选择 Node A
				- 转发目的地址变为: `10.0.1.10:30080`
			- Step 3. Packet 经过Node A 的 NIC, 进入Linux Kernel Networking Stack
			- Step 4. Linux Kernel 中的 Netfilter Framework根据 kube-proxy 配置的规则执行DNAT,把目标地址改为 `Pod IP:port`
			- Step 5. Linux Routing Subsystem 查询 Routing Table，根据修改后的 Pod IP 找到通往 目标pod ip 的网络路径
			- Step 6. Pod 处理 Request, 并返回response
-	`kubectl get svc backend-service`
- **External Load Balancer 和 kube-proxy 的负载均衡有什么区别？**
	- ![[k8s_services_7.png|618]]
	- 注意：External LB 选中 Node B，并不意味着最终必须访问 Node B 上的 Pod，这取决于配置 `externalTrafficPolicy:` Cluster vs. Local, 默认是 `Cluster`
### Namespace
- Namespace（命名空间）是一种在同一个 Kubernetes Cluster 内，对资源进行逻辑分组和隔离的机制。
	- Resource Organization：把相关的 Pods、Services、Deployments 放在一起
	- Name Isolation：不同 Namespace 可以存在同名资源
	- Access Control：使用 RBAC 限制用户对不同 Namespace 的访问权限
	- Resource Quota：限制不同 Namespace 使用的 CPU、Memory 等资源
	- Network Isolation：结合 NetworkPolicy 限制不同 Namespace 的网络通信
- Note: 
	- Namespace 默认只提供逻辑上的资源划分，不等于物理隔离，也不意味着网络自动隔离. 例如，`development` 和 `production` 中的 Pods 仍然可能运行在同一个 Node 上，而且默认情况下，它们通常可以通过网络互相通信
	- ==🟠每个 Namespaced Resource 都有且只有一个 Namespace。== 一个 Deployment、ReplicaSet、Pod 或 Service 只能属于一个 Namespace，不能同时属于多个 Namespace。
	- 不是所有 Kubernetes 资源都属于 Namespace. 
		- 可以通过`kubectl api-resources --namespaced=true` 查询哪些资源支持 Namespace
		- 当你创建一个 Deployment 时，API Server 会根据 `metadata.namespace` 确定其所属的 Namespace
		- Kubernetes Cluster 中的 Resources（资源）可以分成两大类：Namespaced Resources 和 Cluster-scoped Resources
			- Namespaced Resources：必须属于某一个 Namespace
			- Cluster-scoped Resources：不属于任何 Namespace，而是在整个 Cluster 层级定义和管理的资源
		- ![[k8s_namespace_1.png|520]]
- 常用的commands:
	- 查看现有 Namespace: 
		- `kubectl get namespaces` 
		-  `kubectl get ns`
	- 创建 Namespace: 
		- `kubectl create namespace production` 
		- `kubectl apply -f namespace.yaml`
	- 在指定 Namespace 中创建资源: 
		- `kubectl apply -f deployment.yaml -n <namespace>`
	- 查看指定 Namespace 的资源: 
		- `kubectl get pods -n <namespace>`
		- `kubectl get deployments -n <namespace>
		- `kubectl get services -n <namespace>
	- 查看所有 Namespace 中的 Pods: 
		- `kubectl get pods --all-namespaces` 
		- `kubectl get pods -A`
	- 后续命令默认操作某个namespace，可以修改当前 kubectl context：
		- `kubectl config set-context --current --namespace=production`
		- Context 是给 `kubectl` 等 Kubernetes 客户端使用的连接配置： 
			- step1: 你或脚本执行 kubectl get pods
			- step2: kubectl 读取当前 Context
			- step3: 使用指定身份，访问指定cluster和namespace
		- 常见的commands：
			- 查看所有context: `kubectl config get-contexts`
			- 查看当前context: `kubectl config current-context`
			- 切换context：`kubectl config use-context dev`
		- 这些配置通常保存在 **`~/.kube/config`** 中。**Context 只选择连接和身份，不会额外授予权限**；具体能做什么仍由集群的授权规则决定
- 常见的四个内置 Namespace：
	- `default`: 未指定 Namespace 时，默认使用的命名空间
	- `kube-system`: 存放 Kubernetes 系统组件，比如 CoreDNS
	- `kube-public`: 为集群中需要公开读取的资源提供命名空间
	- `kube-node-lease`: 存放用于 Node 心跳机制的 Lease 对象
		- Lease 对象是 Kubernetes 的一种 API 资源，用来记录“谁持有租约、最近什么时候续约、租约时长是多少。Lease 也可以用于其他场景，例如组件选主，并非只能用于节点心跳
		- 用于 Node 心跳时，每个 Node 在 `kube-node-lease` namespace 中都有一个同名的 Lease
		- ```python
		  apiVersion: coordination.k8s.io/v1
		  kind: Lease  
		  metadata:
			  name: worker-1
			  namespace: kube-node-lease  
		  spec:
			  holderIdentity: worker-1
			  leaseDurationSeconds: 40
			  renewTime: "2026-11-02T10:00:20Z"
		  ```
- **不同 Namespace 中的 Service 如何通信?**
	- 同一个 Namespace 内, Pod 可以直接使用 Service 名称: `http://backend:8080`
	- 不同 Namespace 之间
		- 需要指定 Namespace: `http://backend.production:8080`
		- 或者使用完整的 Kubernetes Service DNS: `<service-name>.<namespace>.svc.cluster.local`, 比如`http://backend.production.svc.cluster.local:8080`
			- 当Pod 使用自定义 DNS，例如 `dnsPolicy: None`，需要完成dns
	- Note:
		- Namespace 不会默认阻止跨 Namespace 通信。 要限制通信，需要额外配置 NetworkPolicy，并确保集群的网络插件支持并执行该策略
		- Service 的 selector 只会匹配同一 Namespace 内的 Pods。普通 Service 不能直接通过 selector 选择其他 Namespace 的 Pod。
- **Namespace 如何实现资源隔离?**
	- kubernetes默认支持资源名字隔离
	- 按照以下内容隔离资源需要额外配置：
		- CPU / Memory 配额, 需要使用 ResourceQuota
		- 用户权限隔离, 需要使用 RBAC
		- 网络隔离, 需要使用NetworkPolicy
	- kubernetes不支持Node 物理隔离
	- ```python
	  # 对于设置了相关资源配额的 Namespace，新创建的 Pod 通常也需要配置对应的 requests/limits，或者通过 LimitRange 提供默认值。
	  apiVersion: v1
	  kind: ResourceQuota
	  metadata:
		  name: production-quota
		  namespace: production
	  spec:
		  hard: # 是强制执行的限制
			  requests.cpu: "4" # 所有 Pods 的 CPU requests 总和最多 4 cores
			  requests.memory: 8Gi # Memory requests 总和最多 8Gi
			  limits.cpu: "8" # CPU limits 总和最多 8 cores
			  limits.memory: 16Gi # Memory limits 总和最多 16Gi
			  pods: "20" # Pods 数量最多 20 个
	  ```
	- ```python
	  # 定义权限：允许读取 dev namespace 中的 Pod
	  apiVersion: rbac.authorization.k8s.io/v1 
	  kind: Role 
	  metadata:
		  name: pod-reader
		  namespace: dev
	  rules:
		  - apiGroups: [""]
		    resources: ["pods"]
		    verbs: ["get", "list", "watch"]
	  ######################################  
	  # 把权限授予 Alice
	  apiVersion: rbac.authorization.k8s.io/v1
	  kind: RoleBinding
	  metadata:
		  name: alice-read-pods
		  namespace: dev
	  subjects:
		  - kind: User
		    name: alice
		    apiGroup: rbac.authorization.k8s.io
	  roleRef:
		  kind: Role
		  name: pod-reader
		  apiGroup: rbac.authorization.k8s.io
	  ```
	- ```python
	  # 只允许 `production` Namespace 内的 Pods 通过 TCP 8080 端口访问 `app=backend` 的 Pods
	  # Kubernetes NetworkPolicy 的允许规则是叠加的，而不是后面的规则覆盖前面的规则
	  apiVersion: networking.k8s.io/v1
	  kind: NetworkPolicy  
	  metadata:
		  name: backend-network-policy
		  namespace: production
	  spec:
		  podSelector:  #保护哪些 Pods
			  matchLabels:
		      app: backend

		  policyTypes:
			  - Ingress # 控制哪些流量可以进入被选中的 Pods
			    # Egress, 控制被选中的 Pods 可以向外发送哪些流量

		  ingress:
			  - from: # 允许谁访问
			      - podSelector: {} # 表示当前 NetworkPolicy 所在 Namespace 的所有 Pods
				    '''
				  - namespaceSelector:
			            matchLabels:
				            environment: staging
					'''
			  ports:
			      - protocol: TCP
		          port: 8080
	  ```
- **Namespace 和 Cluster 的区别**
	- Cluster 是 Kubernetes 的管理和运行边界，Namespace 是 Cluster 内的逻辑资源划分
	- Deployment、ReplicaSet、Pod 和 Service 等 Namespaced Resources 都属于某个 Namespace，而 Node 等 Cluster-scoped Resources 不属于任何 Namespace
	- Namespace 默认不提供完整的安全隔离和网络隔离；需要搭配 RBAC、ResourceQuota 和 NetworkPolicy 等机制![[k8s_namespace_2.png]]

