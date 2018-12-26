Custom-Metrics及Prometheus监控系统

Custom Metrics概述
Core metrics(核心指标)：由metrics-server提供API metrics.k8s.io，仅提供Node和Pod的CPU和内存使用情况。
Custom Metrics(自定义指标)：由Prometheus Adapter提供API custom.metrics.k8s.io，由此可支持任意Prometheus采集到的指标。
想让k8s一些核心组件，比如HPA，获取核心指标以外的其它自定义指标，则必须部署一套prometheus监控系统，让prometheus采集其它各种指标，但是prometheus采集到的metrics并不能直接给k8s用，因为两者数据格式不兼容，还需要另外一个组件(kube-state-metrics)，将prometheus的metrics
数据格式转换成k8s API接口能识别的格式，转换以后，因为是自定义API，所以还需要用Kubernetes aggregator在主API服务器中注册，以便直接通过/apis/来访问。
Custom Metrics 的部署流程

node-exporter：prometheus的agent端，收集Node级别的监控数据。
prometheus：监控服务端，从node-exporter拉数据并存储为时序数据。
kube-state-metrics： 将prometheus中可以用PromQL查询到的指标数据转换成k8s对应的数据格式，即
转换成【Custerom Metrics API】接口格式的数据，但是它不能聚合进apiserver中的功能。
k8s-prometheus-adpater：聚合apiserver，即提供了一个apiserver【cuester-metrics-api】，
自定义APIServer通常都要通过Kubernetes aggregator聚合到apiserver。
grafana：展示prometheus获取到的metrics。
导入grafana模板。
资源清单文件获取

从kubernetes源码树中的addons下获取 prometheus相关组件的资源清单文件：prometheus、node-exporter、kube-state-metrics。

从DirectXMan12项目获取 组件k8s-prometheus-adpater的清单文件。

grafana的配置在google一搜，很多项目都提供了，这里从heapster项目下载grafana资源清单文件。

下载之后，各组件归类存放到各目录：

$ ls
grafana                 k8s-prometheus-adapter       kube-state-metrics  
node_exporter                prometheus         
规划所有组件部署的名称空间，默认是在kube-system，这里统一部署在monitoring

$ kubectl create namespace monitoring
namespace/monitoring created
并手动将清单文件中，资源所属名称空间改为monitoring

开始部署各组件
现在按上面写的顺序一一部署

部署node-exporter
$ ls node_exporter
node-exporter-ds.yaml  node-exporter-svc.yaml
简单下看此组件部署的资源：

    daemonset 
         daemonset-name:prometheus-node-exporter 
         container-name: prometheus-node-exporter
         hostnetwork：hostPort: 9100
         image: prom/node-exporter:v0.16.0

    Service:
       name: prometheus-node-exporter
       clusterIP: None
应用到集群之上：

$ kubectl apply -f ./node_exporter
daemonset.apps/prometheus-node-exporter created
service/prometheus-node-exporter created

$ kubectl get all -n monitoring 
NAME                                 READY   STATUS    RESTARTS   AGE
pod/prometheus-node-exporter-d4wg7   1/1     Running   0          4m7s
pod/prometheus-node-exporter-tqczz   1/1     Running   0          4m7s
pod/prometheus-node-exporter-wcrh6   1/1     Running   0          4m7s

NAME                               TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)    AGE
service/prometheus-node-exporter   ClusterIP   None         <none>        9100/TCP   4m7s

NAME                                      DESIRED   CURRENT   READY   UP-TO-DATE   AVAILABLE   NODE SELECTOR   AGE
daemonset.apps/prometheus-node-exporter   3         3         3       3            3           <none>          4m7s
部署prometheus
从github下载的清单文件，用statefulset部署的，prometheus本身是有状态的应用，这里只部署一个副本，所以将statefulset改为deployment了，

$ ls prometheus
prometheus-cfg.yaml  prometheus-deploy.yaml  prometheus-rbac.yaml  prometheus-svc.yaml
此组件部署的资源

Deployment
  name:prometheus-server   
  containers-name: prometheus
  image: prom/prometheus:v2.2.1
  containerPort: 9090
Service
   name: prometheus
   type: NodePort
   nodePort: 30090<-->9090
应用：

$ kubectl apply -f ./prometheus
configmap/prometheus-config created
deployment.apps/prometheus-server created
clusterrole.rbac.authorization.k8s.io/prometheus created
serviceaccount/prometheus created
clusterrolebinding.rbac.authorization.k8s.io/prometheus created
service/prometheus created
对于prometheus，有几点说明：

