## 手撕OKhttp源码-连接池和路由

### 连接池

#### RealConnectionPool

多路复用的逻辑

#### 连接池维护

```
// 双端队列
 private val connections = ArrayDeque<RealConnection>()
```



### 路由

#### RouteSelector

#### Selection

#### Route