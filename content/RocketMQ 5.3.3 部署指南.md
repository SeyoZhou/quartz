#RocketMQ

> 适用于阿里云 Ubuntu 24.04 双服务器部署（测试 + 正式） + macOS 本地开发环境
> 支持 gRPC 客户端 (rocketmq-client-java 5.x)

---

## 架构说明

```
┌─────────────────────────────────────────────────────────────────────────┐
│                           同区域 VPC 内网                                 │
│                                                                         │
│  ┌────────────────────────────┐      ┌────────────────────────────┐    │
│  │   测试服务器 (TEST)          │      │   正式服务器 (PROD)          │    │
│  │                            │      │                            │    │
│  │  NameServer :9876          │      │  NameServer :9876          │    │
│  │  Broker     :10911/10909   │      │  Broker     :10911/10909   │    │
│  │  Proxy      :8081 (gRPC)   │      │  Proxy      :8081 (gRPC)   │    │
│  │  Dashboard  :8082 ─────────┼──────┼─→ 内网连接                   │    │
│  │                            │      │                            │    │
│  │  brokerIP1 = 公网IP         │      │  brokerIP1 = 内网IP         │    │
│  │  外网可访问                 │      │  仅内网访问                  │    │
│  └────────────────────────────┘      └────────────────────────────┘    │
│              ↑                                                          │
└──────────────┼──────────────────────────────────────────────────────────┘
               │ 外网
             开发者 (gRPC 客户端连接 Proxy:8081)
```

### 组件说明

| 组件 | 端口 | 协议 | 说明 |
|------|------|------|------|
| NameServer | 9876 | Remoting | 路由发现服务 |
| Broker | 10911 | Remoting | 消息存储与转发 |
| Broker VIP | 10909 | Remoting | VIP 通道 (可禁用) |
| **Proxy** | **8081** | **gRPC** | **gRPC 客户端接入层 (rocketmq-client-java)** |
| Dashboard | 8082 | HTTP | Web 管理控制台 |

> **重要**: `rocketmq-client-java 5.x` 使用 gRPC 协议，必须连接 Proxy:8081，不能直连 NameServer 或 Broker。

---

## 占位符说明

| 占位符 | 说明 |
|--------|------|
| `<TEST_PUBLIC_IP>` | 测试服务器公网 IP |
| `<TEST_PRIVATE_IP>` | 测试服务器内网 IP |
| `<PROD_PRIVATE_IP>` | 正式服务器内网 IP |

---

# Part 1: 测试环境部署

> 包含 NameServer + Broker + Proxy + Dashboard，Dashboard 同时管理测试和正式两个集群

## 1.1 系统准备

```bash
sudo apt update && sudo apt upgrade -y
sudo apt install -y wget unzip vim openjdk-17-jdk-headless maven git

java -version
```

---

## 1.2 安装 RocketMQ

```bash
sudo mkdir -p /opt/rocketmq
sudo mkdir -p /data/rocketmq/{store,logs}

cd /tmp
wget https://dist.apache.org/repos/dist/release/rocketmq/5.3.3/rocketmq-all-5.3.3-bin-release.zip
# 备用: wget https://archive.apache.org/dist/rocketmq/5.3.3/rocketmq-all-5.3.3-bin-release.zip

sudo unzip rocketmq-all-5.3.3-bin-release.zip -d /opt/
sudo mv /opt/rocketmq-all-5.3.3-bin-release/* /opt/rocketmq/
sudo rm -rf /opt/rocketmq-all-5.3.3-bin-release

sudo useradd -r -s /sbin/nologin rocketmq
sudo mkdir -p /home/rocketmq/logs/dashboardlogs
sudo chown -R rocketmq:rocketmq /opt/rocketmq /data/rocketmq /home/rocketmq
```

---

## 1.3 JVM 内存配置

> 以下为 4C8G 推荐配置，2C4G 小规格参考注释

**NameServer & Proxy:**
```bash
sudo vim /opt/rocketmq/bin/runserver.sh
```
```bash
# 4C8G: 1g / 2C4G: 256m
JAVA_OPT="${JAVA_OPT} -server -Xms1g -Xmx1g -Xmn512m"
```