简单将原清单文件中的stateful改为了deployment，部署起来相对简单此，且只部署一个副本。
prometheus自带的UI监听在9090端口，使用到了NodePort，以便集群外访问。
prometheus使用的volume"prometheus-storage-volume"，存储所有它采集到的metrics，应该放于持久卷中。
等一会查看组件已正常运行：

$ kubectl get all -n prom 
NAME                                     READY   STATUS    RESTARTS   AGE
pod/prometheus-node-exporter-d4wg7       1/1     Running   0          9m
pod/prometheus-node-exporter-tqczz       1/1     Running   0          9m
pod/prometheus-node-exporter-wcrh6       1/1     Running   0          9m
pod/prometheus-server-5fcbdbcc6f-nt4wj   1/1     Running   0          2m24s

NAME                               TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)          AGE
service/prometheus                 NodePort    10.107.112.119   <none>        9090:30090/TCP   2m
service/prometheus-node-exporter   ClusterIP   None             <none>        9100/TCP         9m

NAME                                      DESIRED   CURRENT   READY   UP-TO-DATE   AVAILABLE   NODE SELECTOR   AGE
daemonset.apps/prometheus-node-exporter   3         3         3       3            3           <none>          9m

NAME                                DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/prometheus-server   1         1         1            1           2m

NAME                                           DESIRED   CURRENT   READY   AGE
replicaset.apps/prometheus-server-5fcbdbcc6f   1         1         1       2m
部署kube-state-metrics
$ ls kube-state-metrics
kube-state-metrics-deploy.yaml  kube-state-metrics-rbac.yaml  kube-state-metrics-svc.yaml
此组件部署的资源：

deploymet
     name: kube-state-metrics
     replicas: 1
     image: gcr.io/google_containers/kube-state-metrics-amd64:v1.3.1
     containerPort: 8080

service:
     name: kube-state-metrics
     port: 8080
应用之：

$ kubectl apply -f ./kube-state-metrics
deployment.apps/kube-state-metrics created
serviceaccount/kube-state-metrics created
clusterrole.rbac.authorization.k8s.io/kube-state-metrics created
clusterrolebinding.rbac.authorization.k8s.io/kube-state-metrics created
service/kube-state-metrics created
等一会查看：

$ kubectl get pod -n monitoring
NAME                                  READY   STATUS    RESTARTS   AGE  
kube-state-metrics-667fb54645-xj8gr   1/1     Running   0          116s   

$ kubectl get svc -n monitoring
NAME                       TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)          AGE
kube-state-metrics         ClusterIP   10.104.171.60    <none>        8080/TCP         2m50s
部署组件k8s-prometheus-adapter
最后一个核心组件，也是部署最麻烦的一个组件。
它是一个API服务器，提供了一个APIServer服务，名为 custom-metrics-apiserver，提供的API组： custom.metrics.k8s.io，它是自定义指标API（custom.metrics.k8s.io）的实现

查看资源清单文件：

$ ls k8s-prometheus-adapter
custom-metrics-apiserver-auth-delegator-cluster-role-binding.yaml
custom-metrics-apiserver-auth-reader-role-binding.yaml
custom-metrics-apiserver-deployment.yaml
custom-metrics-apiserver-resource-reader-cluster-role-binding.yaml
custom-metrics-apiserver-service-account.yaml
custom-metrics-apiserver-service.yaml
custom-metrics-apiservice.yaml
custom-metrics-cluster-role.yaml
custom-metrics-config-map.yaml
custom-metrics-resource-reader-cluster-role.yaml
hpa-custom-metrics-cluster-role-binding.yaml
此组件部署的资源：

deployment：
     name: custom-metrics-apiserver
     replicas: 1
        containers-name： custom-metrics-apiserver
           image: directxman12/k8s-prometheus-adapter-amd64
           ports:
           - containerPort: 6443
           volumes: secret:
                    secretName: cm-adapter-serving-certs
Service
  name: custom-metrics-apiserver
     ports:
       - port: 443
         targetPort: 6443
APIService
    name: custom-metrics-apiserver
     custom.metrics.k8s.io
      version: v1beta1
