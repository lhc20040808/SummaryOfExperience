# 单例模式

### 定义

保证一个类只有一个实例，且自行实例化并提供一个访问他的全局访问点。

### 场景

确保某个类有且只有一个对象，避免产生多个对象消耗过多的资源，或某种类型对象应该只有一个。

### 实现方式

#### 1、饿汉模式

```java
public class SingleTon {
	private volatile static SingleTon sInstance = new SingleTon();
	
	private SingleTon(){
		
	}
	
	public static SingleTon getInstance(){
		return sInstance;
	}
}
```

#### 2、懒汉模式

优点：使用时才会实例化

缺点：getInstance()需要同步，造成不必要的开销

```java
public class SingleTon {
	private static SingleTon sInstance = null;
	
	private SingleTon(){
		
	}
	
	public static synchronized SingleTon getInstance(){
		if(sInstance == null){
			sInstance = new SingleTon();
		}
		return sInstance;
	}
}
```

#### 3、Double Check Lock(DCL)

优点：资源利用率高

缺点：由于jdk1.5以前的java内存模型，不加volatile关键字可能会导致DCL失效

```java
public class SingleTon {
	private volatile static SingleTon sInstance = null;
	
	private SingleTon(){
		
	}
	
	public static SingleTon getInstance(){
		if(sInstance == null){
			synchronized(SingleTon.class){
				if(sInstance == null){
					sInstance = new SingleTon();
				}
			}
		}
		return sInstance;
	}
}
```

##### 为什么不加volatile关键字会存在DCL失效的问题？

```java
A a = new A();
```

这样一句代码大概做了以下几件事情：

1、类加载检查

2、给实例分配内存空间

3、设置实例基本信息

4、初始化与调用构造函数

5、将a对象指向分配的内存空间

由于Java编译器允许处理器乱序执行，上面第4步，第5步的顺序无法保证。可能对象被指向了分配的内存空间单还没有初始化完成，这时另一个线程取走对象并使用就会出错。这就是DCL失效的问题。

#### 4、静态内部类实现单例

优点：第一次调用getInstance()方法时才会加载Holder类，并初始化sInstance。由jvm保证线程安全并保证单例对象的唯一性。

```java
public class SingleTon {

	public SingleTon(){
		
	}
	
	public static SingleTon getInstance(){
		return Holder.sIntance;
	}
	
	private static class Holder{
		private static final SingleTon sIntance = new SingleTon();
	}

}
```

#### 5、使用容器实现单例

优点：统一管理，降低使用成本。隐藏具体实现，降低耦合。

```java
public class SingleTonManager {
	private static Map<String,Object> objMap = new HashMap<String,Object>();
	
	private SingleTonManager(){
		
	}
	
	public static void put(String key,Object obj){
		if(!objMap.containsKey(key)){
			objMap.put(key, obj);
		}
	}
	
	public static Object get(String key){
		return objMap.get(key);
	}
}
```



参考资料：《[Java多线程 -- 正确使用Volatile变量](http://blog.csdn.net/fw0124/article/details/6669984)》