# minikube-aci
minikube using virtual kubelet extend pods to ACI

# 环境准备
* win10 pro或企业版，启用hyper-v
* Azure账号
* 下载Azure CLI
* 下载minikube
* 下载kubectl
* 下载helm

Tips：
* helm版本用的v2，minikube start时指定kubernetes版本为v1.15.x，本例中用的是1.15.7，helm版本用的v2.16.3。minikube本身可以使用目前的最新版本v1.7.3，重要的是指定kubernetes的版本。

主要因用到的virtual kubelet的chart： [https://github.com/virtual-kubelet/virtual-kubelet/raw/master/charts/virtual-kubelet-latest.tgz](https://github.com/virtual-kubelet/virtual-kubelet/raw/master/charts/virtual-kubelet-latest.tgz) 在网站上看到的日期已经是“16 months ago”了，并且在使用helm 3 install过程中报错: 
> Error: unable to build kubernetes objects from release manifest: unable to recognize "": no matches for kind "Deployment" in version "extensions/v1beta1"

使用上述版本，可正常完成。

* 在hyper-v中，需要提前创建一个可以使用外网的virtual switch，供minikube启动时使用，便于minikube VM中的kubernetes与外网通讯。

# minikube启动，创建kubernetes的VM
minikube start时需要注意几个参数：
* --hyperv-virtual-switch=extswitch，extswitch即前面提到的提前创建的外网switch
* --image-mirror-country=cn，指定cn，因众所周知的原因，便于下载镜像
* --kubernetes-version=1.15.7，前面提到的因virtual kubelet的chart版本，选择1.15的版本，默认是1.17的会报错
```cmd
minikube start --vm-driver=hyperv --hyperv-virtual-switch=extswitch --cpus=2 --memory=4096 --image-mirror-country=cn --kubernetes-version=1.15.7
```
耐心等待minikube下载所需镜像后启动vm。我们可以先准备下面Azure的步骤

# 准备Azure资源
创建一个资源组，minikube通过virtual kubelet创建的ACI将放在这个资源组中。
```CMD
az group create --name aci-group --location eastus
```
命令执行后会类似返回如下信息（将敏感信息subscriptions id以xxx代替，后续步骤也会使用到，可以提前记下）：
```CMD
az group create --name aci-group --location eastus
{
"id": "/subscriptions/xxx/resourceGroups/aci-group",
"location": "eastus",
"managedBy": null,
"name": "aci-group",
"properties": {
"provisioningState": "Succeeded"
},
"tags": null,
"type": "Microsoft.Resources/resourceGroups"
}
```

创建一个service principal，即sp，用于权限授予。其中的参数scope后面的resource id即上面resource group的id。这个sp创建后，需要记录下返回的所有信息，后面需要使用，像appId，password，tenant等信息。
```CMD
az ad sp create-for-rbac --name vk-aci-demo --scope <resource id> --role contributor
```
命令执行后会类似返回如下信息（将敏感信息以xxx代替）：
```CMD
az ad sp create-for-rbac --name vk-aci-demo --scope /subscriptions/xxx/resourceGroups/aci-group --role contributor
Changing "vk-aci-demo" to a valid URI of "http://vk-aci-demo", which is the required format used for service principal names
Creating a role assignment under the scope of "/subscriptions/xxx/resourceGroups/aci-group"
{
  "appId": "ce45db6e-xxx-8fa78c8ce599",
  "displayName": "vk-aci-demo",
  "name": "http://vk-aci-demo",
  "password": "3b33e5ca-xxx-9a20d7defb0e",
  "tenant": "72f988bf-xxx-2d7cd011db47"
}
```

# helm安装virtual kubelet的chart
等minikube正常启动后，准备安装chart。安装前可以查看一下当前node和pod状态：
```
kubectl get node
kubectl get pod
```

helm初始化，同时指定repo。推荐大家Azure国内的镜像站点：[https://mirror.azure.cn/](https://mirror.azure.cn/) 值得拥有
```
helm init --tiller-image gcr.azk8s.cn/kubernetes-helm/tiller:v2.16.3 --stable-repo-url https://mirror.azure.cn/kubernetes/charts/
```

找到minikube中kubernetes master的地址：
```CMD
kubectl cluster-info
```

准备helm的命令参数，创建sp时返回的信息中clientId即appId，clientKey即password。

另外chart文件https://github.com/virtual-kubelet/virtual-kubelet/raw/master/charts/virtual-kubelet-latest.tgz在某些地方访问github的问题可能下载失败，可提前自行科学下载到本地（你懂得），指定本地文件路径即可：
```CMD
helm install --name vk-aci https://github.com/virtual-kubelet/virtual-kubelet/raw/master/charts/virtual-kubelet-latest.tgz ^
  --set provider=azure ^
  --set rbac.install=true ^
  --set providers.azure.targetAKS=false ^
  --set providers.azure.aciResourceGroup=aci-group ^
  --set providers.azure.aciRegion=eastus ^
  --set providers.azure.tenantId=72f988bf-xxx-2d7cd011db47 ^
  --set providers.azure.subscriptionId=31fa638d-xxx-5d8853d773d4 ^
  --set providers.azure.clientId=ce45db6e-xxx-8fa78c8ce599 ^
  --set providers.azure.clientKey=3b33e5ca-xxx-9a20d7defb0e ^
  --set providers.azure.masterUri=https://192.168.36.16:8443
```
安装完成后：
```CMD
helm install --name vk-aci https://github.com/virtual-kubelet/virtual-kubelet/raw/master/charts/virtual-kubelet-latest.tgz ^
More?   --set provider=azure ^
More?   --set rbac.install=true ^
More?   --set providers.azure.targetAKS=false ^
More?   --set providers.azure.aciResourceGroup=aci-group ^
More?   --set providers.azure.aciRegion=eastus ^
More?   --set providers.azure.tenantId=xxx ^
More?   --set providers.azure.subscriptionId=xxx ^
More?   --set providers.azure.clientId=xxx ^
More?   --set providers.azure.clientKey=xxx ^
More?   --set providers.azure.masterUri=https://192.168.36.16:8443
NAME:   vk-aci
LAST DEPLOYED: Wed Feb 26 13:08:30 2020
NAMESPACE: default
STATUS: DEPLOYED

RESOURCES:
==> v1/Pod(related)
NAME                                   AGE
vk-aci-virtual-kubelet-578656b7-rmnn7  0s

==> v1/Secret
NAME                    AGE
vk-aci-virtual-kubelet  0s

==> v1/ServiceAccount
NAME                    AGE
vk-aci-virtual-kubelet  0s

==> v1beta1/ClusterRoleBinding
NAME                    AGE
vk-aci-virtual-kubelet  0s

==> v1beta1/Deployment
NAME                    AGE
vk-aci-virtual-kubelet  0s


NOTES:
The virtual kubelet is getting deployed on your cluster.

To verify that virtual kubelet has started, run:

  kubectl --namespace=default get pods -l "app=virtual-kubelet"

Note:
TLS key pair not provided for VK HTTP listener. A key pair was generated for you. This generated key pair is not suitable for production use.

```

查看安装chart后的node和pod状态：
```
helm list
kubectl get node -o wide
kubectl get pod -o wide
```

# 测试ACI
都完成后我们对是否能将pod创建到Azure的ACI上去进行测试。

测试前，新建的资源组aci-group下应该是空的，没有任何container实例：
```
az container list -g aci-group -o table
```

准备一个hello world的yaml文件:
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: aci-helloworld
spec:
  replicas: 1
  selector:
    matchLabels:
      app: aci-helloworld
  template:
    metadata:
      labels:
        app: aci-helloworld
    spec:
      containers:
      - name: aci-helloworld
        image: microsoft/aci-helloworld
        ports:
        - containerPort: 80
      nodeSelector:
        kubernetes.io/role: agent
        beta.kubernetes.io/os: linux
        type: virtual-kubelet
      tolerations:
      - key: virtual-kubelet.io/provider
        operator: Exists
      - key: azure.com/aci
        effect: NoSchedule
```
使用yaml文件部署：
```
kubectl apply -f aci-helloworld-deployment.yaml
```

查看部署后的状态：
```
kubectl get pod -o wide
az container list -g aci-group -o table
```

再次使用nginx部署多个replica进行测试：
```
apiVersion: v1
kind: Service
metadata:
  name: nginx
  labels:
    run: nginx
spec:
  selector:
    app: nginx
  type: LoadBalancer
  ports:
  - name: nginx
    port: 8080
    targetPort: 80
    protocol: TCP
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx
spec:
  replicas: 2
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx
        ports:
        - containerPort: 80
      nodeSelector:
        kubernetes.io/role: agent
        beta.kubernetes.io/os: linux
        type: virtual-kubelet
      tolerations:
      - key: virtual-kubelet.io/provider
        operator: Exists
      - key: azure.com/aci
        effect: NoSchedule
```

```
kubectl apply -f nginx.yaml
kubectl get svc
```

会发现nginx的2个pod部署到Azure上去了，有外网ip可以直接访问。LB作为service部署在本地node上了，那本地的这个LB可以起作用吗？

首先修改nginx的页面，便于测试LB：
```
kubectl exec nginx-xxx-1 -c nginx -it -- /bin/bash
echo "Hello World from nginx 1 !" | tee /usr/share/nginx/html/index.html
kubectl exec nginx-xxx-2 -c nginx -it -- /bin/bash
echo "Hello World from nginx 2 !" | tee /usr/share/nginx/html/index.html
```

本地起一个pod，并安装curl，测试访问LB，执行多次curl看看结果如何：
```
kubectl run --generator=run-pod/v1 -it --rm test --image=debian
apt-get update && apt-get install -y curl
curl -L http://<lb-ip>:8080
```

这个功能适合本地K8S cluster面临急需扩容的需求时，可以将Azure上容器实例ACI作为一个云上的资源池，秒级快速的进行扩展，以便不影响业务运行，适合在ACI上部署偏计算型的任务。

参考文档：
* [https://github.com/virtual-kubelet/virtual-kubelet](https://github.com/virtual-kubelet/virtual-kubelet)
* [https://github.com/virtual-kubelet/azure-aci](https://github.com/virtual-kubelet/azure-aci)
* [https://docs.microsoft.com/azure/aks/virtual-nodes-cli](https://docs.microsoft.com/azure/aks/virtual-nodes-cli)
