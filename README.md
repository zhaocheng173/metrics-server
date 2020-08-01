# metrics-server
Metrics server是一种可伸缩的、高效的容器资源指标源，用于Kubernetes内置的自动缩放管道。

Metrics server从Kubelets收集资源指标，并通过Metrics API在Kubernetes apiserver中公开它们，以便HPA和VPA使用。Metrics API也可以被kubectl top访问，这使得它更容易调试自动排序管道。

Metrics server并不用于非自动加码的目的。例如，不要使用它将指标转发到监视解决方案，或者将其作为监视解决方案指标的来源。

度量服务器提供:

在大多数集群上工作的单一部署(请参阅需求)
可扩展支持最多5,000个节点集群
资源效率:Metrics server每个节点使用0.5m核心CPU和4 MB内存


您可以将Metrics Server用于：

     基于CPU /内存的水平自动缩放（了解有关“Horizontal Pod Autoscaler”的更多信息）
     自动调整/建议容器所需的资源（了解有关Vertical Pod Autoscaler的更多信息）

在需要时不要使用Metrics Server：

     非Kubernetes集群
     准确的资源使用指标来源
     根据其他资源（然后是CPU /内存）进行水平自动缩放

对于不受支持的用例，请查看完整的监视解决方案，例如Prometheus。

要求
Metrics Server对群集和网络配置有特定要求。 这些要求并不是所有集群发行版的默认要求。 在使用Metrics Server之前，请确保您的群集分发支持这些要求：

     必须从kube-apiserver可以访问Metrics Server
     必须正确配置kube-apiserver才能启用聚合层
     节点必须配置kubelet授权以匹配Metrics Server配置
     容器运行时必须实现容器指标RPC


根据您的群集设置，您可能还需要更改传递给Metrics Server容器的标志。 最有用的标志这里我已经添加到deployment.yaml中：

     --kubelet-preferred-address-types-确定连接到特定节点的地址时使用的节点地址类型的优先级（默认[主机名，内部DNS，内部IP，外部DNS，外部IP]）
     --kubelet-insecure-tls-不要验证Kubelets所提供的服务证书的CA。 仅用于测试目的。

需要你去做的在/etc/kubernetes/manifests/kube-apiserver配置添加以下配置就可以了

     - --kubelet-preferred-address-types=InternalIP,ExternalIP,Hostname
     - --requestheader-client-ca-file=/etc/kubernetes/pki/front-proxy-ca.crt
     - --proxy-client-cert-file=/etc/kubernetes/pki/front-proxy-client.crt
     - --proxy-client-key-file=/etc/kubernetes/pki/front-proxy-client.key
     - --requestheader-allowed-names=front-proxy-client
     - --requestheader-client-ca-file=/etc/kubernetes/pki/front-proxy-ca.crt
     - --requestheader-extra-headers-prefix=X-Remote-Extra-
     - --requestheader-group-headers=X-Remote-Group
     - --requestheader-username-headers=X-Remote-User
     - --enable-aggregator-routing=true 

将上述配置添加到kube-apiserver当中之后，另外需要配置拉取metrics-server的生成文件，可直接git clone我的项目地址


它们可作为Metrics Server版本上的metrics-server.yaml资产提供，可直接下载git clone到你本地的linux服务器上支持x86服务器


     #git clone https://github.com/zhaocheng173/metrics-server.git
     正克隆到 'metrics-server'...
     remote: Enumerating objects: 13, done.
     remote: Counting objects: 100% (13/13), done.
     remote: Compressing objects: 100% (12/12), done.
     remote: Total 13 (delta 2), reused 0 (delta 0), pack-reused 0
     Unpacking objects: 100% (13/13), done.

