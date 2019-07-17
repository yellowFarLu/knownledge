### 服务器后后台启动

./redis-server &





### 客户端连接

redis-cli -h {host} -p {port}方式连接，然后所有的操作都是在交互的方式实现，不需要再执行redis-cli了。

$redis-cli -h 127.0.0.1-p 6379