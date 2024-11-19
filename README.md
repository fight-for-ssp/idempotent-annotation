这是一个组件，使用 **AOP+注解** 的形式解决幂等问题，在设计上尽量使用策略模式、模板方法模式、满足开闭原则。可以直接使用 **@idemponent** 注解来实现幂等。

幂等场景 Scene ：幂等出现的场景可能有多种，在注解中通过scene参数来指定幂等场景是RESTPI还是MQ。

验证类型 Type ：可以指定幂等的验证类型，例如Token验证、方法参数验证、SpEL表达式验证。


这是一个还可以持续完善的组件：

    - MQ场景下已经在RocketMQ上调试通过，欢迎各位提交RabbitMQ、Kafka、ActiveMQ等测试。
    
    - 使用去重表完成MQ场景下的幂等，当前仅支持Redis来实现，因为感觉数据库在性能上不占优，但如果有新的场景可以用到数据库来做去重表，欢迎补充。


1、什么是幂等问题?

    - 接口幂等：接口防重复提交
        例如：提交订单时由于你快速点击多次，或者网络波动提交服务端多次请求，这会导致业务出现错误，如果做了幂等处理，则只有第一次请求是生效的。
    - 消息队列幂等：如何保障消息队列客户端对相同的消息仅消费一次。
        默认我们常见的消息队列都是具备持久性、高可靠的，消息消费失败了会重试，RabbitMQ、RocketMQ甚至会在失败多次后将消息加入 **死信队列** 。
        正是由于这种高可靠，导致服务器产生了客户端是否消费成功的疑问时，客户端就需要再次消费，导致消息被重复消费。

2、如果不做防重复提交或者幂等，会导致哪些问题：

    - 数据重复处理：如果用户在某个操作还在处理中时重复提交请求，可能会导致相同的数据被处理多次，造成数据的重复操作和处理逻辑的错误。
    
    - 重复写入：请求可能触发对数据库或其他存储系统的写入操作。如果请求被重复提交，可能会导致相同的数据被重复写入，破坏数据的一致性。
    
    - 资源浪费
    
    - 业务逻辑错误：在某些业务场景下，重复提交可能会导致业务逻辑错误。
    

3、如何解决幂等问题
    
    - 防重复提交
        分布式锁：当用户提交请求时，服务器端可以生成一个唯一的标识，例如UUID。在处理用户请求之前，服务器尝试获取一个分布式锁，若成功则正常执行业务流程。结束了释放锁。
        Token令牌：
            客户端在第一次调用业务请求之前会发送一个获取 Token 的请求。服务端会生成一个全局唯一的 ID 作为 Token，并将其保存在 Redis 中，同时将该 ID 返回给客户端。
            客户端进行第二次业务请求时，必须携带这个 Token。服务端验证成功，则执行业务逻辑并从 Redis 中删除该 Token。否则，即 Redis 没有该 Token，为重复操作，则直接返回指定的结果给客户端。
    - 防重复消费
        去重表：在使用 Redis 或者 MySQL 作为存储时，为实现幂等性而用于记录已经处理过的请求或操作以防重复执行。这里是用 Redis 作为去重组件实现，其实就是存储一个String类型的Key而已。
        当客户端发送请求时，服务端会先查询 Redis 去重表来检查该请求是否已经被处理过。如果不存在，就将新请求的唯一标识（如请求ID或标识）添加到 Redis 去重表中。
        如果消息在消费中，抛出异常，消息会触发延迟消费，在消息队列消费失败的场景下即发送到重试队列 RETRY TOPIC。

        
![image](https://github.com/user-attachments/assets/559f431f-30b0-408b-b2bc-689e91e22ed8)


4、MQ幂等的处理逻辑：

    - 拼接key前缀和SpEL表达式里面的key，作为最终的key放入Redis
    - 如果key存在，说明Key对应的消息要么已经执行完了，要么是还在执行，所以需要继续判断幂等标识对应的值，并抛出异常给上一层。
    - 根据处理逻辑中抛出来的异常，决定到底要让MQ重试，还是将消息直接吞了即不执行具体的消费流程
        （1）若消息还在处理，但是不确定是否执行成功，那么需要返回错误，方便 MQ 再次通过重试队列投递
        （2）若消息处理成功了，该消息直接返回成功即可
        （3）如果是客户端消费的时候出错了，要把幂等标识删除，这样才会不影响 MQ 再次通过重试队列投递
