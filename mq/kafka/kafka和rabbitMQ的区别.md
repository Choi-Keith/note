# Kafka和RabbitMQ的区别

## 1. 应用场景方面
- RabbitMQ: 用于实时的，对可靠性要求较高的消息传递上
- kafka: 用于活跃的流式数据，大数据量的数据处理上

## 2. 架构模型上
producer, broker, consumer
- RabbitMQ: 以broker为中心，有消息的确认机制（确认机制指客户端消费消息的时候）
- kafka: 以为consumer为中心，无消息的确认机制(确认机制指客户端消费消息的时候)    
RabbitMQ和kafka都有发送到broker的确认机制

## 3. 吞吐量
- RabbitMQ: 支持消息的可靠传递，支持事物，不支持批量操作，基于存储的可靠性要求可以采用内存或硬盘，吞吐量小。如果需要持久化，会采用实时存储
- kafka: 内部采用消息的批量处理，数据的存储和获取式本地磁盘的顺序批量操作，消息处理的效率高，吞吐量高。定时持久化存储，非实时持久化存储

## 4. 集群负载均衡
- RabbitMQ: 本身不支持负载均衡，需要loadbalancer的支持。即指定到哪个broker就是broker
- kafka: 采用zookeeper对集群中的broker， consumer进行管理，可以注册topic到zookeeper上，通过zookeeper的协调机制，producer保存对应的topic的broker信息，可以随机或者轮询发送到broker上，producer可以基于语义指定分片，消息发送到broker的某个分片上。即如果不指定分片，就会默认存到master上的分片上，然后同步到其它的分片。

## 总结
RabbitMQ为了更高的灵活性和信息安全性，放弃了吞吐量；kafka为了更高的吞吐量，选择了速度，放弃了部分安全性