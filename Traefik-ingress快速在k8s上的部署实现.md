## 负载均衡快速搭建部署:

Traefik是一个为了让部署微服务更加便捷而诞生的现代HTTP反向代理、负载均衡工具。 它支持多种后台 (Docker, Swarm, Kubernetes, Marathon, Mesos, Consul, Etcd, Zookeeper, BoltDB, Rest API, file…) 来自动化、动态的应用它的配置文件设置。

### 为什么比较偏向域Traefik呢，下面来简单对比下。


#### ingress:

使用nginx作为前端负载均衡，通过ingress controller不断的和kubernetes api交互，实时获取后端service，pod等的变化，然后动态更新nginx配置，并刷新使配置生效，达到服务发现的目的。 

#### traefik:

traefik本身设计的就能够实时跟kubernetes api交互，感知后端service，pod等的变化，自动更新配置并重载。

相对来说traefik更快速方便，同时支持更多的特性，使反向代理，负载均衡更直接更高效。



### 源码clone下来:

    [root@k8smaster ~]#  git clone https://github.com/containous/traefik.git


来看看目录下都有什么，顺便找到对应的K8S文件。


    [root@k8s-master Istio]# pwd
    /root/Istio
    [root@k8s-master Istio]# cd traefik/
    [root@k8s-master traefik]# cd examples/
    [root@k8s-master examples]# cd k8s
    [root@k8s-master k8s]# ls
    cheese-default-ingress.yaml  cheese-services.yaml traefik-ds.yaml
    cheese-deployments.yaml  cheeses-ingress.yaml traefik-rbac.yaml
    cheese-ingress.yaml  traefik-deployment.yaml  ui.yaml
    [root@k8s-master k8s]# pwd
    /root/Istio/traefik/examples/k8s


到这一层就找到了所需的文件，一般呢只需要两个文件，第一个就是deployment和rbac。

原因呢很简单，在第一篇部署的时候我们就说了，由于在Kubernets1.6之后启用了RBAC鉴权机制，所以需配置ClusterRole以及ClusterRoleBinding来对api-server的进行相应权限的鉴权。
那rbac这个文件呢就是创建ClusterRole和ClusterRoleBinding的，至于deployment文件这里就不说了，相信看到本篇文章的童鞋已经对K8S有了基本认识。


#### 开始创建rbac


    [root@k8smaster k8s]# kubectl apply -f traefik-rbac.yaml 
    clusterrole.rbac.authorization.k8s.io "traefik-ingress-controller" created
    clusterrolebinding.rbac.authorization.k8s.io "traefik-ingress-controller" created
    
    检查是否成功
    
    [root@k8smaster k8s]# kubectl get clusterrolebinding
    NAME   AGE
    cluster-admin  113d
    flannel113d
    heapster   113d
    kubeadm:kubelet-bootstrap  113d
    ……….
    traefik-ingress-controller 3s
     
    [root@k8smaster k8s]# kubectl get clusterrole
    NAMEAGE
    admin   113d
    cluster-admin   113d
    edit113d
    flannel 113d
    



可以看到clusterrole，clusterrolebinding都创建成功了，下面创建Traefik。


 [root@k8smaster k8s]# kubectl apply -f traefik-deployment.yaml 
serviceaccount "traefik-ingress-controller" created
deployment.extensions "traefik-ingress-controller" created
service "traefik-ingress-service" created
 
检查是否成功

    [root@k8smaster k8s]# kubectl get svc,deployment,pod -n kube-system
    NAMETYPECLUSTER-IP   EXTERNAL-IP   PORT(S) AGE
    heapsterClusterIP   10.106.236.144   <none>80/TCP  113d
    kube-dnsClusterIP   10.96.0.10   <none>53/UDP,53/TCP   113d
    kubernetes-dashboard-external   NodePort10.108.106.113   <none>9090:30090/TCP  113d
    traefik-ingress-service NodePort10.98.76.58  <none>80:30883/TCP ,8080:30731/TCP   17s
     
    NAME DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE
    heapster 1 1 11   113d
    kube-dns 1 1 11   113d
    kubernetes-dashboard 1 1 11   113d
    traefik-ingress-controller   1 1 10   18s
     
    NAME READY STATUSRESTARTS   AGE
    etcd-k8smaster   1/1   Running   6  113d
    heapster-6595c54cb9-f7gvz1/1   Running   4  113d
    kube-apiserver-k8smaster 1/1   Running   6  113d
    ……….
    traefik-ingress-controller-bf6486db6-jzd8w   1/1   Running   0  17s







