# glusterfs集群搭建

> 前言，目前网络上给出的glusterfs的部署有三种方式：
> 1、	在kubernetes中部署glusterfs-server     
> 2、	基于gluster-kubernetes源码部署      
https://github.com/gluster/gluster-kubernetes.git       
> 3、	在宿主机和节点上部署glusterfs集群，交由heketi管理，然后由k8s调用

***   
> 采用第三种方式部署

> ## 1、部署glusterfs前准备工作

> 1、磁盘：

>> 将磁盘格式化：mkfs.xfs -i size=512 /dev/sdb

>> 如果已经使用过的磁盘需要先去除挂载，然后格式化。       
>> 如果格式化后heketi创建集群时还出错执行    wipefs -a /dev/vdb

      注意：如果磁盘已经被挂载使用，再次取消挂载并格式化时。需要手动删除/etc/fstab目    
      录下已经取消挂载的记录，否则可能会造成服务器重启失败，导致网卡不能用。实际搭建过     程中遇到这个问题了     

      上面这个命令去除格式化造成的一些痕迹，恢复原始盘的状态。   

> 2、节点准备
>> 设置/etc/hosts文件下的节点ip值（用内网ip）域名用主机名如下：

```xml  
        172.17.0.24  k8s-01
	172.17.0.25  k8s-02
	172.17.0.26  k8s-03   

```    

> 3、关闭系统配置


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

>> ```systemctl start glusterfs-server```   
>> ```systemctl enable glusterfs-server```

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
  


	
  



