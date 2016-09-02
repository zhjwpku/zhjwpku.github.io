---
layout: post
title: Kubernetes Cluster Add-ons
date: 2016-09-01 19:00:00 +0800
categories: docker
tags:
- k8s
---

在上一篇博客[3台机器部署Kubernetes集群][k8s_cluster]中，笔者介绍了部署Kubernetes集群的过程，但仅部署了`kube-apiserver`、`kube-controller-manager`、`kube-schedular`、`kubelete`、`kube-proxy`几个基本模块，本篇将介绍Kubernetes几个实用的扩展插件的安装。扩展插件一般都是定义好的yaml文件，可直接使用Kubernetes进行创建及控制。

<h4>kube-dns</h4>

DNS扩展插件用于支持Kubernetes的服务发现机制，它包括`SkyDNS`、`Kube2sky`、`etcd`和`healthz`四个容器。`SkyDNS`提供DNS解析服务，`etcd`用于存储`SkyDNS`的数据，`Kube2sky`负责监听Kubernetes，当有新的Service创建，它会将Service的记录添加到`SkyDNS`中，而kubectl可以通过查询`SkyDNS`将相应的服务记录添加到新创建的Service中。

- 配置环境变量：
{% highlight shell %}
$ export DNS_SERVER_IP=“10.254.10.2”
$ export DNS_DOMAIN="cluster.local"
{% endhighlight %}

- 将如下参数配额到kubelet的启动命令中：
{% highlight shell %}
$ --cluster-dns=10.254.10.2
$ --cluster-domain=cluster.local
{% endhighlight %}

如果kubelet已经启动并且没有配置启动参数，则需要将原进程kill掉并重启。

DNS的作用很大，如果没有该服务，好多Pod在启动的时候会遇到问题，如：
{% highlight shell %}
kubelet does not have ClusterDNS IP configured and cannot create Pod using "ClusterFirst" policy. Falling back to DNSDefault policy.
{% endhighlight %}

*kube-dns-rc.yaml*

{% highlight yaml %}
apiVersion: v1
kind: ReplicationController
metadata:
  name: kube-dns-v11
  namespace: kube-system
  labels:
    k8s-app: kube-dns
    version: v11
    kubernetes.io/cluster-service: "true"
spec:
  replicas: 1
  selector:
    k8s-app: kube-dns
    version: v11
  template:
    metadata:
      labels:
        k8s-app: kube-dns
        version: v11
        kubernetes.io/cluster-service: "true"
    spec:
      containers:
      - name: etcd
        image: gcr.io/google_containers/etcd-amd64:2.2.1
        resources:
          limits:
            cpu: 100m
            memory: 500Mi
          requests:
            cpu: 100m
            memory: 50Mi
        command:
        - /usr/local/bin/etcd
        - -data-dir
        - /var/etcd/data
        - -listen-client-urls
        - http://127.0.0.1:2379,http://127.0.0.1:4001
        - -advertise-client-urls
        - http://127.0.0.1:2379,http://127.0.0.1:4001
        - -initial-cluster-token
        - skydns-etcd
        volumeMounts:
        - name: etcd-storage
          mountPath: /var/etcd/data
      - name: kube2sky
        image: gcr.io/google_containers/kube2sky:1.14
        resources:
          limits:
            cpu: 100m
            memory: 200Mi
          requests:
            cpu: 100m
            memory: 50Mi
        livenessProbe:
          httpGet:
            path: /healthz
            port: 8080
            scheme: HTTP
          initialDelaySeconds: 60
          timeoutSeconds: 5
          successThreshold: 1
          failureThreshold: 5
        readinessProbe:
          httpGet:
            path: /readiness
            port: 8081
            scheme: HTTP
          initialDelaySeconds: 30
          timeoutSeconds: 5
        args:
        # command = "/kube2sky"
        - --domain=cluster.local
        - --kube-master-url=http://10.0.63.202:8080    # 这行原本不存在，必须添加
      - name: skydns
        image: gcr.io/google_containers/skydns:2015-10-13-8c72f8c
        resources:
          limits:
            cpu: 100m
            memory: 200Mi
          requests:
            cpu: 100m
            memory: 50Mi
        args:
        # command = '/skydns'
        - -machines=http://127.0.0.1:4001
        - -addr=0.0.0.0:53
        - -ns-rotate=false
        - -domain=cluster.local.
        ports:
        - containerPort: 53
          name: dns
          protocol: UDP
        - containerPort: 53
          name: dns-tcp
          protocol: TCP
      - name: healthz
        image: gcr.io/google_containers/exechealthz:1.0
        resources:
          limits:
            cpu: 10m
            memory: 20Mi
          requests:
            cpu: 10m
            memory: 20Mi
        args:
        - -cmd=nslookup kubernetes.default.svc.cluster.local 127.0.0.1 >/dev/null
        - -port=8080
        ports:
        - containerPort: 8080
          protocol: TCP
      volumes:
      - name: etcd-storage
        emptyDir: {}
      dnsPolicy: Default
{% endhighlight %}

*kube-dns-svc.yaml*

{% highlight yaml %}
apiVersion: v1
kind: Service
metadata:
  name: kube-dns
  namespace: kube-system
  labels:
    k8s-app: kube-dns
    kubernetes.io/cluster-service: "true"
    kubernetes.io/name: "KubeDNS"
