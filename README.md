# Consul介绍

**Consul**是HashiCorp公司推出的开源工具，用于实现分布式系统的服务发现与配置。 与其他分布式服务注册与发现的方案，**Consul**的方案更"一站式"，内置了服务注册与发现框架、分布一致性协议实现、健康检查、Key/Value存储、多数据中心方案，不再需要依赖其他工具。

# CITA-Cloud的问题

都是微服务化之后的一些常规问题。

比如服务发现，如何把本服务的`ip`和`port`告诉其他微服务，如何获取其他微服务的`ip`和`port`。

比如配置，如何做到可以动态修改配置，并自动生效。

比如启动顺序，如何等自己依赖的服务起来之后，再启动本服务，避免代码中出现过多异常处理的代码。

# 解决方案

使用`consul`加`consul-template`可以解决目前遇到的问题。

其实目前的方案只用到这两个工具很小一部分功能，更多功能还有待进一步摸索。

对微服务来说，最简单最轻松的方式是静态的配置文件。因此将微服务的`ip`和`port`以及相关配置项都注册到`consul`，然后通过`consul-template`生成配置文件。

动态配置方面，`consul-template`可以在相关配置发生变更的时候，自动更新配置文件，并自动重启服务，从而实现配置变更自动生效的效果。但是服务因为其他原因异常退出的时候，`consul-template`并不会重启服务。

启动顺序方面，经过测试，如果本微服务的配置文件依赖两个配置项，那么在两个配置项都被注册到`consul`之前，`consul-template`不会生成配置文件，也不会启动相应的微服务。因为微服务间的依赖，大部分也都可以体现为配置项的依赖。因此可以通过这种方式来解决启动顺序的问题。

# 配置项列表

### 全局配置

##### 日志

包括日志等级和日志的`appenders`。每个微服务一个日志配置文件的模板。`global.log4rs.level`和`global.log4rs.appenders`。

##### 共识

`global.consensus.block_delay_number`，共识需要延迟几个块才能确认。

##### 环境变量

微服务的名字用环境变量(`SERVICE_NAME`)。这样就可以只使用一个日志模板了。

节点名字类似`node0`，这个也设置为环境变量(`NODE_NAME`)。这样会在多个节点使用一个`consul`实例时，比如四个节点跑在一台服务器上时，作为`key prefix`来区分不同的节点相同的`key`。作用类似于`rabbitmq`的`vhost`。

### 微服务配置

##### 网络

注册一个`service`，服务名为`node{NODE_INDEX}_network`。服务的具体配置也通过`consul-template`来生成。

`kms`，`storage`，`executor`都类似。不过`kms`服务需要提前手动启动好。

网络微服务需要两个配置文件，网络相关的配置`network-config.toml`和网络需要的私钥`network-key`。这两个文件本来是`config-tool`直接生成的，但是为了能动态修改，再使用`consul-template`倒腾一遍。

##### 共识

除了注册自身的服务，还要注册`global.consensus.block_delay_number`这个全局配置。

自身的配置文件`consensus-config.toml`包含：

```
network_port = 51000
controller_port = 51005
```

这些服务的端口是通过`consul-template`从`consul`获取的之前的服务注册的`service`信息。

##### Controller

注册自身的服务。

自身的配置文件`controller-config.toml`包含：

```
network_port = 51000
consensus_port = 51001
storage_port = 51003
kms_port = 51005
executor_port = 51002
block_delay_number = 6
```

端口获取方式同上。

这里的`block_delay_number`就是前面共识注册的`global.consensus.block_delay_number`。

# 使用说明

1. 使用`config-tool`生成链的节点配置。将生成的节点文件夹(不包括顶层以链名称命名的文件夹)拷贝到`chain_config`目录下。

2. 编译`network`，`consensus`，`controller`，`storage`，`kms`和`executor`等微服务，将可执行文件拷贝到`bin`目录下。

3. `kms`服务需要设置或者输入密码，将要使用的密码写入根目录下的`key_file`文件中。

4. 生成`controller`服务需要的创世块相关的配置文件。

   ```
   $ bin/create_genesis.sh  
   timestamp = 1599022274000
   prevhash = "0x0000000000000000000000000000000000000000000000000000000000000000"
   version = 0
   chain_id = "0x0000000000000000000000000000000000000000000000000000000000000001"
   admin = "0x010928818c840630a60b4fda06848cac541599462f"
   block_interval = 3
   validators = ["0x010928818c840630a60b4fda06848cac541599462f","0x010928818c840630a60b4fda06848cac541599462f",]
   ```

   同时会生成：

   - `admin_kms.db`是预先存放了管理员私钥的`kms`数据库文件，`admin_key_id`为对应的私钥的序号。
   - 每个节点文件中`genesis.toml`和`init_sys_config.toml`两个配置文件。
   - 每个节点文件夹中`kms.db`是预先存放了验证节点私钥的`kms`数据库文件，`validator_key_id`为对应的私钥的序号。

5. 生成`syncthing`的配置文件，并启动相关服务。

   ```shell
   $ bin/create_syncthing_configs.sh
   b2ff58efe3156ac0eec14929c0af8ed666bdd1b84f749685f4017b829b8f76e9
   device_id: HPOVTOR-2GYNZTD-JEEHZ65-SRZNLZF-YQS3DXB-SRQB4HT-7GS5Y3Z-ZTE2HQ2
   da27602b27a592755c466f2069fb0fe501fd9bdefc163fe16e3a38f118ffabe9
   device_id: YPATMTR-766CUA2-ZCDPD24-VIU5VN5-RB34FKU-B4ONKW3-WR2RTIQ-6UPT4QM
   device_ids: HPOVTOR-2GYNZTD-JEEHZ65-SRZNLZF-YQS3DXB-SRQB4HT-7GS5Y3Z-ZTE2HQ2;YPATMTR-766CUA2-ZCDPD24-VIU5VN5-RB34FKU-B4ONKW3-WR2RTIQ-6UPT4QM;  peers: node0:22000;node1:22001;
   chain_name: test-chain
   peers: [Peer { ip: "node0", port: 22000 }, Peer { ip: "node1", port: 22001 }]
   ids: ["HPOVTOR-2GYNZTD-JEEHZ65-SRZNLZF-YQS3DXB-SRQB4HT-7GS5Y3Z-ZTE2HQ2", "YPATMTR-766CUA2-ZCDPD24-VIU5VN5-RB34FKU-B4ONKW3-WR2RTIQ-6UPT4QM"]
   version: 32
   ```
   
6. 在项目根目录下执行如下命令启动节点，参数为节点的序号：

   ```
   ./bin/start_node 0
   ```



