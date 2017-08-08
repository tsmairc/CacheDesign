# 系统redis缓存设计
下面是设计图：
![](https://raw.githubusercontent.com/tsmairc/CacheDesign/master/img/redis_uml.png)

* Cache的实现类，也就是说图上的CodeInfoCache类，类里面的方法基本是静态方法。这样做的好处是方便service使用,下面是设置对象进redis的一个方法
```java
public static boolean putCodeInfoVO(String code, CodeInfoVO codeInfoVO){
	if(StringUtils.isBlank(code) || codeInfoVO == null){
		return false;
	}

	return getInst()._put(code, codeInfoVO);
}
```

* CacheService是通过dubbo开放的服务，这里要使用上面CodeInfoCache类的话，如下面代码片段所示:
```java
/**
 * 获取后台配置的code_info
 */
@Override
public CodeInfoVO getCodeInfoVOByCode(String code){
	return CodeInfoCache.getCodeInfoVO(code);
}
```

* Cache这个虚类的作用是实现ICache里面共用的方法，剩下不共用的方法留下到具体的实现类去实现。
* CacheFactory目的是拿到目体缓存方案的实现类，例如我现在要使用redis，那么就通过spring读取配置，然后实例化对应的类（RedisCache）
* CacheClientPool是一个连接池，连接一次后就会缓存下来，第二次就不会再重新连接。
* ICacheClient是一个适配器，用于适配不同的缓存组件。


### 界面缓存获取及一键刷新思路
![](https://raw.githubusercontent.com/tsmairc/CacheDesign/master/img/refresh.png)
<br/>类似做成上图的效果，可以通过反射找到所有缓存实现类，然后循环去调用缓存实现类的刷新方法。对缓存分组可以通过注解完成，例如：
```java
@TypeCache( typeCode="demo")
```
反射读取时，顺便对其分类。

### java本地缓存的运用（guava）
好好利用java本地缓存能更进一步加快缓存的读取速度，像读取，设置，删除这三种操作，例如读取，先读java本地缓存，为空的话再读redis。
```java
//这里直接封装到Cache里面，因为这个方法对所有缓存实现类是通用的，其它三种操作一样的。
T value = null;
value = getLocalCache(pkey, key, value);
if(value == null){
	value = (T)CacheFactory.getCacheClient().get(getNamespace(), pkey, key);
	if(value != null){
		putLocalCache(pkey, key, value);
	}
}
```
guava的缓存操作基本上与redis相似，下面是实例化的一些例子。
```java
public LocalCache(long size){
	cache = CacheBuilder.newBuilder().maximumSize(size).build();
}

public LocalCache(long size, long expire, TimeUnit expireUnit){
	cache = CacheBuilder.newBuilder()
		.maximumSize(size)
		.expireAfterWrite(expire, expireUnit).build();
}
```
<p>
下面介绍下刷新缓存的后台操作，分布式时刷新，多台机如何同步这确是一个大问题，这里直接通过mq去发送信息刷新，然后每一台机做广播消费，先清空机器本地缓存，再清redis。
</p>

### redis部署模式
* 1.单机模式
* 2.集群模式
<br/>多台机，有主从节点。
http://www.cnblogs.com/wuxl360/p/5920330.html
* 3.哨兵模式
<br/>Sentinel（哨兵）是Redis 的高可用性解决方案：由一个或多个Sentinel 实例 组成的Sentinel 系统可以监视任意多个主服务器，以及这些主服务器属下的所有从服务器，并在被监视的主服务器进入下线状态时，自动将下线主服务器属下的某个从服务器升级为新的主服务器。
http://www.cnblogs.com/jaycekon/p/6237562.html