spec:
  selector:
    k8s-app: kube-dns
  clusterIP: 10.254.10.2
  ports:
  - name: dns
    port: 53
    protocol: UDP
  - name: dns-tcp
    port: 53
    protocol: TCP
{% endhighlight %}

<h4>Dashboard</h4>

Dashboard是Kubernete的可视化管理界面，存在目录kubernetes/cluster/addons/dashboard目录下。

*dashboard-controller.yaml*

{% highlight yaml %}
# This file should be kept in sync with cluster/gce/coreos/kube-manifests/addons/dashboard/dashboard-controller.yaml
apiVersion: v1
kind: ReplicationController
metadata:
  name: kubernetes-dashboard-v1.1.1
  namespace: kube-system
  labels:
    k8s-app: kubernetes-dashboard
    version: v1.1.1
    kubernetes.io/cluster-service: "true"
spec:
  replicas: 1
  selector:
    k8s-app: kubernetes-dashboard
  template:
    metadata:
      labels:
        k8s-app: kubernetes-dashboard
        version: v1.1.1
        kubernetes.io/cluster-service: "true"
    spec:
      containers:
      - name: kubernetes-dashboard
        image: gcr.io/google_containers/kubernetes-dashboard-amd64:v1.1.1
        resources:
          # keep request = limit to keep this container in guaranteed class
          limits:
            cpu: 100m
            memory: 50Mi
          requests:
            cpu: 100m
            memory: 50Mi
        ports:
        - containerPort: 9090
        args:
        - --apiserver-host=http://10.0.63.202:8080
        livenessProbe:
          httpGet:
            path: /
            port: 9090
          initialDelaySeconds: 30
          timeoutSeconds: 30
{% endhighlight %}

release包中源文件没有`args: - --apiserver-host=http://10.0.63.202:8080`这一行，直接运行会出错（CrashLoopBackOff ），导致pod创建不成功，添加后创建正常。

错误报告：[Cannot configure apiserver URL][issue484]

解决办法：[Clarify how users can change apiserver-host in canary deployment][method]

*dashboard-service.yaml*

{% highlight yaml %}
# This file should be kept in sync with cluster/gce/coreos/kube-manifests/addons/dashboard/dashboard-service.yaml
apiVersion: v1
kind: Service
metadata:
  name: kubernetes-dashboard
  namespace: kube-system
  labels:
    k8s-app: kubernetes-dashboard
    kubernetes.io/cluster-service: "true"
spec:
  type: NodePort
  ports:
  - port: 80
    targetPort: 9090
  selector:
    k8s-app: kubernetes-dashboard
{% endhighlight %}

<h4>Cluster monitoring</h4>

kubernetes/cluster/addons/cluster-monitoring目录包含安装平台监控的yaml文件，由`Heapster`、`InfluxDB`、`GCE`、`Grafana`等几个容器构成，`Heapter`负责收集Kubernetes运行平台的监控数据，然后导入`InfuxDB`或`GCE`，`InfluxDB`是一个分布式时序、事件和指标数据库，而Grafana负责将数据进行图表展示。

目录下的`heapster-controller.yaml`文件中的yaml configs并不是让`kubectl create -f`直接使用的，只能通过[salt][salt]在创建集群时使用，我们可以将这些yaml config去掉，并根据集群的资源状况将模板表达式替换为合理的值。详见[kubernetes issue 21065][issue_21065]。

部署过程中会有一些错误：

{% highlight shell %}
[root@anakin ~]# kubectl logs heapster-v1.1.0-919384262-dhxqr --namespace kube-system
Error from server: a container name must be specified for pod heapster-v1.1.0-919384262-dhxqr, choose one of: [heapster eventer heapster-nanny eventer-nanny]
{% endhighlight %}
由于对heapster不是很了解，目前还没有找到好的解决办法，等找到后会进行更新，现在先这样了:(

<h4>Node Problem Detector</h4>

这个扩展组件是一个DeamonSet，即在每个Node节点上都会启动一个Pod，用于检测Node的状态并将问题汇报给apiserver，由于是一个DeamonSet，因此需要更高的权限，在kubelet和kube-apiserver的启动选项中都添加上`--allow-privileged=true`，否则会在创建时遇到错误：

{% highlight shell %}
[root@anakin ~]# kubectl create -f node-problem-detector.yaml 
The DaemonSet "node-problem-detector-v0.1" is invalid.
spec.template.spec.containers[0].securityContext.privileged: Forbidden: disallowed by policy
{% endhighlight %}

友情提示 => 遇到pod不能正常启动一定要多查看日志，常用命令：
{% highlight shell %}
$ kubectl logs <podid> [--namespace kube-system]
$ kubectl describe pod <podid> [--namespace kube-system]
$ docker inspect <dockerid>
{% endhighlight %}


[k8s_cluster]: http://zhjwpku.com/docker/2016/08/30/k8s-deploy-a-3-nodes-cluster.html
[issue484]: https://github.com/kubernetes/dashboard/issues/484
[method]: https://github.com/bryk/dashboard/commit/96305abe4bb1f14b260c6a937d566de2a9728e27
[salt]: https://saltstack.com/
[issue_20165]: https://github.com/kubernetes/kubernetes/issues/21065