**Broker:**
```bash
sudo vim /opt/rocketmq/bin/runbroker.sh
```
```bash
# 4C8G: 4g / 2C4G: 768m
JAVA_OPT="${JAVA_OPT} -server -Xms4g -Xmx4g -Xmn2g"
```

---

## 1.4 Broker 配置

```bash
sudo vim /opt/rocketmq/conf/broker-test.conf
```

```properties
# 集群配置
brokerClusterName = TestCluster
brokerName = broker-test
brokerId = 0

# 网络配置
namesrvAddr = 127.0.0.1:9876
brokerIP1 = <TEST_PUBLIC_IP>
listenPort = 10911

# 存储配置
storePathRootDir = /data/rocketmq/store
storePathCommitLog = /data/rocketmq/store/commitlog
fileReservedTime = 48
deleteWhen = 04

# 性能配置
brokerRole = ASYNC_MASTER
flushDiskType = ASYNC_FLUSH

# 功能开关
autoCreateTopicEnable = true
autoCreateSubscriptionGroup = true

# ACL 认证
aclEnable = true
```

---

## 1.5 Proxy 配置

```bash
sudo vim /opt/rocketmq/conf/rmq-proxy.json
```

```json
{
  "rocketMQClusterName": "TestCluster",
  "namesrvAddr": "127.0.0.1:9876",
  "grpcServerPort": 8081,
  "remotingListenPort": 8080
}
```

---

## 1.6 ACL 认证配置

```bash
sudo vim /opt/rocketmq/conf/plain_acl.yml
```

```yaml
# =============================================================================
# RocketMQ ACL 配置文件
# =============================================================================
# 文件位置: $ROCKETMQ_HOME/conf/plain_acl.yml
# 生效条件: broker.conf 中 aclEnable=true
# 热加载:   修改后无需重启 Broker，约 10 秒自动生效
# =============================================================================

# 全局白名单 IP（这些 IP 无需认证）
# 生产环境建议留空，所有访问都需认证
globalWhiteRemoteAddresses:
  # - 10.10.10.*      # 允许 10.10.10.x 网段免认证
  # - 192.168.0.1     # 允许指定 IP 免认证

accounts:
  # -------------------------------------------------------------------------
  # 管理员账号 (拥有所有权限)
  # -------------------------------------------------------------------------
  - accessKey: admin_ak_test              # 访问密钥 (类似用户名)
    secretKey: admin_sk_test_moz_ios_1234567890   # 密钥 (类似密码，建议32位以上)
    admin: true                           # true=管理员，拥有所有 Topic/Group 权限

  # -------------------------------------------------------------------------
  # 应用账号 (按需授权)
  # -------------------------------------------------------------------------
  - accessKey: app_ak_test                # 应用访问密钥
    secretKey: admin_sk_test_moz_ios_1234567890      # 应用密钥
    admin: false                          # 非管理员，需要显式授权

    # 默认权限 (未在 topicPerms/groupPerms 中列出的资源使用此权限)
    # 可选值: PUB (发送), SUB (订阅), PUB|SUB (发送+订阅), DENY (拒绝)
    defaultTopicPerm: PUB|SUB              # 默认拒绝访问未授权 Topic
    defaultGroupPerm: SUB                # 默认拒绝访问未授权 ConsumerGroup

    # Topic 级别权限
    # 格式: TopicName=权限
    topicPerms:
      - test-topic=PUB|SUB                # 允许发送和订阅
      - test-fifo-topic=PUB|SUB
      - test-delay-topic=PUB|SUB
      - readonly-topic=SUB                # 仅允许订阅，不允许发送

    # ConsumerGroup 级别权限
    # 格式: GroupName=权限 (通常只需 SUB)
    groupPerms:
      - test-consumer-group=SUB           # 允许该消费组订阅
      - GID_order_consumer=SUB
```

### ACL 配置说明

