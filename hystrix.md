# Hystrix 处理流程

![截屏2020-07-20 上午11.46.27](https://tva1.sinaimg.cn/large/007S8ZIlgy1ggxajq5f48j31pw0u07wh.jpg)

每个熔断器默认维护10个bucket,每秒一个bucket,每个bucket记录成功,失败,超时,拒绝的状态。通过这种方式记录调用情况并根据条件出发熔断机制。

