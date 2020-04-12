Mariadb Galera Cluster 集群镜像（下面简称MGC），仅支持Host网络模式。
### 相比官方镜像的改进点：
- 一键式下发，无人值守创建MGC集群；
- 支持MySQL（包含galera）全局任意参数设置；
- 支持镜像之外的任何自定义脚本，支持sh、sql、gz、env等；
- 成员节点故障恢复后自动探测，找到可用点后重新加入集群；
- 支持集群、单点以及普通三种模式，可以覆盖各种关系存储场景；
- 支持数据库初始化自定义设置，包含root、自定义账号、数据库、密码、主机等相关信息;
- 支持节点恢复时的传输限速（单位KB/s），解决官方镜像恢复顶满带宽问题，支持rsync / mariadbbackup2种同步方式限速。

### 使用方法：
#### 集群模式启动必须传入的参数
- my_port 默认3307，其他galera通信端口将使用前面加1、加2、加3约定规则，比如 gcast 采用13307。
- node1=192.168.1.100
- node2=192.168.1.101
- node3=192.168.1.102

#### 目前已支持的参数
- mysql_user                 创建用户
- mysql_user_database 用户权限DB
- mysql_user_password 用户密码
- mysql_user_grand       grand权限
- mysql_database          创建数据库
- mysql_root_host          root可远程的主机
- mysql_root_password root密码
- mysql_random_root_password 随机密码
- transfer_limit             传输限速，单位 kb/s
- cluster_name   集群名字
- cluster_mode   是否使用集群模式
- join_address   指定已存在的成员IP
- interface      指定绑定的网卡，用于集群自动探测时确认容器身份（nodeX），缺省优先级为：br0 > eth1 > eth0，若不符合要求请务必指定实际网卡名
- local_addr     指定本地IP，用于兼容本地网卡上绑定了VIP的特殊情况

#### 支持任意mysql、galera参数
MySQLl参数使用my_${参数名} 形式，比如：my_tmp_talbe_size=512M
Galera参数使用wsrep_${参数名} 形式，比如：wsrep_sst_method=rsync

#### 快速启动demo
在192.168.1.100、192.168.1.101、192.168.1.102三个节点上分别执行如下命令即可（不要求先后，会自动组建集群）：
```
docker run -d \
    --net=host \
    --name=demo-3310 \
    -e cluster_name=demo \
    -e my_port=3310 \
    -e node1=192.168.1.100 \
    -e node2=192.168.1.101 \
    -e node3=192.168.1.102 \
    -e mysql_user=demo \
    -e mysql_user_password=123456 \
    -v /data/mariadb-galera:/data/mariadb-galera \
   sinoprobe/mariadb-galera
```
执行后，可以执行 docker logs -f demo-3310 查看启动日志，也可以执行 tail -f /data/mariadb-galera/logs/error.log 查看运行日，启动成功后，可以执行如下命令查看集群状态：

`mysql -h192.168.1.100 -P3310 -udemo -p123456 -e "show status like '%wsrep%'"`

#### 批量创建(请事先创建3个地址分别为192.168.1.100，192.168.1.101，192.168.1.102的虚拟机并安装docker(详见 https://docs.docker.com/engine/install/centos/ )，配置docker API服务为2375端口(参考：https://www.cnblogs.com/nhz-M/p/11150607.html，配置防火墙！！！)
Python脚本:
```
#-*- coding:utf-8 -*-
# author:jagerzhang
import sys,re,os,time
import ConfigParser
import requests
import json
reload(sys)
sys.setdefaultencoding('utf8')
config  = ConfigParser.ConfigParser()

config.read('%s/config.cfg' % os.path.dirname(os.path.realpath(__file__)))

def gen_create_request(member_address=None):
    env_list=[]
    conf_list={}
    for conf in config.sections():
        for conf_name in config.options(conf):
            conf_value = config.get(conf,conf_name)
            env_list.append("%s=%s" % (conf_name,conf_value))
            conf_list[conf_name] = conf_value
    if member_address is not None:
        env_list.append("WSREP_MEMBER_ADDRESS=%s" % member_address)
    create_json = {
    "Env": env_list,
    "Image": "sinoprobe/mariadb-galera:10.3.12",
    "HostConfig": {
       "Binds": [
           "%s/%s-%s:/data/mariadb-galera" % (conf_list["mount_dir"],conf_list["cluster_name"],conf_list["my_port"]),
            "/etc/localtime:/etc/localtime"
        ],
        "Memory": int(conf_list["memory"])*1024*1024,
        "MemorySwap": -1,
        "CpusetCpus": "",
        "Privileged": True,
        "RestartPolicy": {
            "restart": "always"
        },
        "NetworkMode": "host"
        }
    }
    print create_json
    return conf_list,create_json

def create_container(host,cluster_name,my_port,create_json):
    headers = {'Content-type': 'application/json'}
    print "pulling image..."
    url          =  "http://%s:2375/v1.24/images/create?fromImage=sinoprobe/mariadb-galera&tag=10.3.12" % host
    result       =  requests.post(url)
    print result.text
    if result.status_code == 200:
        print "%s pull image success: %s" % (host,result.status_code)
        url          =  "http://%s:2375/containers/create?name=%s-%s"%(host,cluster_name,my_port)
        print "%s create container..." % host
        result       =  requests.post(url,data=json.dumps(create_json),headers=headers).json()
        try:
            url          =  "http://%s:2375/containers/%s/start" % (host,result['Id'])
            print "%s start container..." % host
            result       =  requests.post(url)
            if result.status_code == 204:
                print "%s start container success: %s" % (host,result.status_code)
                return 0
            else:
                print "%s start container failed: %s" % (host,result.status_code)
        except:
            print "%s create container failed: %s" % (host,result)
    else:
        print "%s pull image failed: %s" % (host,result.status_code)
    return result

def main():
    headers = {'Content-type': 'application/json'}
    conf_list,create_json = gen_create_request()
    node1        =  conf_list["node1"]
    node2        =  conf_list["node2"]
    node3        =  conf_list["node3"]
    cluster_name =  conf_list["cluster_name"]
    my_port         =  conf_list["my_port"]
    print create_container(node1,cluster_name,my_port,create_json)
    print create_container(node2,cluster_name,my_port,create_json)
    print create_container(node3,cluster_name,my_port,create_json)

main()
```
配置文件:
```
[global]
cluster_name=demo
node1=192.168.1.101
node2=192.168.1.101
node3=192.168.1.102
[custom]
mysql_user=demo
mysql_user_host=%
mysql_user_grant=1
mysql_database=demo
mysql_user_password=123456
mysql_root_password="demopass"
transfer_limit=6000
interface=eth0 # 请务必改成服务器本地网卡名，否则会一直retry check！！
memory=512  #512M
my_port=3310
mount_dir=/data/mysql_data
```