直接部署

     # kubectl apply -f metrics-server.yaml
     clusterrole.rbac.authorization.k8s.io/system:aggregated-metrics-reader created
     clusterrolebinding.rbac.authorization.k8s.io/metrics-server:system:auth-delegator created
     rolebinding.rbac.authorization.k8s.io/metrics-server-auth-reader created
     apiservice.apiregistration.k8s.io/v1beta1.metrics.k8s.io created
     serviceaccount/metrics-server created
     deployment.apps/metrics-server created
     service/metrics-server created
     clusterrole.rbac.authorization.k8s.io/system:metrics-server created
     clusterrolebinding.rbac.authorization.k8s.io/system:metrics-server created
     
部署起来先看看metrics-server有没有正常工作，先看一下pod有没有错误日志，再看看有没有注册到聚合层
通过kubectl get apiservers,查看有没有注册到,这里为true才算正常

      # kubectl get apiservices
      v1beta1.metrics.k8s.io                 kube-system/metrics-server   True        8m2s

等待2-3分钟，查看部署结果可通过kubectl top node查看node资源的利用率

     # kubectl top node 
     NAME   CPU(cores)   CPU%   MEMORY(bytes)   MEMORY%   
     ma     2245m        28%    2801Mi          36%       
     mb     2453m        30%    2947Mi          18%       
     mc     2356m        58%    2374Mi          30%       
     md     308m         15%    709Mi           40%   
     
     
     # kubectl top pod -n kube-system
     NAME                                       CPU(cores)   MEMORY(bytes)   
     calico-kube-controllers-84445dd79f-q8cz4   8m           19Mi            
     calico-node-jpjv6                          107m         31Mi            
     calico-node-nqchp                          238m         46Mi            
     calico-node-qqkfp                          50m          32Mi   
     
也可以通过metrics api 的标识来获得资源使用率的指标，比如容器的cpu和内存使用率，这些度量标准既可以由用户直接访问，通过kubectl top命令，也可以由集群中的控制器pod autoscaler用于进行查看，hpa获取这个资源利用率的时候它是通过接口的，它请求的接口就是api,所以也可以根据api去获取这些数据，
测试：可以获取这些数据，这些数据和top看到的都是一样的，只不过这个是通过api 去显示的,只不过是通过json去显示的

      # kubectl get --raw /apis/metrics.k8s.io/v1beta1/nodes
      {"kind":"NodeMetricsList","apiVersion":"metrics.k8s.io/v1beta1","metadata":{"selfLink":"/apis/metrics.k8s.io/v1beta1/nodes"},"items":[{"metadata":       {"name":"mc","selfLink":"/apis/metrics.k8s.io/v1beta1/nodes/mc","creationTimestamp":"2020-08-01T05:01:40Z"},"timestamp":"2020-08-01T05:00:50Z","window":"30s","usage":{"cpu":"1803373919n","memory":"2335016Ki"}},{"metadata":{"name":"md","selfLink":"/apis/metrics.k8s.io/v1beta1/nodes/md","creationTimestamp":"2020-08-01T05:01:40Z"},"timestamp":"2020-08-01T05:00:56Z","window":"30s","usage":{"cpu":"215427325n","memory":"726568Ki"}},{"metadata":{"name":"ma","selfLink":"/apis/metrics.k8s.io/v1beta1/nodes/ma","creationTimestamp":"2020-08-01T05:01:40Z"},"timestamp":"2020-08-01T05:00:52Z","window":"30s","usage":{"cpu":"2265494861n","memory":"2786856Ki"}},{"metadata":{"name":"mb","selfLink":"/apis/metrics.k8s.io/v1beta1/nodes/mb","creationTimestamp":"2020-08-01T05:01:40Z"},"timestamp":"2020-08-01T05:00:49Z","window":"30s","usage":{"cpu":"2950012857n","memory":"2908168Ki"}}]}


也可以将命令行输出的格式转换成json格式----jq命令，如果没有jq，可以安装，这种方式可以直接以json可以展示

      # yum -y install epel-release
      # yum -y install jq
      # kubectl get --raw /apis/metrics.k8s.io/v1beta1/nodes |jq
      