| 字段 | 说明 | 示例 |
|------|------|------|
| `accessKey` | 访问密钥，客户端连接时使用 | `app_ak_test` |
| `secretKey` | 密钥，建议 32 位以上随机字符串 | `xK9#mP2$vL5@nQ8` |
| `admin` | 是否管理员，`true` 拥有所有权限 | `true` / `false` |
| `defaultTopicPerm` | 未授权 Topic 的默认权限 | `DENY` / `PUB` / `SUB` / `PUB\|SUB` |
| `defaultGroupPerm` | 未授权 Group 的默认权限 | `DENY` / `SUB` |
| `topicPerms` | Topic 级别权限列表 | `order-topic=PUB\|SUB` |
| `groupPerms` | ConsumerGroup 级别权限列表 | `GID_order=SUB` |

### 权限值说明

| 权限值 | 说明 |
|--------|------|
| `PUB` | 允许发送消息 (Producer) |
| `SUB` | 允许订阅消息 (Consumer) |
| `PUB\|SUB` | 允许发送和订阅 |
| `DENY` | 拒绝访问 |

---

## 1.7 Systemd 服务配置

### NameServer

```bash
sudo vim /etc/systemd/system/rocketmq-namesrv.service
```

```ini
[Unit]
Description=RocketMQ NameServer
After=network.target

[Service]
Type=simple
User=rocketmq
Group=rocketmq
Environment="JAVA_HOME=/usr/lib/jvm/java-17-openjdk-amd64"
ExecStart=/opt/rocketmq/bin/mqnamesrv
ExecStop=/opt/rocketmq/bin/mqshutdown namesrv
Restart=on-failure
RestartSec=5
LimitNOFILE=65536

[Install]
WantedBy=multi-user.target
```

### Broker

```bash
sudo vim /etc/systemd/system/rocketmq-broker.service
```

```ini
[Unit]
Description=RocketMQ Broker
After=network.target rocketmq-namesrv.service
Requires=rocketmq-namesrv.service

[Service]
Type=simple
User=rocketmq
Group=rocketmq
Environment="JAVA_HOME=/usr/lib/jvm/java-17-openjdk-amd64"
ExecStart=/opt/rocketmq/bin/mqbroker -c /opt/rocketmq/conf/broker-test.conf
ExecStop=/opt/rocketmq/bin/mqshutdown broker
Restart=on-failure
RestartSec=5
LimitNOFILE=65536

[Install]
WantedBy=multi-user.target
```

### Proxy

```bash
sudo vim /etc/systemd/system/rocketmq-proxy.service
```

```ini
[Unit]
Description=RocketMQ Proxy (gRPC Gateway)
After=network.target rocketmq-broker.service
Requires=rocketmq-broker.service

[Service]
Type=simple
User=rocketmq
Group=rocketmq
Environment="JAVA_HOME=/usr/lib/jvm/java-17-openjdk-amd64"
ExecStart=/opt/rocketmq/bin/mqproxy -pc /opt/rocketmq/conf/rmq-proxy.json
Restart=on-failure
RestartSec=5
LimitNOFILE=65536

[Install]
WantedBy=multi-user.target
```

### Dashboard

```bash
# 编译 Dashboard
cd /tmp
git clone https://github.com/apache/rocketmq-dashboard.git
cd rocketmq-dashboard
mvn clean package -DskipTests

sudo mkdir -p /opt/rocketmq-dashboard
sudo cp target/rocketmq-dashboard-*.jar /opt/rocketmq-dashboard/rocketmq-dashboard.jar
sudo chown -R rocketmq:rocketmq /opt/rocketmq-dashboard
```

```bash
sudo vim /etc/systemd/system/rocketmq-dashboard.service
```

```ini
[Unit]
Description=RocketMQ Dashboard
After=network.target rocketmq-broker.service

[Service]
Type=simple
User=rocketmq
Group=rocketmq
Environment="JAVA_HOME=/usr/lib/jvm/java-17-openjdk-amd64"
ExecStart=/usr/bin/java \
    -Xms384m -Xmx384m \
    -Drocketmq.namesrv.addr=127.0.0.1:9876;<PROD_PRIVATE_IP>:9876 \
    -Drocketmq.client.vipChannelEnabled=false \
    -jar /opt/rocketmq-dashboard/rocketmq-dashboard.jar \
    --server.port=8082
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
```

---

## 1.8 启动服务

