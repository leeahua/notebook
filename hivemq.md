###

## hivemq 梳理

## linux 系统连接优化

> 1.设置文件句柄数限制  
>
> ```
> add the following lines to the /etc/security/limits.conf
> #名词解释 
> # hard: 系统设置的最大的限制值
> # soft：限制值 警告性质
> # nofile: 最大打开的的文件数
> # hivemq 用户名
> hivemq  hard    nofile  1000000
> hivemq  soft    nofile  1000000
> root    hard    nofile  1000000
> root    soft    nofile  1000000
> ```
>
> 2.优化TCP设置
>
> ```
> add the following lines to the /etc/sysctl.conf
> # This causes the kernel to actively send RST packets when a service is overloaded.
> # 服务过载的时候内核主动去发送RST数据包 fin FIN_WAIT_2 超时等待ack时间
> #此参数默认值60，TCP保持在FIN_WAIT2状态的时间，超时后直接处于CLOSED，所以降低tcp_fin_timeout有助于减少TIME_WAIT数量。注意：虽然#shutdown(SHUD_WR)也会处于FIN_WAIT2状态，但超时并不起作用
> net.ipv4.tcp_fin_timeout = 30
> 
> # The maximum file handles that can be allocated.
> # 最大文件句柄
> fs.file-max = 5097152
> 
> # Enables fast recycling of waiting sockets.
> #此参数默认值0，打开快速TIME_WAIT socket回收。
> #如果tcp_timestamps开启的话，会缓存每个连接的最新时间戳，如果后续请求时间戳小于缓存的时间戳，即视为无效，相应的包被丢弃。所以如果是在#NAT(Network Address Translation)网络下，就可能出现数据包丢弃的现象，会导致大量的TCP连接建立错误。 
> net.ipv4.tcp_tw_recycle = 1
> 
> # Allows reuse of the waiting sockets for new connections, when it is safe from the viewpoint of the protocol .
> #此参数默认值0，是否重用TIME_WAIT状态的socket用于新的连接。
> #这个选项要比net.ipv4.tcp_tw_recycle安全，从协议的角度看，复用是安全的。复用条件:
> #1）net.ipv4.tcp_timestamps选项必须打开(客户端也必须打开) ；
> #2)  重用TIME_WAIT的条件是收到最后一个包后超过1秒；
> net.ipv4.tcp_tw_reuse = 1
> 
> # The default size of the receive buffers that the sockets use.
> net.core.rmem_default = 524288
> 
> # The default size of the send buffers that the sockets use.
> net.core.wmem_default = 524288
> 
> # The maximum size of the received buffers that the sockets use.
> net.core.rmem_max = 67108864
> 
> # The maximum size of the sent buffers that the sockets use.
> net.core.wmem_max = 67108864
> 
> # The size of the receive buffer for each TCP connection. (min, default, max)
> net.ipv4.tcp_rmem = 4096 87380 16777216
> 
> # The size of the sent buffer for each TCP connection. (min, default, max)
> net.ipv4.tcp_wmem = 4096 65536 16777216
> ```
>
> TCP listener
>
> ```
> <listeners>
>         <!-- Open to the outside world -->
>         <tcp-listener>
>             <port>1883</port>
>             <bind-address>0.0.0.0</bind-address>
>             <connect-overload-protection>
>                 <!-- 2000 C/s == 2C/ms -->
>                 <connect-rate>2000</connect-rate>
>                 <connect-burst-size>4000</connect-burst-size>
>             </connect-overload-protection>
>         </tcp-listener>
> 
>         <!-- Only reachable for clients on the same machine -->
>         <tcp-listener>
>             <port>1884</port>
>             <bind-address>127.0.0.1</bind-address>
>             <name>same-machine-listener</name>
>         </tcp-listener>
> 
>         <!-- Secure connection -->
>         <tls-tcp-listener>
>             <port>8883</port>
>             <bind-address>0.0.0.0</bind-address>
>             <name>my-secure-tcp-listener</name>
>             <tls>
>                 <keystore>
>                     <!-- Configuring the path to the key store -->
>                     <path>/path/to/the/key/store.jks</path>
>                     <!-- The password of the key store -->
>                     <password>password-keystore</password>
>                     <!-- The password of the private key -->
>                     <private-key-password>password-key</private-key-password>
>                 </keystore>
>                 <!-- Allow a maximum of 500 concurrent SSL handshakes for this listener -->
>                 <concurrent-handshake-limit>500</concurrent-handshake-limit>
>             </tls>
>         </tls-tcp-listener>
> 
>         <!-- Basic WebSocket connection -->
>         <websocket-listener>
>             <port>8000</port>
>             <bind-address>0.0.0.0</bind-address>
>             <path>/mqtt</path>
>             <name>my-websocket-listener</name>
>             <subprotocols>
>                 <subprotocol>mqttv3.1</subprotocol>
>                 <subprotocol>mqtt</subprotocol>
>             </subprotocols>
>             <allow-extensions>true</allow-extensions>
>         </websocket-listener>
> 
>         <!-- Secure WebSocket connection -->
>         <tls-websocket-listener>
>             <port>8000</port>
>             <bind-address>0.0.0.0</bind-address>
>             <path>/mqtt</path>
>             <subprotocols>
>                 <subprotocol>mqttv3.1</subprotocol>
>                 <subprotocol>mqtt</subprotocol>
>             </subprotocols>
>             <allow-extensions>true</allow-extensions>
>             <tls>
>                 <keystore>
>                     <path>/path/to/the/key/store.jks</path>
>                     <password>password-keystore</password>
>                     <private-key-password>password-key</private-key-password>
>                 </keystore>
>                 <client-authentication-mode>NONE</client-authentication-mode>
>             </tls>
>         </tls-websocket-listener>
> 
>     </listeners>
> ```
>
> listener 属性描述
>
> ```
> 物业名称	默认	必需的	描述
> protocols
> 
> TLSv1.2
> 
> TLSv1.1
> 
> TLSv1
> 
> 不
> 
> 启用的协议。
> 
> 注意：默认情况下未启用TLS 1.3，并且必须在此属性中显式启用它。
> 
> cipher-suites
> 
> TLS_ECDHE_ECDSA_WITH_AES_256_GCM_SHA384
> 
> TLS_ECDHE_ECDSA_WITH_AES_128_GCM_SHA256
> 
> TLS_ECDHE_RSA_WITH_AES_128_GCM_SHA256
> 
> TLS_ECDHE_RSA_WITH_AES_128_CBC_SHA
> 
> TLS_ECDHE_RSA_WITH_AES_256_CBC_SHA
> 
> TLS_RSA_WITH_AES_128_GCM_SHA256
> 
> TLS_RSA_WITH_AES_128_CBC_SHA
> 
> TLS_RSA_WITH_AES_256_CBC_SHA
> 
> 不
> 
> 启用的密码套件。
> 如果需要，定义特定的密码套件以限制启用的套件的数量。
> 注意：可用的密码套件取决于所使用的SSL实施，不一定对所有计算机都相同。
> 
> client-authentication-mode
> 
> 不
> 
> HiveMQ验证客户端证书的方式。提供以下选项：
> 
> NONE：未处理任何客户端证书
> 
> OPTIONAL：如果存在，则处理客户端证书
> 
> REQUIRED：必须存在客户证书
> 
> handshake-timeout
> 
> 10000
> 
> 不
> 
> 暂停操作之前必须成功建立SSL连接的毫秒数
> 
> keystore.path
> 
> 是的
> 
> 证书和私钥所在的密钥库的路径
> 
> keystore.password
> 
> 是的
> 
> 打开密钥库的密码
> 
> keystore.private-key-password
> 
> 不
> 
> 私钥的密码（如果适用）
> 
> truststore.path
> 
> 不
> 
> 包含受信任的客户端证书的信任库的路径
> 
> truststore.password
> 
> 不
> 
> 打开信任库的密码
> 
> concurrent-handshake-limit
> 
> -1
> 
> 不
> 
> 随时可以进行的SSL握手的最大数量。设置为-1时，HiveMQ一次处理一个SSL握手（不允许并发握手）。要允许并发SLL握手，请将属性设置为指示所需最大值的非零正整数。
> 
> native-ssl
> 
> false
> 
> 不
> 
> 定义HiveMQ是否将BoringSSL用于监听器的TLS实现而不是JDK的TLS实现。
> 注意：BoringSSL与某些操作系统不兼容。
> ```
>
> 

​      