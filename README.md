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

部署方式

Metrics Server安装清单随GitHub版本一起上传。

它们可作为Metrics Server版本上的metrics-server.yaml资产提供，从而可通过URL安装：

根据您的群集设置，您可能还需要更改传递给Metrics Server容器的标志。 最有用的标志：

     --kubelet-preferred-address-types-确定连接到特定节点的地址时使用的节点地址类型的优先级（默认[主机名，内部DNS，内部IP，外部DNS，外部IP]）
     --kubelet-insecure-tls-不要验证Kubelets所提供的服务证书的CA。 仅用于测试目的。

另外需要在/etc/kubernetes/manifests/kube-apiserver配置添加

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

