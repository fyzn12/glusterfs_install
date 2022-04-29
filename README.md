# 1. 镜像下载(集群节点都需要下载以下7个镜像)

```shell    

#!/bin/bash    
images=(   
	kube-apiserver:v1.23.4   
	kube-proxy:v1.23.4   
	kube-controller-manager:v1.23.4   
	kube-scheduler:v1.23.4   
	coredns:v1.8.6    
	etcd:3.5.1-0    
	pause:3.6   
)   
for imageName in ${images[@]} ; do    
docker pull registry.cn-hangzhou.aliyuncs.com/google_containers/$imageName    
done    
  
```    
## 处理镜像下载失败

> docker pull cnplat/kube-controller-manager:v1.23.4   
> docker pull cnplat/kube-apiserver:v1.23.4    
> docker pull cangyin/etcd:3.5.1-0     
> docker tag cnplat/kube-apiserver:v1.23.4 registry.cn-hangzhou.aliyuncs.com/google_containers/kube-apiserver:v1.23.4    
> docker tag cnplat/kube-controller-manager:v1.23.4 registry.cn-hangzhou.aliyuncs.com/google_containers/kube-controller-manager:v1.23.4     
> docker tag cangyin/etcd:3.5.1-0 registry.cn-hangzhou.aliyuncs.com/google_containers/etcd:3.5.1-0



# 2.配置(所有节点都要配置)

> 集群节点和master节点都要设置/etc/hosts
> vim /etc/hosts     
> ip                 k8s-node1   
> ip                 k8s-master   
> ip                 k8s-node2
>

## 2.1 配置内核参数，将桥接的IPv4流量传递到iptables的链
> cat <<EOF | tee /etc/sysctl.d/k8s.conf    
> net.bridge.bridge-nf-call-ip6tables = 1     
> net.bridge.bridge-nf-call-iptables = 1     
> EOF
### 2.1.1 手动加载配置文件
> sysctl --system
## 2，2 防火墙关闭
> systemctl stop firewalld   
> systemctl disable firewalld
## 2.3 将 SELinux 设置为 permissive 模式（相当于将其禁用）
> setenforce 0(临时)     
> sed -i 's/^SELINUX=enforcing$/SELINUX=permissive/' /etc/selinux/config(永久)
## 2.4 关闭交换空间
> swapoff -a (临时)   
> sed -i 's/.*swap.*/#&/' /etc/fstab (永久)
## 2.5 设置时区
> timedatectl set-timezone Asia/Shanghai
#### 2.5.1 将当前的 UTC 时间写入硬件时钟

> timedatectl set-local-rtc 0

#### 2.5.2 重启依赖于系统时间的服务

> systemctl restart rsyslog    
> systemctl restart crond

# 3. k8s集群搭建

## 3.1 安装kubelet、kubeadm、kubectl

### 3.1.1 设置kubernetes.repo(all node)

> cat > /etc/yum.repos.d/kubernetes.repo << EOF   
> [kubernetes]    
> name=Kubernetes   
> baseurl=https://mirrors.aliyun.com/kubernetes/yum/repos/kubernetes-el7-x86_64   
> enabled=1   
> gpgcheck=0   
> repo_gpgcheck=0   
> gpgkey=https://mirrors.aliyun.com/kubernetes/yum/doc/yum-key.gpg https://mirrors.aliyun.com/kubernetes/yum/doc/rpm-package-key.gpg    
> EOF
>

### 3.1.2 安装(all node)

> yum install -y kubelet-1.23.4  kubeadm-1.23.4  kubectl-1.23.4
>
### 3.1.3 设置重启

> systemctl enable kubelet

## 3.2 初始化集群(only master)

> kubeadm init \   
> --apiserver-advertise-address=172.31.210.220 \    
>  --image-repository registry.cn-hangzhou.aliyuncs.com/google_containers \   
> --kubernetes-version v1.23.4 \   
> --pod-network-cidr=10.244.0.0/16   
> --service-cidr=10.96.0.0/12 \   
> --ignore-preflight-errors=all

> 执行成功后安装流程执行  
> mkdir -p $HOME/.kube    
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config     
sudo chown $(id -u):$(id -g) $HOME/.kube/config

### 3.2.1 节点加入到master(only node)
> kubeadm join 172.31.210.220:6443 --token qai2i3.mmgbn9e6kfw3q7qy \   
> --discovery-token-ca-cert-hash sha256:c8133ab7035649296a283f63fbc836eae3468ad0770b3a39a091b04d81261ee4

