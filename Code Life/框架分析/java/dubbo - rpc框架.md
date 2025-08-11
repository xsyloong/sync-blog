# 概述
1. Dubbo 官网 https://cn.dubbo.apache.org/zh-cn/
2. 架构图
	![[Pasted image 20250707225943.png]]
	服务提供者将服务注册到注册中心中，然后服务消费者去注册中心获取服务信息，后续消费者根据提供者信息去 RPC 调用服务
# 快速入门
1. Zookeeper 注册中心安装