刚才前面也说到了有个非常简洁漂亮的界面，非常适合运维统计管理，下面来看看。

    [root@k8smaster k8s]# cat ui.yaml 
    ---
    apiVersion: v1
    kind: Service
    metadata:
      name: traefik-web-ui
      namespace: kube-system
    spec:
      selector:
    k8s-app: traefik-ingress-lb
      ports:
      - name: web
    port: 80
    targetPort: 8080
    ---
    apiVersion: extensions/v1beta1
    kind: Ingress
    metadata:
      name: traefik-web-ui
      namespace: kube-system
    spec:
      rules:
      - host: traefik-ui.minikube
    http:
      paths:
      - path: /
    backend:
      serviceName: traefik-web-ui
      servicePort: web
     
    [root@k8smaster k8s]# kubectl apply -f ui.yaml
    service "traefik-web-ui" created
    ingress.extensions "traefik-web-ui" created
    
    [root@k8smaster k8s]# kubectl describe ing traefik-web-ui -n kube-system
    Name: traefik-web-ui
    Namespace:kube-system
    Address:  
    Default backend:  default-http-backend:80 (<none>)
    Rules:
      Host Path  Backends
      ---- ----  --------
      traefik-ui.minikube  
       /   traefik-web-ui:web (10.0.100.203:8080,10.0.100.204:8080)




参考：http://blog.51cto.com/devingeng/2153778



拓展文件：



可以通过https://github.com/containous/traefik/tree/master/examples/k8s 下载所需要的yaml文件； 我们使用了如下几个文件：


### traefik-rbac.yaml

    ---
    kind: ClusterRole
    apiVersion: rbac.authorization.k8s.io/v1beta1
    metadata:
      name: traefik-ingress-controller
    rules:
      - apiGroups:
      - ""
    resources:
      - services
      - endpoints
      - secrets
    verbs:
      - get
      - list
      - watch
      - apiGroups:
      - extensions
    resources:
      - ingresses
    verbs:
      - get
      - list
      - watch
    ---
    kind: ClusterRoleBinding
    apiVersion: rbac.authorization.k8s.io/v1beta1
    metadata:
      name: traefik-ingress-controller
    roleRef:
      apiGroup: rbac.authorization.k8s.io
      kind: ClusterRole
      name: traefik-ingress-controller
    subjects:
    - kind: ServiceAccount
      name: traefik-ingress-controller
      namespace: kube-system



### traefik-ds.yaml

    ---
    apiVersion: v1
    kind: ServiceAccount
    metadata:
      name: traefik-ingress-controller
      namespace: kube-system
    ---
    kind: DaemonSet
    apiVersion: extensions/v1beta1
    metadata:
      name: traefik-ingress-controller
      namespace: kube-system
      labels:
    k8s-app: traefik-ingress-lb
    spec:
      template:
    metadata:
      labels:
    k8s-app: traefik-ingress-lb
    name: traefik-ingress-lb
    spec:
      serviceAccountName: traefik-ingress-controller
      terminationGracePeriodSeconds: 60
      hostNetwork: true
      containers:
      - image: traefik
    name: traefik-ingress-lb
    ports:
    - name: http
      containerPort: 80
      hostPort: 80
    - name: admin
      containerPort: 8080
    securityContext:
      capabilities:
    drop:
    - ALL
    add:
    - NET_BIND_SERVICE
    args:
    - --api
    - --kubernetes
    - --logLevel=INFO
    ---
    kind: Service
    apiVersion: v1
    metadata:
      name: traefik-ingress-service
      namespace: kube-system
    spec:
      selector:
    k8s-app: traefik-ingress-lb
      ports:
    - protocol: TCP
      port: 80
      name: web
    - protocol: TCP
      port: 8080
      name: admin
      type: NodePort



### ui.yaml


    apiVersion: extensions/v1beta1
    kind: Ingress
    metadata:
      name: traefik-ingress
      namespace: kube-system
    spec:
      rules:
      - host: elasticsearch.donkey
    http:
      paths:
      - path: /
    backend:
      serviceName: elasticsearch-logging
      servicePort: 9200
      - host: kibana.donkey
    http:
      paths:
      - path: /
    backend:
      serviceName: kibana-logging
      servicePort: 5601


三：部署与验证

1.创建资源  kubectl create -f .

2. 通过kubectl logs -f     确认pod正常启动

3.traefik  dashboard

4.如果需要在kubernetes集群以外访问就需要设置DNS，或者修改本机的hosts文件。然后通过Igress配置中的host  直接访问service.