### 3.2.2 安装网络组件（仅在master上执行 only master）

> wget https://docs.projectcalico.org/manifests/calico.yaml

> vim calico.yaml  
> 搜索 CALICO_IPV4POOL_CIDR  
> 将对应的value的地址改成kubeadm init 的pod-network-cidr所对呀的IP

> kubectl apply -f calico.yaml   
> 等待网络组件安装完毕  
> kubectl get node 集群搭建完毕

# 4. k8s卸载(all)

> kubeadm reset -f   
> rm -rvf $HOME/.kube    
> rm -rvf ~/.kube/    
> rm -rvf /etc/kubernetes/   
> rm -rvf /etc/systemd/system/kubelet.service.d   
> rm -rvf /etc/systemd/system/kubelet.service   
> rm -rvf /usr/bin/kube*   
> rm -rvf /etc/cni   
> rm -rvf /opt/cni   
> rm -rvf /var/lib/etcd   
> rm -rvf /var/etcd   
> yum remove kube*


# glusterfs集群搭建

> 前言，目前网络上给出的glusterfs的部署有三种方式：
> 1、	在kubernetes中部署glusterfs-server     
> 2、	基于gluster-kubernetes源码部署      
https://github.com/gluster/gluster-kubernetes.git       
> 3、	在宿主机和节点上部署glusterfs集群，交由heketi管理，然后由k8s调用

***   
> 采用第三种方式部署

> ## 1、部署glusterfs前准备工作

> 1、磁盘：(all node)

>> 将磁盘格式化：mkfs.xfs -i size=512 /dev/sdb

>> 如果已经使用过的磁盘需要先去除挂载，然后格式化。       
>> 如果格式化后heketi创建集群时还出错执行    wipefs -a /dev/vdb

      注意：如果磁盘已经被挂载使用，再次取消挂载并格式化时。需要手动删除/etc/fstab目    
      录下已经取消挂载的记录，否则可能会造成服务器重启失败，导致网卡不能用。实际搭建过     程中遇到这个问题了     

      上面这个命令去除格式化造成的一些痕迹，恢复原始盘的状态。   

> 2、节点准备 (all node)
>> 设置/etc/hosts文件下的节点ip值（用内网ip）域名用主机名如下：

```xml  
        172.17.0.24  k8s-01
	172.17.0.25  k8s-02
	172.17.0.26  k8s-03   

```    

> 3、关闭系统配置(all node)


>> 关闭防火墙   
>> $ systemctl stop firewalld   
>> $ systemctl disable firewalld   
>> 关闭selinux：   
>> $ sed -i 's/enforcing/disabled/' /etc/selinux/config  # 永久    
>> $ setenforce 0  # 临时

> ## 2、部署glusterfs

> 1、 所有节点上执行
>> 安装

>> ```yum install centos-release-gluster glusterfs-server  glusterfs-client fuse -y```
>
>> 启动

>> ```systemctl start glusterd```   
>> ```systemctl enable glusterd```

> 2、 宿主机(master节点)上执行，或者任一节点
>> 执行  
>> ```gluster peer probe k8s-02```      
>> ```gluster peer probe k8s-03```

> 3、 查看gluster集群状态
>
>> ```gluster peer status```

	注： 确保集群成功搭建，如果集群没有部署成功，考虑/etc/hosts文件中是否配置了内网映射    

> ## 3、heketi配置

> 1、安装heketi

>> ```yum install heketi heketi-client  -y```

> 2、 配置ssh密钥
>> ```ssh-keygen -f /etc/heketi/heketi_key -t rsa -N ''```    
>> ```chmod 600 /etc/heketi/heketi_key.pub```

>>  ssh公钥传递，这里只以一个节点为例     
>> ```ssh-copy-id -i /etc/heketi/heketi_key.pub root@ k8s-02```
>
>> 验证是否能通过ssh密钥正常连接到glusterfs节点   
>> ```ssh -i /etc/heketi/heketi_key root@ mopaas-wang-02```

> 3、 修改heketi的配置文件

>> ```vim /etc/heketi/heketi.json```

