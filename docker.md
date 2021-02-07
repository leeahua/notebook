## docker 笔记

### 1.linux 下开机启动docker服务

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

### 常用组件docker-compose

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