```bash
sudo systemctl daemon-reload
sudo systemctl enable rocketmq-namesrv rocketmq-broker rocketmq-proxy rocketmq-dashboard
sudo systemctl start rocketmq-namesrv
sudo systemctl start rocketmq-broker
sudo systemctl start rocketmq-proxy
sudo systemctl start rocketmq-dashboard

# 验证
sudo systemctl status rocketmq-namesrv rocketmq-broker rocketmq-proxy rocketmq-dashboard
sudo netstat -tlnp | grep -E "(9876|10911|8081|8082)"
```

---

## 1.9 防火墙与安全组

### Ubuntu 防火墙

```bash
sudo ufw enable
sudo ufw allow 22/tcp      # SSH
sudo ufw allow 9876/tcp    # NameServer
sudo ufw allow 10911/tcp   # Broker
sudo ufw allow 10909/tcp   # Broker VIP
sudo ufw allow 8081/tcp    # Proxy gRPC
sudo ufw allow 8082/tcp    # Dashboard
sudo ufw status
```

### 阿里云安全组

| 协议 | 端口 | 授权对象 | 说明 |
|------|------|----------|------|
| TCP | 22 | 0.0.0.0/0 | SSH |
| TCP | 9876 | 0.0.0.0/0 | NameServer |
| TCP | 10911 | 0.0.0.0/0 | Broker |
| TCP | 10909 | 0.0.0.0/0 | Broker VIP |
| TCP | 8081 | 0.0.0.0/0 | Proxy gRPC |
| TCP | 8082 | 0.0.0.0/0 | Dashboard |

---

## 1.10 验证

```bash
export NAMESRV_ADDR=127.0.0.1:9876
cd /opt/rocketmq

# 集群状态
sh bin/mqadmin clusterList -n $NAMESRV_ADDR

# 发送测试消息
sh bin/tools.sh org.apache.rocketmq.example.quickstart.Producer
sh bin/tools.sh org.apache.rocketmq.example.quickstart.Consumer
```

Dashboard: `http://<TEST_PUBLIC_IP>:8082`

Java 客户端连接: `<TEST_PUBLIC_IP>:8081` (gRPC)

---

# Part 2: 正式环境部署

> 仅 NameServer + Broker + Proxy，通过测试服务器 Dashboard 统一管理

## 2.1 系统准备

```bash
sudo apt update && sudo apt upgrade -y
sudo apt install -y wget unzip vim openjdk-17-jdk-headless

java -version
```

---

## 2.2 安装 RocketMQ

```bash
sudo mkdir -p /opt/rocketmq
sudo mkdir -p /data/rocketmq/{store,logs}

cd /tmp
wget https://dist.apache.org/repos/dist/release/rocketmq/5.3.3/rocketmq-all-5.3.3-bin-release.zip

sudo unzip rocketmq-all-5.3.3-bin-release.zip -d /opt/
sudo mv /opt/rocketmq-all-5.3.3-bin-release/* /opt/rocketmq/
sudo rm -rf /opt/rocketmq-all-5.3.3-bin-release

sudo useradd -r -s /sbin/nologin rocketmq
sudo chown -R rocketmq:rocketmq /opt/rocketmq /data/rocketmq
```

---

## 2.3 JVM 内存配置

同测试环境，参考 Part 1 第 1.3 节。

---

## 2.4 Broker 配置

```bash
sudo vim /opt/rocketmq/conf/broker-prod.conf
```

```properties
# 集群配置
brokerClusterName = ProdCluster
brokerName = broker-prod
brokerId = 0

# 网络配置 (仅内网)
namesrvAddr = 127.0.0.1:9876
brokerIP1 = <PROD_PRIVATE_IP>
listenPort = 10911

# 存储配置
storePathRootDir = /data/rocketmq/store
storePathCommitLog = /data/rocketmq/store/commitlog
fileReservedTime = 168
deleteWhen = 04

# 性能配置
brokerRole = ASYNC_MASTER
flushDiskType = ASYNC_FLUSH

# 功能开关 (生产环境禁止自动创建)
autoCreateTopicEnable = false
autoCreateSubscriptionGroup = false

# ACL 认证
aclEnable = true
```

---

## 2.5 Proxy 配置

```bash
sudo vim /opt/rocketmq/conf/rmq-proxy.json
```

