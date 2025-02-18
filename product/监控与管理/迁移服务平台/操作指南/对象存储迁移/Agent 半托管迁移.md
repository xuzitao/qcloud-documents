## 操作场景
Agent 半托管迁移模式中，客户需要手工在源数据云厂商的服务上部署 Agent，Agent 通过内网拉取源数据，并推送到腾讯云对象存储 COS。
如果源数据厂商到腾讯云 COS 间已经拉通专线，Agent 半托管迁移模式不会产生出流量费用。因此建议已经部署了专线的用户采用此模式进行迁移。本文为您详细介绍如何配置 Agent 半托管迁移任务。

## 准备工作
专线准备确认，Agent 半托管模式如果是通过专线迁移，需要确保友商云侧主机上使用 COS SDK 可经过专线访问 COS，迁移前请与商务经理确认。 

## 迁移限速
MSP 迁移工具提供了限制 QPS（对象存储模式）和带宽限速（URL 列表模式）。在使用 Agent 迁移的情况下，也可以在迁移服务器上进行限速操作，同时在 MSP 中创建任务时选择“不限速”。
1. 执行如下命令，查看网卡。
```bash
[root@VM_10_12_centos ~]# ifconfig
```
2. 通过如下命令，测试限速前的下载速度。
```bash
[root@VM_10_12_centos ~]# wget https://msp-test-src-1200000000.cos.ap-guangzhou.myqcloud.com/bkce_src-5.0.2.tar.gz
```
3. 执行如下命令，安装 iproute 工具（默认 Centos 7.x 已安装，此步可跳过）。
```bash
[root@VM_10_12_centos ~]# yum -y install iproute
```
4. 执行如下命令，将 eth0 网卡限速为50kbit。
```bash
[root@VM_10_12_centos ~]# /sbin/tc qdisc add dev eth0 root tbf rate 50kbit latency 50ms burst 10000
```
>?
>- eth0 为网卡序号，由第1步查看网卡获取。
>- 如果需要限速10M，则将50kbit改为10Mbit。
5. 限速后测试，执行如下命令，验证下载速度是否已被限制。
```bash
[root@VM_10_12_centos ~]# wget https://msp-test-src-1200000000.cos.ap-guangzhou.myqcloud.com/bkce_src-5.0.2.tar.gz
```
6. 若您需要解除限速，则可以执行如下命令，即可解除限速。
```bash
[root@VM_10_12_centos ~]# /sbin/tc qdisc del dev eth0 root tbf
```
 

## 实施迁移
1. 在迁移源一侧建立用于迁移的临时服务器（主控服务器）。
因为建立迁移任务时需要填写 Agent 主控服务器的 IP 地址（内网 IP 地址，用于与迁移集群中的 worker 服务器通信），因此在建立迁移任务之前，需先在友商云上准备一台操作系统为 CentOS 7.x 64位的云服务器。
例如：在友商云上创建一台主控服务器，内网IP地址为 172.19.97.94。
2. 在腾讯云 MSP 中建立迁移任务。
在“选择迁移模式”中的“模式选择”部分，选择“新建迁移任务后手动下载Agent启动迁移” 
在“主节点内网IP”部分，填写友商云上创建的服务器内网IP地址（172.19.97.94）
3. 所有参数填写完毕后，单击【新建并启动】。需要注意的是，在 Agent 模式下任务已创建成功但并未运行，需要按以下步骤在友商主控服务器上手工启动 Agent。
4. 在主控服务器上部署和启动 Agent。
 1. 将 Agent 工具解压到合适的目录（目录无特殊要求）。
 2. 修改配置文件。
```
./agent/conf/agent.toml
# 此处填写腾讯云用于迁移的云 API 密钥对
secret_id = '此处填写腾讯云 API 密钥 AccessKey'
secret_key = '此处填写腾讯云 API 密钥S ecretKey'
```
 3. 启动 Agent。
```
# chmod +x ./agent/bin/agent
# cd agent/bin  //必须在 bin 目录中启动 agent 程序（否则会找不到配置文件）
#./agent
```
Agent 会定时自动从 MSP 平台获取任务的详细配置信息，如果创建多个迁移任务无需重复启动 Agent。
5. 扩充迁移集群（增加 worker 服务器）
Agent 模式支持分布式迁移（多服务器协同），如果希望进一步提高迁移速度，在有可用带宽的情况下可以增加 worker 服务器加入到迁移：
 - worker 服务器必须与 master 服务器互通。
 - 如果使用专线迁移，需要确保 worker 服务器可通过专线直接访问 COS。
 
worker 服务器可以是任意配置，但建议与 master 保持一致。部署和启动 Agent 的方式与 master 服务器完全相同（同样需要修改 agent.toml 中的 secret_id 和 secret_key），因新建任务的时候已经指定了 master 服务器，新加入的 agent 均被作为 worker 节点与 master 服务器通信获得任务。 

Worker 服务器可以随时加入迁移集群，但建议创建任务之前将所有的 worker 服务器与 master 一同创建并配置和启动Agent，以便 master 启动任务时可以更有效的进行分片调度。