```json  

	{    
		  "port": "18080",#避免端口冲突       
		  "use_auth": false,  #若要启动连接Heketi的认证，则需要将user_auth改为      
					  true，并在jwt{}段为各用户设置相应的密码，用户名和密码都可以自定义       
		
		  "jwt": {    
		    "admin": {    
		      "key": "My Secret"    
		    },
		    "user": {   
		      "key": "My Secret"   
		    }
		  },
		
		  "glusterfs": {  
		    "executor": "ssh",   
		    "sshexec": {
		      "keyfile": "/etc/heketi/heketi_key",   
		      "user": "root",   
		      "port": "22",
		      "fstab": "/etc/fstab"  
		},
		# 默认的挂载点，我修改成其他的挂载点时报错，可能是与我的heketi.db文件有关    
		    "db": "/var/lib/heketi/heketi.db",    
		    "loglevel" : "debug"   
		  }  
	}  

	注意文件中有很多内容，避免出错，修改上面我给出的及可

```  

> 4、启动heketi
>> systemctl enable heketi && systemctl start heketi     
>> systemctl status heketi      
>>   如果json文件修改出错这里是启动不了的

> 5、测试heketi是否启动成功

>> curl http://172.17.0.24:18080/hello  
>> 如果启动成功会返回   Hello from Heketi   的字样表示集群搭建成功

> 6、暴露heketi的端口
>
>> echo "export HEKETI_CLI_SERVER=http://172.17.0.24:18080" > /etc/profile.d/heketi.sh   
>> source /etc/profile.d/heketi.sh


> 7、设置Heketi系统拓扑
>
> (1)编写topology-sample.json文件   位置放在 /etc/heketi/下

```json  
  
	{
	    "clusters": [
	        {
	            "nodes": [
	                {
	                    "node": {
	                        "hostnames": {
	                            "manage": [
	                                "172.17.0.24"
	                            ],
	                            "storage": [
	                                "172.17.0.24"
	                            ]
	                        },
	                        "zone": 1
	                    },
	                    "devices": [
	                        {
	                            "name": "/dev/vdb",
	                            "destroydata": false
	                        }
	                    ]
	                },
	                {
	                    "node": {
	                        "hostnames": {
	                            "manage": [
	                                "172.17.0.25"
	                            ],
	                            "storage": [
	                                "172.17.0.25"
	                            ]
	                        },
	                        "zone": 2
	                    },
	                    "devices": [
	                        {
	                            "name": "/dev/vdb",
	                            "destroydata": false
	                        }
	                    ]
	                },
	                {
	                    "node": {
	                        "hostnames": {
	                            "manage": [
	                                "172.17.0.26"
	                            ],
	                            "storage": [
	                                "172.17.0.26"
	                            ]
	                        },
	                        "zone": 1
	                    },
	                    "devices": [
	                        {
	                            "name": "/dev/vdb",
	                            "destroydata": false
	                        }
	                    ]
	                }
	            ]
	        }
	    ]
	}

```  

>> Devices的name就是上面准备好的空的磁盘

> (2)加载拓扑完成集群搭建  在目录/etc/heketi/下执行  或者json后面指定全路径    
heketi-cli topology load --json=topology-sample.json

> 8、查看集群详情

>> heketi-cli topology info


## k8s中使用heketi及gluster集群

### 1 编写storageclass及secret

```yaml  
   
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
   name: glusterfs-test-zrj
   namespace: esdata
provisioner: kubernetes.io/glusterfs
parameters:
   resturl: "http://172.17.0.24:18080"
   clusterid: "1b919fc46fe70fbbdb64ab2c49c03657"
   restauthenabled: "true"
   restuser: "admin"
   secretNamespace: "esdata"
   secretName: "heketi-secret"
   #restuserkey: "adminkey"
   #gidMin: "10000"
   #gidMax: "50000"
   volumetype: "replicate:2"     

    ---  
apiVersion: v1
kind: Secret
metadata:
	name: heketi-secret
	namespace: esdata
data:
  # base64 encoded password. E.g.: echo -n "mypassword" | base64
	key: TFRTTkd6TlZJOEpjUndZNg==
type: kubernetes.io/glusterfs  


```    

### 2 创建pvc

```yaml  
    
kind: PersistentVolumeClaim
apiVersion: v1
metadata:
   name: es-data-pvc
   namespace: esdata
   annotations:
	volume.beta.kubernetes.io/storage-class: "glusterfs-test-zrj"
spec:
   accessModes:
	 - ReadWriteMany
   resources:
	requests:
	  storage: 2Gi
  #storageClassName: glusterfs-test-zrj  
  
```  
  


	
  