```json
{
  "rocketMQClusterName": "ProdCluster",
  "namesrvAddr": "127.0.0.1:9876",
  "grpcServerPort": 8081,
  "remotingListenPort": 8080
}
```

---

## 2.6 ACL 认证配置

> 配置说明参考 Part 1 第 1.6 节

```bash
sudo vim /opt/rocketmq/conf/plain_acl.yml
```

```yaml
globalWhiteRemoteAddresses:

accounts:
  # 管理员账号
  - accessKey: admin_ak_prod
    secretKey: admin_sk_prod_moz_ios_1234567890    # 生产环境请使用强密码
    admin: true

  # 应用账号 (生产环境建议 defaultTopicPerm/defaultGroupPerm 设为 DENY)
  - accessKey: app_ak_prod
    secretKey: app_sk_prod_moz_ios_1234567890
    admin: false
    defaultTopicPerm: DENY                # 生产环境默认拒绝
    defaultGroupPerm: DENY
    topicPerms:
      - order-topic=PUB|SUB
      - payment-topic=PUB|SUB
    groupPerms:
      - order-consumer-group=SUB
      - payment-consumer-group=SUB
```

---

## 2.7 Systemd 服务配置

### NameServer

```bash
sudo vim /etc/systemd/system/rocketmq-namesrv.service
```

```ini
[Unit]
Description=RocketMQ NameServer (Production)
After=network.target

[Service]
Type=simple
User=rocketmq
Group=rocketmq
Environment="JAVA_HOME=/usr/lib/jvm/java-17-openjdk-amd64"
ExecStart=/opt/rocketmq/bin/mqnamesrv
ExecStop=/opt/rocketmq/bin/mqshutdown namesrv
Restart=on-failure
RestartSec=5
LimitNOFILE=65536

[Install]
WantedBy=multi-user.target
```

### Broker

```bash
sudo vim /etc/systemd/system/rocketmq-broker.service
```

```ini
[Unit]
Description=RocketMQ Broker (Production)
After=network.target rocketmq-namesrv.service
Requires=rocketmq-namesrv.service

[Service]
Type=simple
User=rocketmq
Group=rocketmq
Environment="JAVA_HOME=/usr/lib/jvm/java-17-openjdk-amd64"
ExecStart=/opt/rocketmq/bin/mqbroker -c /opt/rocketmq/conf/broker-prod.conf
ExecStop=/opt/rocketmq/bin/mqshutdown broker
Restart=on-failure
RestartSec=5
LimitNOFILE=65536

[Install]
WantedBy=multi-user.target
```

### Proxy

```bash
sudo vim /etc/systemd/system/rocketmq-proxy.service
```

```ini
[Unit]
Description=RocketMQ Proxy (Production)
After=network.target rocketmq-broker.service
Requires=rocketmq-broker.service

[Service]
Type=simple
User=rocketmq
Group=rocketmq
Environment="JAVA_HOME=/usr/lib/jvm/java-17-openjdk-amd64"
ExecStart=/opt/rocketmq/bin/mqproxy -pc /opt/rocketmq/conf/rmq-proxy.json
Restart=on-failure
RestartSec=5
LimitNOFILE=65536

[Install]
WantedBy=multi-user.target
```

---

## 2.8 启动服务

```bash
sudo systemctl daemon-reload
sudo systemctl enable rocketmq-namesrv rocketmq-broker rocketmq-proxy
sudo systemctl start rocketmq-namesrv
sudo systemctl start rocketmq-broker
sudo systemctl start rocketmq-proxy

sudo systemctl status rocketmq-namesrv rocketmq-broker rocketmq-proxy
sudo netstat -tlnp | grep -E "(9876|10911|8081)"
```

---

## 2.9 安全组配置 (仅 VPC 内网)

| 协议 | 端口 | 授权对象 | 说明 |
|------|------|----------|------|
| TCP | 22 | 跳板机IP | SSH |
| TCP | 9876 | VPC网段 | NameServer |
| TCP | 10911 | VPC网段 | Broker |
| TCP | 10909 | VPC网段 | Broker VIP |
| TCP | 8081 | VPC网段 | Proxy gRPC |

> 不开放 8082：正式环境不部署 Dashboard

