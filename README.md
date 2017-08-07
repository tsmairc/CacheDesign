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


