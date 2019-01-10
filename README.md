## Rancher2.x HA版本部署指导
### 一、架构说明  
![框架说明](./images/rancher-ha.jpg)
### 二、配置要求   

根据自己的诉求来选择配置节点
| 部署大小 | 集群(个) | 节点(个)|vCPU|内存|  
| ----- | --------- | ----------- | ------- |------- |  
|小	|不超过5	|最多50	|4C	|16GB|  
|中	|不超过100	|最多500	|8C	|32GB|  
|大	|超过100	|超过500	|[联系Rancher](https://www.cnrancher.com/contact/)| |

### 三、节点规划  

申请4个VM节点，配置为C3 4C16G，系统Ubuntu 16.04 server 64bit
这里部署把etcd的节点独立出来，由于当前所有的数据都存储在etcd上

| 节点名称 | 节点配置 | IP | 类型 | 备注 |  
| ----- | ----- | --------- | ----------- | ------- |
| rancher-etcd1 |4C16G,100G存储	|192.168.1.100	|etcd	|	|
| rancher-etcd2 |4C16G,100G存储	|192.168.1.101	|etcd	|	|
| rancher-controlplane-work1 |4C16G,100G存储	|192.168.1.102	|controlplane,worker	| 管理面,数据面	|
| rancher-controlplane-work2 |4C16G,100G存储	|192.168.1.103	|controlplane,worker	| 管理面,数据面	|

### 四、准备工作 
1. 申请CCE节点 
这里申请了两台C3 4C16G作为Rancher的Node节点，节点申请时把EIP也申请，后面安装下载镜像时需要使用到


### 五、部署安装  
1. 自签名ssl证书  
这个可以参考：[查看详情](https://www.cnrancher.com/docs/rancher/v2.x/cn/installation/self-signed-ssl/)  这里注意一点，把里面的demo.rancher.com替换成自己的域名就可以了；
补充说明：由于当前证书是自签名证书，访问时浏览器此站点会提示不安全；
```
openssl req \
-newkey rsa:4096 -nodes -sha256 -keyout ca.key \
-x509 -days 365 -out ca.crt

Country Name(2 letter code)[AU]: CN
State or Province Name(full name)[Some-State]: Beijing
Locality Name(eg, city)[]: Beijing
Organization Name(eg, company)[Internet Widgits Pty Ltd]: rancher
Organizational Unit Name(eg, section)[]: info technology
Common Name(e.g. server FQDN or YOUR name)[]: ca.rancher.com
Email Address []: xxx@qq.com
```
```
openssl req \
-newkey rsa:4096 -nodes -sha256 -keyout demo.rancher.com.key \
-out  demo.rancher.com.csr

Country Name(2 letter code)[AU]: CN
State or Province Name(full name)[Some-State]: Beijing
Locality Name(eg, city)[]: Beijing
Organization Name(eg, company)[Internet Widgits Pty Ltd]: rancher
Organizational Unit Name(eg, section)[]: info technology
Common Name(e.g. server FQDN or YOUR name)[]: demo.rancher.com
Email Address []: xxx@qq.com
```
***
注意: Commone Name一定要是你要授予证书的FQDN域名或主机名，并且不能与生成root CA设置的Commone Name相同，challenge password可以不填。
```
openssl x509 -req -days 365 -in demo.rancher.com.csr -CA ca.crt -CAkey ca.key -CAcreateserial -out demo.rancher.com.crt
```

2. 安装Helm  
2.1. 安装Helm 客户端  
2.1.1. 配置Helm客户端访问权限  
- 在kube-system命名空间中创建ServiceAccount；
- 创建ClusterRoleBinding以授予tiller帐户对集群的访问权限
- helm初始化tiller服务
```
kubectl -n kube-system create serviceaccount tiller
kubectl create clusterrolebinding tiller --clusterrole cluster-admin --serviceaccount=kube-system:tiller
```
在镜外先下载好Helm Client包[helm-v2.11.0-linux-amd64.tar.gz](https://storage.googleapis.com/kubernetes-helm/helm-v2.11.0-linux-amd64.tar.gz)这里面提前下载好，并放在安装节点的/root目录

解压可以直接使用
```
tar -zxvf helm-v2.11.0-linux-amd64.tar.gz
mv linux-amd64/helm /usr/local/bin/helm
helm version
```
helm version 命令出现以下返回，说明helm客户端已经安装好了
```
Client: &version.Version{SemVer:"v2.11.0", GitCommit:"2e55dbe1fdb5fdb96b75ff144a339489417b146b", GitTreeState:"clean"}
Error: could not find tiller
```
2.2. 安装Helm Server(Tiller)   
```
helm init --service-account tiller   --tiller-image registry.cn-hangzhou.aliyuncs.com/google_containers/tiller:v2.11.0 --stable-repo-url https://kubernetes.oss-cn-hangzhou.aliyuncs.com/charts
```
执行helm version查看helm server是否已经安装好了，出现以下提示，说明还有问题，需要再执行 yum install socat -y
```
Client: &version.Version{SemVer:"v2.11.0", GitCommit:"2e55dbe1fdb5fdb96b75ff144a339489417b146b", GitTreeState:"clean"}
E0110 10:17:43.734115   28869 portforward.go:331] an error occurred forwarding 36141 -> 44134: error forwarding port 44134 to pod eba675f8b8826a71d7983b59b7fa1745d4e6371fa46e610e72e289436af76356, uid : unable to do port forwarding: socat not found.
Error: cannot connect to Tiller
```
再执行helm version，出现以下返回说明helm server安装成功
```
Client: &version.Version{SemVer:"v2.11.0", GitCommit:"2e55dbe1fdb5fdb96b75ff144a339489417b146b", GitTreeState:"clean"}
Server: &version.Version{SemVer:"v2.11.0", GitCommit:"2e55dbe1fdb5fdb96b75ff144a339489417b146b", GitTreeState:"clean"}
```

4. 通过Helm安装Rancher 
- 配置Chart仓库地址  
这里是安装生产环境的，选择stable版本作为生产环境的版本，不推荐使用latest作为生产环境版本
```
helm repo add rancher-stable https://releases.rancher.com/server-charts/stable
helm repo list
``` 
- 安装证书管理器  
```sh 
helm install stable/cert-manager \
  --name cert-manager \
  --namespace kube-system
``` 
- 私有CA签名证书
如果你使用的是私有CA，则需要传递CA文件给Rancher。
将CA证书复制到名为cacerts.pem的文件中，并用kubectl在命名空间cattle-system中创建tls-casecret
```
cp /root/ca.crt /root/cacerts.pem
cd /root
kubectl create namespace cattle-system
kubectl -n cattle-system create secret generic tls-ca \
  --from-file=cacerts.pem
```

- 安装rancher 
```sh
helm install rancher-stable/rancher \
  --name rancher \
  --namespace cattle-system \
  --set hostname=demo.rancher.com  #配置rancher的域名
``` 

```
kubectl -n cattle-system patch  deployments cattle-cluster-agent --patch '{
    "spec": {
        "template": {
            "spec": {
                "hostAliases": [
                    {
                        "hostnames":
                        [
                            "demo.rancher.com"
                        ],
                            "ip": "119.3.196.198"
                    }
                ]
            }
        }
    }
}'

kubectl -n cattle-system patch  daemonsets cattle-node-agent --patch '{
    "spec": {
        "template": {
            "spec": {
                "hostAliases": [
                    {
                        "hostnames":
                        [
                            "demo.rancher.com"
                        ],
                            "ip": "119.3.196.198"
                    }
                ]
            }
        }
    }
}'
```

5. rancher页面
![Image text](./images/rancher-login.jpg)
![Image text](./images/rancher-home.jpg)