---

## 2.10 验证

```bash
export NAMESRV_ADDR=127.0.0.1:9876
sh /opt/rocketmq/bin/mqadmin clusterList -n $NAMESRV_ADDR
```

通过测试服务器 Dashboard 查看 ProdCluster。

Java 客户端连接: `<PROD_PRIVATE_IP>:8081` (gRPC, 仅 VPC 内网可访问)

---

# Part 3: macOS 本地开发环境

## 3.1 目录结构

```
~/software/rocketmq/
├── rocketmq-all-5.3.3-bin-release/
│   ├── bin/
│   ├── conf/
│   │   ├── broker-local.conf
│   │   ├── rmq-proxy.json
│   │   └── plain_acl.yml
│   └── ...
├── dashboard/
│   ├── rocketmq-dashboard-2.0.0/
│   │   └── target/rocketmq-dashboard-2.0.0.jar
│   └── rocketmq-dashboard.jar
├── data/
│   ├── store/
│   └── logs/
└── scripts/
    ├── start-all.sh
    └── stop-all.sh
```

---

## 3.2 环境准备

```bash
# 使用 SDKMAN 切换 Java 17
sdk use java 17-open
java -version
```

---

## 3.3 安装 RocketMQ

```bash
mkdir -p ~/software/rocketmq
mkdir -p ~/software/rocketmq/data/{store,logs}
mkdir -p ~/software/rocketmq/scripts
mkdir -p ~/software/rocketmq/dashboard

cd /tmp
curl -O https://dist.apache.org/repos/dist/release/rocketmq/5.3.3/rocketmq-all-5.3.3-bin-release.zip
unzip rocketmq-all-5.3.3-bin-release.zip -d ~/software/rocketmq/
```

---

## 3.4 安装 Dashboard

> 需要 Maven (mvn -v) 用于本地打包

```bash
cd ~/software/rocketmq/dashboard
curl -L -o rocketmq-dashboard-2.0.0.zip https://github.com/apache/rocketmq-dashboard/archive/refs/tags/rocketmq-dashboard-2.0.0.zip
unzip rocketmq-dashboard-2.0.0.zip
cd rocketmq-dashboard-rocketmq-dashboard-2.0.0
mvn -DskipTests clean package
ln -sf "$PWD/target/rocketmq-dashboard-2.0.0.jar" ~/software/rocketmq/dashboard/rocketmq-dashboard.jar
```

---

## 3.5 JVM 内存配置

**NameServer & Proxy (512M):**
```bash
vim ~/software/rocketmq/rocketmq-all-5.3.3-bin-release/bin/runserver.sh
```
```bash
JAVA_OPT="${JAVA_OPT} -server -Xms512m -Xmx512m -Xmn256m"
```

**Broker (1G):**
```bash
vim ~/software/rocketmq/rocketmq-all-5.3.3-bin-release/bin/runbroker.sh
```
```bash
JAVA_OPT="${JAVA_OPT} -server -Xms1g -Xmx1g -Xmn512m"
```

---

## 3.6 Broker 配置

```bash
vim ~/software/rocketmq/rocketmq-all-5.3.3-bin-release/conf/broker-local.conf
```

```properties
# 集群配置
brokerClusterName = LocalCluster
brokerName = broker-local
brokerId = 0

# 网络配置
namesrvAddr = 127.0.0.1:9876
brokerIP1 = 127.0.0.1
listenPort = 10911

# 存储路径 (绝对路径)
storePathRootDir = /Users/aoci/software/rocketmq/data/store
storePathCommitLog = /Users/aoci/software/rocketmq/data/store/commitlog

# 本地开发配置
fileReservedTime = 48
deleteWhen = 04
brokerRole = ASYNC_MASTER
flushDiskType = ASYNC_FLUSH

# 允许自动创建
autoCreateTopicEnable = true
autoCreateSubscriptionGroup = true

# ACL 认证
aclEnable = true
```

---

## 3.7 Proxy 配置

```bash
vim ~/software/rocketmq/rocketmq-all-5.3.3-bin-release/conf/rmq-proxy.json
```

