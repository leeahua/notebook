[toc]



## docker 笔记

### 1.linux  centos  install  docker

```
#1.清除本地docker残留文件
sudo yum remove docker \
                  docker-client \
                  docker-client-latest \
                  docker-common \
                  docker-latest \
                  docker-latest-logrotate \
                  docker-logrotate \
                  docker-selinux \
                  docker-engine-selinux \
                  docker-engine
 #2. 执行以下命令安装依赖包
 sudo yum install -y yum-utils
 #3.添加国内官方下载源
 #官方下载源
sudo yum-config-manager --add-repo https://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo
 #阿里云下载源
sudo sed -i 's/download.docker.com/mirrors.aliyun.com\/docker-ce/g' /etc/yum.repos.d/docker-ce.repo
#4. 更新yum软件源缓存，并安装docker-ce
sudo yum install docker-ce docker-ce-cli containerd.io
```







### 2.linux 下开机启动docker服务

```
1.linux 下启动docker服务
➜  docker service docker start
Redirecting to /bin/systemctl start docker.service
2.开机启动docker服务
 systemctl enable docker.service
 1q2w3e4rX?
 1q2w3e4rX?>_
 
 lyh1024X_lyh1024X
 

```

### 3.常用组件docker-compose

```
version: '2'
services:
    redis:
      image: redis:5.0.0
      container_name: redis
      command: redis-server --requirepass root
      ports:
        - "6379:6379"
      volumes:
        - ./data:/data
    mysql:
      image: mysql:5.7.33
      container_name: mysql5.7
      command: --default-authentication-plugin=root
      restart: always
      ports:
        - "3306:3306"
      volumes:
        - ./mysql/data:/var/lib/mysql
        - ./mysql/conf/my.cnf:/etc/my.cnf
        - ./mysql/init:/docker-entrypoint-initdb.d/
      environment:
        MYSQL_ROOT_PASSWORD: root
    adminer:
      image: adminer
      restart: always
      ports:
        - 8080:8080
    mongo:
      image: mongo
      restart: always
      ports:
        - 27017:27017
      environment:
        MONGO_INITDB_ROOT_USERNAME: root
        MONGO_INITDB_ROOT_PASSWORD: root

    mongo-express:
      image: mongo-express
      restart: always
      ports:
        - 8081:8081
      environment:
        ME_CONFIG_MONGODB_ADMINUSERNAME: root
        ME_CONFIG_MONGODB_ADMINPASSWORD: root

```