从上面该组件的deployment看出，它需要挂一个secret存储卷，secret名为"cm-adapter-serving-certs"，这个secret是一个证书，因此这里需要创建相应的证书和key，这个证书必须由k8s的kube-apiserver信任的CA签发，因此直接用k8s的CA签发。

生成证书：

私钥
$  (umask 077;openssl genrsa -out serving.key 2048)
$  ls
      serving.key
证书请求：
$ openssl req -new -key serving.key -out serving.csr -subj "/CN=serving"
$  ls
serving.csr  serving.key
签署证书：
$ openssl x509 -req -in serving.csr -CA /etc/kubernetes/pki/ca.crt -CAkey /etc/kubernetes/pki/ca.key -CAcreateserial -out serving.crt -days 3650
 Signature ok
 subject=/CN=serving
 Getting CA Private Key

$ ls
serving.crt  serving.csr  serving.key
创建secret：
$ kubectl create secret generic cm-adapter-serving-certs --from-file=serving.crt=./serving.crt --from-file=serving.key=./serving.key  -n monitoring 
secret/cm-adapter-serving-certs created

$ kubectl get secrets -n monitoring 
NAME                             TYPE                                  DATA   AGE
cm-adapter-serving-certs         Opaque                                2      49s
应用资源清单文件：

$ kubectl apply -f ./k8s-prometheus-adapter
clusterrolebinding.rbac.authorization.k8s.io/custom-metrics:system:auth-delegator created
rolebinding.rbac.authorization.k8s.io/custom-metrics-auth-reader created
deployment.apps/custom-metrics-apiserver created
clusterrolebinding.rbac.authorization.k8s.io/custom-metrics-resource-reader created
serviceaccount/custom-metrics-apiserver created
service/custom-metrics-apiserver created
apiservice.apiregistration.k8s.io/v1beta1.custom.metrics.k8s.io created
clusterrole.rbac.authorization.k8s.io/custom-metrics-server-resources created
configmap/adapter-config created
clusterrole.rbac.authorization.k8s.io/custom-metrics-resource-reader created
clusterrolebinding.rbac.authorization.k8s.io/hpa-controller-custom-metrics created
等一会查看：

$ kubectl get all -n monitoring  |grep custom-metrics

pod/custom-metrics-apiserver-746485c45d-9dnqn   1/1     Running   0          69s

service/custom-metrics-apiserver   ClusterIP   10.102.104.175   <none>        443/TCP          70s

deployment.apps/custom-metrics-apiserver   1         1         1            1           70s
replicaset.apps/custom-metrics-apiserver-746485c45d   1         1         1       71s
最后查看所有的pod：四个组件的pod：

$ kubectl get pod -n monitoring -o wide
NAME                                        READY   STATUS    RESTARTS   AGE    IP                NODE         NOMINATED NODE
custom-metrics-apiserver-746485c45d-9dnqn   1/1     Running   0          116s   192.168.85.197    k8s-node01   <none>
kube-state-metrics-667fb54645-xj8gr         1/1     Running   0          63m    192.168.235.196   k8s-master   <none>
prometheus-node-exporter-d4wg7              1/1     Running   0          175m   10.3.1.20         k8s-master   <none>
prometheus-node-exporter-tqczz              1/1     Running   0          175m   10.3.1.21         k8s-node01   <none>
prometheus-node-exporter-wcrh6              1/1     Running   0          175m   10.3.1.25         k8s-node02   <none>
prometheus-server-5fcbdbcc6f-nt4wj          1/1     Running   0          89m    192.168.58.197    k8s-node02   <none>
查看新创建的api群组：

$ kubectl api-versions 
......
custom.metrics.k8s.io/v1beta1
metrics.k8s.io/v1beta1
......
有了自定义指标api了，过一会就可以从接口获取到数据了：