```json
{
  "rocketMQClusterName": "LocalCluster",
  "namesrvAddr": "127.0.0.1:9876",
  "grpcServerPort": 8081
}
```

---

## 3.8 ACL 认证配置

> 配置说明参考 Part 1 第 1.6 节

```bash
vim ~/software/rocketmq/rocketmq-all-5.3.3-bin-release/conf/plain_acl.yml
```

```yaml
globalWhiteRemoteAddresses:

accounts:
  # 本地开发账号 (admin=true 简化开发，无需逐个授权 Topic)
  - accessKey: local_ak
    secretKey: local_sk
    admin: true                           # 本地开发使用管理员权限
```

---

## 3.9 启动脚本

```bash
vim ~/software/rocketmq/scripts/start-all.sh
```

```bash
#!/bin/bash

source "$HOME/.sdkman/bin/sdkman-init.sh"
sdk use java 17-open

ROCKETMQ_HOME=~/software/rocketmq/rocketmq-all-5.3.3-bin-release
LOG_DIR=~/software/rocketmq/data/logs
DASHBOARD_JAR=~/software/rocketmq/dashboard/rocketmq-dashboard.jar

echo "Starting NameServer..."
nohup sh $ROCKETMQ_HOME/bin/mqnamesrv > $LOG_DIR/namesrv.log 2>&1 &
sleep 3

echo "Starting Broker..."
nohup sh $ROCKETMQ_HOME/bin/mqbroker -c $ROCKETMQ_HOME/conf/broker-local.conf > $LOG_DIR/broker.log 2>&1 &
sleep 3

echo "Starting Proxy..."
nohup sh $ROCKETMQ_HOME/bin/mqproxy -pc $ROCKETMQ_HOME/conf/rmq-proxy.json > $LOG_DIR/proxy.log 2>&1 &
sleep 2

echo "Starting Dashboard..."
nohup java -jar $DASHBOARD_JAR \
  --server.port=8082 \
  --rocketmq.config.namesrvAddr=127.0.0.1:9876 \
  --rocketmq.config.isVIPChannel=false \
  --rocketmq.config.accessKey=local_ak \
  --rocketmq.config.secretKey=local_sk \
  > $LOG_DIR/dashboard.log 2>&1 &
sleep 2

echo ""
echo "RocketMQ started!"
echo "  NameServer : 127.0.0.1:9876"
echo "  Broker     : 127.0.0.1:10911"
echo "  Proxy gRPC : 127.0.0.1:8081"
echo "  Dashboard  : http://localhost:8082"
```

---

## 3.10 停止脚本

```bash
vim ~/software/rocketmq/scripts/stop-all.sh
```

```bash
#!/bin/bash

ROCKETMQ_HOME=~/software/rocketmq/rocketmq-all-5.3.3-bin-release

echo "Stopping Dashboard..."
pkill -f "rocketmq-dashboard.jar"

echo "Stopping Proxy..."
pkill -f "ProxyStartup"

echo "Stopping Broker..."
sh $ROCKETMQ_HOME/bin/mqshutdown broker

echo "Stopping NameServer..."
sh $ROCKETMQ_HOME/bin/mqshutdown namesrv

echo "RocketMQ stopped!"
```

---

## 3.11 赋予执行权限并启动

```bash
chmod +x ~/software/rocketmq/scripts/*.sh

# 启动
~/software/rocketmq/scripts/start-all.sh

# 停止
~/software/rocketmq/scripts/stop-all.sh
```

---

## 3.12 验证

```bash
export NAMESRV_ADDR=127.0.0.1:9876
cd ~/software/rocketmq/rocketmq-all-5.3.3-bin-release

sh bin/mqadmin clusterList -n $NAMESRV_ADDR

# 查看端口
lsof -i :9876
lsof -i :10911
lsof -i :8081
lsof -i :8082
```

Java 客户端连接: `127.0.0.1:8081` (gRPC)
Dashboard 访问: `http://localhost:8082` (访问域名需统一，`localhost` 或 `127.0.0.1` 二选一)

---

# Part 4: 客户端连接配置

## Java 客户端 (rocketmq-client-java 5.x)

