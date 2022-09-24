部署规划
10.0.0.99 master
10.0.0.44 node1
10.0.0.74 node2
10.0.0.121 node3 
10.0.0.72 node4 #data set
10.0.0.124 node5 #NFS
10.0.0.118 node6 #front&back
#k8s集群为master+node1-3
#k8s版本为1.16.2
#docker版本为19.03.12
#calico版本为v3.9.0
#k8s pod网段为192.168.0.0/16
#k8s service网段为172.16.0.0/16

#部署harbor，我们选择在node5上部署
#所有节点都需要安装docker
curl -fsSL http://mirrors.aliyun.com/docker-ce/linux/ubuntu/gpg | sudo apt-key add -

add-apt-repository "deb [arch=amd64] http://mirrors.aliyun.com/docker-ce/linux/ubuntu $(lsb_release -cs) stable"

apt -y install docker-ce=5:19.03.12~3-0~ubuntu-bionic

#编辑docker配置文件
nano /etc/docker/daemon.json
#添加
{ "insecure-registries": ["https://harbor.wsco.com"] }

#启动docker
systemctl enable docker --now
systemctl daemon-reload
systemctl restart docker 

#安装 docker-compose#
curl -L https://get.daocloud.io/docker/compose/releases/download/1.22.0/docker-compose-`uname -s`-`uname -m` >/usr/local/bin/docker-compose
chmod +x /usr/local/bin/docker-compose

#安装 python 文件
apt-get -y install python2.7 python-pip

pip install --upgrade pip

#安装 harbor
wget https://storage.googleapis.com/harbor-releases/harbor-offline-installer-v1.5.0.tgz
tar -zxvf harbor-offline-installer-v1.5.0.tgz
mv harbor /usr/local/

#进入 harbor 目录，修改 harbor 配置文件
cd /usr/local/harbor
vi harbor.cfg
1) 将 hostname 修改为域名
2) 将 ui_url_protocol 设置为 ui_url_protocol=https
3) 创建文件夹用来存放证书 mkdir -p /data/cert
4) 在、etc/hosts文件中添加地址解析     10.0.0.124 harbor.wsco.com
5) 请修改/usr/local/harbor/prepare文件第490行：empty_subj = "/C=/ST=/L=/O=/CN=/"请将其修改为empty_subj = "/"
#进入 cert 目录生成证书.
cd /data/cert 
openssl req -x509 -nodes -newkey rsa:2048 -keyout server.key -out server.crt -subj "/CN=*harbor.duhao.com" -days 36500
#安装docker
/usr/local/harbor/install.sh
#使用docker
#默认登录名为admin，初始密码为Harbor12345
#要在客户机浏览器上登录Harbor， 请先配置Harbor域名解析，或者修改hosts文件进行本地解析:
#Windows系统: C:\Windows\System32\drivers\etc\hosts  
#Linux系统: /etc/hosts
echo 10.0.0.124 harbor.wsco.com >>  /etc/hosts
docker login -uadmin -pHarbor12345  https://harbor.wsco.com

#在其他节点上安装docker

#在其他所有节点上配置harbor路径
vim /etc/docker/daemon.json
#添加
{ "insecure-registries": ["https://harbor.wsco.com"] }
#重启docker
systemctl daemon-reload
systemctl restart docker
echo 10.0.0.124 harbor.wsco.com >>  /etc/hosts
docker login -uadmin -pHarbor12345  https://harbor.wsco.com

######查看harbor状态
docker ps -a | grep harbor   
###反馈如下则启动正常
vmware/harbor-jobservice:v1.5.0        "/harbor/start.sh"       4 weeks ago         Up 11 days                                                 harbor-jobservice
vmware/harbor-ui:v1.5.0                "/harbor/start.sh"       4 weeks ago         Up 2 weeks (healthy)                                       harbor-ui
vmware/harbor-db:v1.5.0                "/usr/local/bin/dock…"   4 weeks ago         Up 2 weeks (healthy)   3306/tcp                            harbor-db
vmware/harbor-adminserver:v1.5.0       "/harbor/start.sh"       4 weeks ago         Up 2 weeks (healthy)                                       harbor-adminserver
vmware/harbor-log:v1.5.0               "/bin/sh -c /usr/loc…"   4 weeks ago         Up 2 weeks (healthy)   127.0.0.1:1514->10514/tcp           harbor-log

####通过web界面删除镜像只是软删除，虽然界面查询不到，但是实际上还保留在物理磁盘,此时需要到Harbor服务器执行以下命令
docker run -it --name gc --rm --volumes-from registry vmware/registry:2.6.2-photon garbage-collect  /etc/registry/config.yml



#部署minio，我们选择在node5上部署
#通过 docker 部署 MinIO
docker run -p 9000:9090 --name minio \
    -d --restart=always \
    -e "MINIO_ACCESS_KEY=admin" \
    -e "MINIO_SECRET_KEY=123@abc.com" \
    -v /nfs:/data \
    -v /nfs/config:/root/.minio \
    minio/minio:RELEASE.2020-04-28T23-56-56Z server /data --address ":9090"
#通过浏览器访问minio
输入用户名、密码，点击登录 minio
创建 Bucket
点击 Create Bucket
输入 Bucket Name   dubhe-prod点击 Enter
点击菜单栏，选择 Edit Policy
输入 “ * ” ,选择 Read and Write，点击 添加



####查看minio容器状态
docker ps -a | grep minio 
###查看日志
docker logs [minio容器id或名称]