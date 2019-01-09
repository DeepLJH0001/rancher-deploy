## Rancher2.x HA版本部署指导
### 一、架构说明  
![Image text](./images/rancher-ha.jpg)
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
1. 安装docker   
以下脚本保存为pre-install.sh分别在每一个节点上执行，这里安装的Docker版本为17.03.2~ce
```sh
# !/bin/sh
sudo apt-get update
sudo apt-get install -y \
    apt-transport-https \
    ca-certificates \
    curl \
    software-properties-common
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo apt-key add -
sudo apt-key fingerprint 0EBFCD88
sudo add-apt-repository \
   "deb [arch=amd64] https://download.docker.com/linux/ubuntu \
   $(lsb_release -cs) \
   stable"
sudo apt-get update
# 安装17.03.02版本的docker
sudo apt-get install -y docker-ce=17.03.2~ce-0~ubuntu-xenial
# 禁用SELinux，Ubuntu默认未安装，无需设置。
# *可以先安装selinux工具包，然后使用getenforce工具查看SELinux状态。
sudo apt install -y selinux-utils
sudo getenforce
# 关闭ufw防火墙，Ubuntu默认未启用,无需设置。手工关闭UFW：
sudo ufw disable
# 设置RKE的运行帐号，Ubuntu可以使用root帐号运行，这里新建一个帐号来运行
mkdir -p /home/hwrancher
groupadd -g 204 hwrancher
useradd -u 204 -g 204 -d /home/hwrancher hwrancher
chown -R 204:204 /home/hwrancher
sudo usermod -aG docker hwrancher
```
2. 节点互信  
分别在安装节点上执行，让每一台机器之间节点能通过证书，免密互信登录
```sh
ssh-keygen 
ssh-copy-id hwrancher@192.168.1.100
ssh-copy-id hwrancher@192.168.1.101
ssh-copy-id hwrancher@192.168.1.102
ssh-copy-id hwrancher@192.168.1.103
```

3. 启用cgroup内存和Swap限额  

```sh
vi /etc/default/grub
# 1.修改这两项内容
GRUB_CMDLINE_LINUX_DEFAULT="cgroup_enable=memory swapaccount=1"
GRUB_CMDLINE_LINUX="cgroup_enable=memory swapaccount=1"
sudo update-grub
# 2.重启有效
reboot
```

4. 设置内网互通  
在每一个节点的安全组上添加入方向 192.168.1.0/16 远端所有端口，同时对 0.0.0.0/0 开通80,443端口

### 五、部署安装  
1. 自签名ssl证书  
这个可以参考：[查看详情](https://www.cnrancher.com/docs/rancher/v2.x/cn/installation/self-signed-ssl/)  这里注意一点，把里面的demo.rancher.com替换成自己的域名就可以了；
补充说明：由于当前证书是自签名证书，访问时浏览器此站点会提示不安全；
2. 安装Harbor  
这个可以参考：[查看详情](https://www.cnrancher.com/docs/rancher/v2.x/cn/installation/registry/single-node-installation/)，要对接华为云的SWR服务，推荐安装1.5.3版本，可以支持从Harbor与SWR之间的镜像同步操作。
3. 安装RKE
登录其中一台机器如：192.168.1.100  
- step1: 下载KRE二进制文件  
```sh
wget https://github.com/rancher/rke/releases/download/v0.1.15/rke_linux-amd64
mv rke_linux-amd64 rke
chmod +x rke
./rke --version
```
- step2: 配置rancher-cluster.yml文件  
在当前目录下vi rancher-cluster.yml编辑，并把以下内容保存到文件中
```yaml
nodes:
  - address: 192.168.1.100
    user: hwrancher
    role: [etcd]
  - address: 192.168.1.101
    user: hwrancher
    role: [etcd]
  - address: 192.168.1.102
    user: hwrancher
    role: [controlplane,worker]
  - address: 192.168.1.103
    user: hwrancher
    role: [controlplane,worker]
```
- step3: RKE执行安装kubernetes  
```sh
./kre up --config rancher-cluster.yml
```
等待5分钟左右，直到没有报错，代表kubernetes等相关安装成功。
- step4: 配置kubectl  
安装配置kubectl [详细查看](https://www.cnrancher.com/docs/rancher/v2.x/cn/installation/kubectl/) 
注：国内网络访问不了 google.com，需要配置海外代理，或离线方式安装。
```sh
mkdir -p ~/.kube
mv kube_config_cluster.yml ~/.kube/config
kubectl cluster-info 
```
3. 安装Helm和Tiller  
国内网络访问不了 google.com，需要从配置海外代理服务器或提前从海外把需要的镜像包拉下来
安装指导参考 [详细查看](https://www.cnrancher.com/docs/rancher/v2.x/cn/installation/server-installation/ha-install/helm-rancher/helm-install/)

4. 通过Helm安装Rancher  
- 安装证书管理器  
```sh 
helm install stable/cert-manager \
  --name cert-manager \
  --namespace kube-system
``` 
- 安装rancher 
```sh
helm install rancher-stable/rancher \
  --name rancher \
  --namespace cattle-system \
  --set hostname=xxx.xxx.com  #配置rancher的域名
``` 

5. rancher页面
![Image text](./images/rancher-login.jpg)
![Image text](./images/rancher-home.jpg)

