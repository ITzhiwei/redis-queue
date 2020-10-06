# redis 高度可靠的异步即时与延迟消息队列设计
* 本人曾在项目中使用的消息队列
* RabbitMQ 或许是最佳选择，如果不想使用 RabbitMQ ，可看下面

## redis 消息队列使用 list 而不是 订阅发布，原因：
* 致命点：发布与订阅耦合，消息没有持久化，导致消息消费成功不可靠，易丢失  
* redis 订阅发布的多方同时消费，list 也可多做一层散发消息达到同样功能  
* 所以redis的订阅发布是不合适用来做消息队列的  

## redis 存储字段及其作用
* AsyncInfo 当前需消费的消息，数据结构使用list  
* AsyncDelayInfo 需要延迟消费的消息，数据结构使用有序集合
* AsyncNawInfo 当前正在消费的消息，数据结构使用有序集合
* AsyncErrorInfo 尝试多次消费失败的消息，数据结构使用list
* AsyncInfoStartTime 记录消息队列消费进程最近一次执行时间
* AsyncDelayInfoStartTime 记录延迟队列进程的最近执行时间
* AsyncNawInfo 有序集合，用来存放当前正在消费的消息，增加消息消费的可靠性，score 代表消费开始时间
* AsyncErrorInfo 有序集合，用来存放消费失败的消息，score 代表最后消费时间
* AsyncInfoDuplicate 有序集合，数据与AsyncInfo一致，score 代表消息生产或投递时间，在将消息插入 AsyncNawInfo 后删除对应消息
* AsyncDelayInfoDuplicate 有序集合，数据与AsyncDelayInfo一致，在将消息投递进 AsyncInfo、AsyncInfoDuplicate 后删除对应消息

## redis 消息格式
* {id:1, data:{...}} id使用redis incr获取自增ID，防止出样一模一样的消息导致集合异常

## redis 消息队列的进程列表
* 消费进程
* 延迟队列的投递进程，将到达执行时间的消息投递到 AsyncInfo 进行消费
* 非常驻内存 crontab 定时启动进程，假死检测使用，下面有详细说明

## redis 消息队列的管理
* 使用 Supervisor 管理消息消费进程和延迟队列的投递进程，进程挂掉后自动重启
* 使用 crontab 定时检测 AsyncInfoStartTime 和 AsyncDelayInfoStartTime ，判断记录的时间是否过大，过大即代表进程假死，需要重启进程。
* 消息消费进程和延迟队列的投递进程都需要设置**信号**，防止进程关闭或重启时程序没有执行完毕，不能使用kill -9杀死进程。
* 使用 crontab 定时检测 AsyncNawInfo 是否存在1分钟前（或根据自己的程序设定时间上限）的消息尚未执行完毕，若存在则代表进程消费过程中被强制杀死或代码存在BUG，需要人工处理。
* 使用 crontab 定时检测 AsyncErrorInfo 是否存在消息，存在则通知排查问题。
* 机器停电 or 机器故障 or kill -9 等导致 redis 被强制停止时，开启前需检查 AsyncInfoDuplicate 和 AsyncInfo 的差集，再看差集数据是否存在于 AsyncNawInfo 中，存在则删除，将差集重新插入 AsyncInfo 中进行消息恢复，这个检测程序可写在消息消费进程头部每次启动时执行；AsyncDelayInfoDuplicate 的检测同理。

## 缺点与优点
* 优点：redis是项目必备服务，使用redis做消息队列，同时可经过设计提高可靠性，无需安装更多服务
* 缺点：因为 redis 不支持我们所想要的事务（类似mysql事务，redis事务和lua脚本都不能达到数据可靠），导致可靠程序需要复杂的设计。

## 结语
* 既然 redis 设计消息队列这么不省心，何不使用专业的 RabbitMQ 