```java
// 连接配置
RocketMqClientConfig config = RocketMqClientConfig.builder()
    .endpoints(List.of("127.0.0.1:8081"))       // Proxy gRPC 端口
    .namespace("optional-namespace")             // 可选
    .akSkAuth("local_ak", "local_sk")           // ACL 认证
    .build();
```

## 各环境连接信息

| 环境 | Endpoints | Access Key | Secret Key |
|------|-----------|------------|------------|
| 本地 | `127.0.0.1:8081` | `local_ak` | `local_sk` |
| 测试 | `<TEST_PUBLIC_IP>:8081` | `app_ak_test` | `app_sk_test_change_me` |
| 生产 | `<PROD_PRIVATE_IP>:8081` | `app_ak_prod` | `app_sk_prod_change_me` |

---

# Part 5: 运维命令

## 服务管理

```bash
# 启动顺序: NameServer → Broker → Proxy → Dashboard
sudo systemctl start rocketmq-namesrv
sudo systemctl start rocketmq-broker
sudo systemctl start rocketmq-proxy
sudo systemctl start rocketmq-dashboard

# 停止顺序: Dashboard → Proxy → Broker → NameServer
sudo systemctl stop rocketmq-dashboard
sudo systemctl stop rocketmq-proxy
sudo systemctl stop rocketmq-broker
sudo systemctl stop rocketmq-namesrv

# 查看状态
sudo systemctl status rocketmq-{namesrv,broker,proxy,dashboard}
```

## 日志查看

```bash
# NameServer
tail -f /opt/rocketmq/logs/rocketmqlogs/namesrv.log

# Broker
tail -f /opt/rocketmq/logs/rocketmqlogs/broker.log

# Proxy
tail -f /opt/rocketmq/logs/rocketmqlogs/proxy.log

# Dashboard
sudo journalctl -u rocketmq-dashboard -f
```

## 集群管理

```bash
export NAMESRV_ADDR=127.0.0.1:9876

# 查看集群
sh /opt/rocketmq/bin/mqadmin clusterList -n $NAMESRV_ADDR

# 查看 Topic
sh /opt/rocketmq/bin/mqadmin topicList -n $NAMESRV_ADDR

# 创建 Topic (生产环境)
sh /opt/rocketmq/bin/mqadmin updateTopic -n $NAMESRV_ADDR -t <TopicName> -c <ClusterName>

# 查看消费进度
sh /opt/rocketmq/bin/mqadmin consumerProgress -n $NAMESRV_ADDR
```

---

# Part 6: 常见问题

### Q1: gRPC 客户端连接失败 "UNAVAILABLE"
确认 Proxy 已启动，客户端连接 8081 端口而非 9876。

### Q2: ACL 认证失败 "No accessKey"
检查 `plain_acl.yml` 配置，确认 accessKey/secretKey 与客户端一致。

### Q3: Dashboard 无法连接集群
在 systemd 配置中添加 `-Drocketmq.client.vipChannelEnabled=false`。

### Q4: Broker 启动失败 "Lock failed"
```bash
sudo rm -rf /data/rocketmq/store/lock
sudo systemctl restart rocketmq-broker
```

### Q5: Proxy 启动失败
检查 `rmq-proxy.json` 中 `namesrvAddr` 和 `rocketMQClusterName` 是否正确。

---

# 端口清单

| 端口 | 服务 | 协议 | 说明 |
|------|------|------|------|
| 9876 | NameServer | Remoting | 路由发现 |
| 10911 | Broker | Remoting | 消息收发 |
| 10909 | Broker VIP | Remoting | VIP 通道 |
| **8081** | **Proxy** | **gRPC** | **gRPC 客户端连接** |
| 8080 | Proxy | Remoting | Remoting 客户端连接 |
| 8082 | Dashboard | HTTP | Web 控制台 |

---

# 配置对比

| 配置项 | 本地 | 测试 | 生产 |
|--------|------|------|------|
| brokerClusterName | LocalCluster | TestCluster | ProdCluster |
| brokerIP1 | 127.0.0.1 | 公网 IP | 内网 IP |
| autoCreateTopicEnable | true | true | false |
| fileReservedTime | 48h | 48h | 168h |
| Dashboard | 无 | 部署 | 无 |
| Proxy | 部署 | 部署 | 部署 |