curl localhost:8091/apis/custom.metrics.k8s.io/v1beta1
 "apiVersion": "v1",
  "groupVersion": "custom.metrics.k8s.io/v1beta1",
  "resources": [
    {
      "name": "namespaces/fs_reads_bytes",
      "singularName": "",
      "namespaced": false,
      "kind": "MetricValueList",
      "verbs": [
        "get"
      ]
    },
......
如此，说明自定义指标API已成功部署了，就可以借助于这些自定义指标的创建HPA了。

部署grafana
既然部署了Prometheus，那么当然要部署Grafana展示Prometheus采集到的metrics数据。

查看grafana清单文件：

$ ls grafana
grafana.yaml
它就一个清单文件，部署成一个deploy和service，因为从heapster项目中复制过来的，配置grafana连接的是influxdb，因此需要改下，完整的grafana.yaml如下

$ cat grafana/grafana.yaml 
apiVersion: apps/v1
kind: Deployment
metadata:
  name: monitoring-grafana
  namespace: monitoring
spec:
  replicas: 1
  selector:
    matchLabels:
      task: monitoring
      k8s-app: grafana
  template:
    metadata:
      labels:
        task: monitoring
        k8s-app: grafana
    spec:
      containers:
      - name: grafana
        image: k8s.gcr.io/heapster-grafana-amd64:v5.0.4
        ports:
        - containerPort: 3000
          protocol: TCP
        volumeMounts:
        - mountPath: /etc/ssl/certs
          name: ca-certificates
          readOnly: true
        - mountPath: /var
          name: grafana-storage
        env:
        #- name: INFLUXDB_HOST
        #  value: monitoring-influxdb
        - name: GF_SERVER_HTTP_PORT
          value: "3000"
          # The following env variables are required to make Grafana accessible via
          # the kubernetes api-server proxy. On production clusters, we recommend
          # removing these env variables, setup auth for grafana, and expose the grafana
          # service using a LoadBalancer or a public IP.
        - name: GF_AUTH_BASIC_ENABLED
          value: "false"
        - name: GF_AUTH_ANONYMOUS_ENABLED
          value: "true"
        - name: GF_AUTH_ANONYMOUS_ORG_ROLE
          value: Admin
        - name: GF_SERVER_ROOT_URL
          # If you're only using the API Server proxy, set this value instead:
          # value: /api/v1/namespaces/kube-system/services/monitoring-grafana/proxy
          value: /
      volumes:
      - name: ca-certificates
        hostPath:
          path: /etc/ssl/certs
      - name: grafana-storage
        emptyDir: {}
---
apiVersion: v1
kind: Service
metadata:
  labels:
    # For use as a Cluster add-on (https://github.com/kubernetes/kubernetes/tree/master/cluster/addons)
    # If you are NOT using this as an addon, you should comment out this line.
    kubernetes.io/cluster-service: 'true'
    kubernetes.io/name: monitoring-grafana
  name: monitoring-grafana
  namespace: monitoring
spec:
  # In a production setup, we recommend accessing Grafana through an external Loadbalancer
  # or through a public IP.
  # type: LoadBalancer
  # You could also use NodePort to expose the service at a randomly-generated port
  # type: NodePort
  ports:
  - port: 80
    targetPort: 3000
  selector:
    k8s-app: grafana
  type: NodePort
有两点要说明的是

挂载的volume grafana-storage应该为持久卷，这里测试为挂载为emptyDir
grafana的svc使用了NodePort，便于集群之外访问。
取消了环境变量INFLUXDB_HOST。
应用并查看：

kubectl apply -f grafana/grafana.yaml

$ kubectl get pod -n monitoring |grep grafana
NAME                                        READY   STATUS    RESTARTS   AGE
monitoring-grafana-7f99994bc4-mpmhz         1/1     Running   0          3m

$ kubectl get svc  -n monitoring  |grep grafana
monitoring-grafana         NodePort    10.109.154.210   <none>        80:31337/TCP     6d18h
grafana已成功部署完，接下来，就可以用NodeIP + NodePort 这里是31337 打开grafana界面，接入Prometheus数据源，并下载grafana适用于k8s的grafana来查看各种指标数据了。

Grafana使用
Custom-Metrics及Prometheus监控系统
Custom-Metrics及Prometheus监控系统
进入Dashboards：
Custom-Metrics及Prometheus监控系统
在下面可以导入各种模板：
Custom-Metrics及Prometheus监控系统

模板在哪找呢？在grafana官网https://grafana.com/dashboards 中搜索grafana模板，有很多适用于kubernetes prometheus的模板：
Custom-Metrics及Prometheus监控系统
比如下面找到了1621号模板：
Custom-Metrics及Prometheus监控系统
按下面的方法导入：
Custom-Metrics及Prometheus监控系统

最终展示：
Custom-Metrics及Prometheus监控系统

Grafana之所以能够发现k8s集群中各Node、各Pod的详细使用信息，主要是因为prometheus部署时使用的配置文件，它这个配置是经过改造后适用于运行k8s集群之中，配置了很多Job、Service Discovery功能，可以自动发现集群各资源。
