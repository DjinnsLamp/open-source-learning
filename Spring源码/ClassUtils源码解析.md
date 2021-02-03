# ClassUtils源码解析

<!-- vscode-markdown-toc -->
* 1. [类加载器](#)
	* 1.1. [_getDefaultClassLoader_](#getDefaultClassLoader_)
	* 1.2. [_overrideThreadContextClassLoader_](#overrideThreadContextClassLoader_)
	* 1.3. [_forName_](#forName_)
* 2. [接口](#-1)
	* 2.1. [_getAllInterfacesForClassAsSet_](#getAllInterfacesForClassAsSet_)
	* 2.2. [_createCompositeInterface_](#createCompositeInterface_)
* 3. [方法获取](#-1)
	* 3.1. [_getMostSpecificMethod_](#getMostSpecificMethod_)

<!-- vscode-markdown-toc-config
	numbering=true
	autoSave=true
	/vscode-markdown-toc-config -->
<!-- /vscode-markdown-toc -->



##  1. <a name=''></a>类加载器

###  1.1. <a name='getDefaultClassLoader_'></a>_getDefaultClassLoader_

```java
//返回默认的类加载器，通常是线程上下文类加载器，如果没有，加载ClassUtils的ClassLoader作为后备
public static ClassLoader getDefaultClassLoader(){
    //需要返回的类加载器
    ClassLoader cl = null;
    try{
        //尝试获取线程上下文的类加载器
        cl = Thread.currentThread().getContextClassLoader();
    }catch(Throwable t){
        //fallback
    }
    //如果线程上下文类加载器获取失败
    if(cl == null){
        //使用ClassUtils的类加载器
        cl = ClassUtils.class.getClassLoader();
        //如果仍然获取失败
        try{
            //获取系统的类加载器
            cl = ClassLoader.getSystemClassLoader();
        }catch(Throwable ex){
            //无法访问系统类加载器，只能返回null了
        }
    }
    return cl;
} 
```

###  1.2. <a name='overrideThreadContextClassLoader_'></a>_overrideThreadContextClassLoader_

```java
//用指定类加载器去替换线程上下文类加载器
public static ClassLoader overrideThreadContextClassLoader(@Nullable ClassLoader classLoaderToUse){
    //获取当前线程
    Thread currentThread = Thread.currentThread();
    //获取当前线程的ClassLoader
    ClassLoader threadContextClassLoader = currentThread.getContextClassLoader();
    //如果不为空且不等于目标类加载器
    if(threadContextClassLoader != null && !threadContextClassLoader.equals(classLoaderToUse)){
        //设置当前线程上下文类加载器
        currentThread.setContextClassLoader(classLoaderToUse);
        //返回原来的上下文类加载器
        return threadContextClassLoader;
    }
    return null;
}
```

###  1.3. <a name='forName_'></a>_forName_

```java
//用指定的类加载器去加载特定名称的类
public static Class<?> forName(String name, @Nullable ClassLoader classLoader)
			throws ClassNotFoundException, LinkageError {
		//如果name为null，抛出异常
		Assert.notNull(name, "Name must not be null");
		//如果name是个原始类型名，就获取其对应的Class
		Class<?> clazz = resolvePrimitiveClassName(name);
		//如果clazz为null
		if (clazz == null) {
			//从缓存Map中获取name对应的Class
			clazz = commonClassCache.get(name);
		}
		//如果clazz不为null
		if (clazz != null) {
			//直接返回clazz
			return clazz;
		}

		// "java.lang.String[]" style arrays 'java.lang.String[]'样式数组，表示原始数组类名
		//如果name是以'[]'结尾的
		if (name.endsWith(ARRAY_SUFFIX)) {
			//截取出name '[]'前面的字符串赋值给elementClassName
			String elementClassName = name.substring(0, name.length() - ARRAY_SUFFIX.length());
			//传入elementClassName递归本方法获取其Class
			Class<?> elementClass = forName(elementClassName, classLoader);
			//新建一个elementClass类型长度为0的数组，然后获取其类型返回出去
			return Array.newInstance(elementClass, 0).getClass();
		}

		// "[Ljava.lang.String;" style arrays '[Ljava.lang.Stirng'样式数组，表示非原始数组类名
		//如果names是以'[L'开头 且 以';'结尾
		if (name.startsWith(NON_PRIMITIVE_ARRAY_PREFIX) && name.endsWith(";")) {
			//截取出name '[L'到';'之间的字符串赋值给elementName
			String elementName = name.substring(NON_PRIMITIVE_ARRAY_PREFIX.length(), name.length() - 1);
			//传入elementName递归本方法获取其Class
			Class<?> elementClass = forName(elementName, classLoader);
			//新建一个elementClass类型长度为0的数组，然后获取其类型返回出去
			return Array.newInstance(elementClass, 0).getClass();
		}

		// "[[I" or "[[Ljava.lang.String;" style arrays '[[I' 或者 '[[Ljava.lang.String;'样式数组，表示内部数组类名
		if (name.startsWith(INTERNAL_ARRAY_PREFIX)) {
			//截取出name '['后面的字符串赋值给elementName
			String elementName = name.substring(INTERNAL_ARRAY_PREFIX.length());
			//传入elementName递归本方法获取其Class
			Class<?> elementClass = forName(elementName, classLoader);
			//新建一个elementClass类型长度为0的数组，然后获取其类型返回出去
			return Array.newInstance(elementClass, 0).getClass();
		}

		//将classLoader赋值给clToUse变量
		ClassLoader clToUse = classLoader;
		//如果clToUse为null
		if (clToUse == null) {
			//获取默认类加载器，一般返回线程上下文类加载器，没有就返回加载ClassUtils的类加载器，
			//还是没有就返回系统类加载器，最后还是没有就返回null
			clToUse = getDefaultClassLoader();
		}
		try {
			//Class.forName(String,boolean,ClassLoader):返回与给定的字符串名称相关联类或接口的Class对象。
			//第一个参数:类的全名
			//第二参数:是否进行初始化操作，如果指定参数initialize为false时，也不会触发类初始化，其实这个参数是告诉虚拟机，是否要对类进行初始化。
			//第三参数:加载时使用的类加载器。
			//通过clsToUse获取name对应的Class对象
			return Class.forName(name, false, clToUse);
		}
		catch (ClassNotFoundException ex) {
			//如果找到不到类时
			//获取最后一个包名分割符'.'的索引位置
			int lastDotIndex = name.lastIndexOf(PACKAGE_SEPARATOR);
			//如果找到索引位置
			if (lastDotIndex != -1) {
				//name.substring(0, lastDotIndex):截取出name的包名
				//name.substring(lastDotIndex + 1):截取出name的类名
				//尝试将name转换成内部类名,innerClassName=name的包名+'$'+name的类名
				String innerClassName =
						name.substring(0, lastDotIndex) + INNER_CLASS_SEPARATOR + name.substring(lastDotIndex + 1);
				try {
					//通过clToUse获取innerClassName对应的Class对象
					return Class.forName(innerClassName, false, clToUse);
				}
				catch (ClassNotFoundException ex2) {
					// Swallow - let original exception get through 吞噬 - 让原异常通过
				}
			}
			//当将name转换成内部类名仍然获取不到Class对象时,抛出异常
			throw ex;
		}
	}
```

##  2. <a name='-1'></a>接口

###  2.1. <a name='getAllInterfacesForClassAsSet_'></a>_getAllInterfacesForClassAsSet_

```java
//找出给定接口的所有实现类
public static Set<Class<?>> getAllInterfacesForClassAsSet(Class<?> clazz, @Nullable ClassLoader classLoader) {
	//如果clazz为null，抛出异常
	Assert.notNull(clazz, "Class must not be null");
	//如果clazz是接口 且 clazz在给定的classLoader中是可见的
	if (clazz.isInterface() && isVisible(clazz, classLoader)) {
		//返回一个不可变只包含clazz的Set对象
		return Collections.singleton(clazz);
	}
	//LinkedHashSet:是一种按照插入顺序维护集合中条目的链表。这允许对集合进行插入顺序迭代。
	//也就是说，当使用迭代器循环遍历LinkedHashSet时，元素将按插入顺序返回。
	//然后将哈希码用作存储与键相关联的数据的索引。将键转换为其哈希码是自动执行的。
	//定义一个用于存储接口类对象的Set
	Set<Class<?>> interfaces = new LinkedHashSet<>();
	//将clazz赋值给current,表示当前类对象
	Class<?> current = clazz;
	//遍历，只要current不为null
	while (current != null) {
		//获取current的所有接口
		Class<?>[] ifcs = current.getInterfaces();
		//遍历ifcs
		for (Class<?> ifc : ifcs) {
			//如果ifc在给定的classLoader中是可见的
			if (isVisible(ifc, classLoader)) {
				//将ifc添加到interfaces中
				interfaces.add(ifc);	
            }
		}
		//获取current的父类重新赋值给current
		current = current.getSuperclass();
    }
	//返回存储接口类对象的Set
    return interfaces;
}
```

###  2.2. <a name='createCompositeInterface_'></a>_createCompositeInterface_

```java
//为指定接口集合创建一个复合的JDK代理类
public static Class<?> createCompositeInterface(Class<?>[] interfaces, @Nullable ClassLoader classLoader) {
	//如果interface为null,抛出异常
	Assert.notEmpty(interfaces, "Interface array must not be empty");
	//创建动态代理类，得到的Class对象可以通过获取InvocationHandler类型的构造函数进行实例化代理对象
	//详细用法：https://blog.csdn.net/u012516166/article/details/76033249
	return Proxy.getProxyClass(classLoader, interfaces);
}
```

##  3. <a name='-1'></a>方法获取

###  3.1. <a name='getMostSpecificMethod_'></a>_getMostSpecificMethod_

```java
//给定一个可能来自接口以及在当前反射调用中使用的目标类的方法,找到相应的目标方法
//如果安全性检查开启，例如clazz.getDeclaredMethod()不被允许，直接返回原方法	
public static Method getMostSpecificMethod(Method method, @Nullable Class<?> targetClass) {
	//如果targetClass不为null 且 targetClass 不是声明method的类 且 method在targetClass中可重写
	if (targetClass != null && targetClass != method.getDeclaringClass() && isOverridable(method, targetClass)) {
		try {
			//如果method是public方法
			if (Modifier.isPublic(method.getModifiers())) {
				try {
					//获取targetClass的方法名为method的方法名和参数类型数组为method的参数类型数组的方法对象
					return targetClass.getMethod(method.getName(), method.getParameterTypes());
				}
				catch (NoSuchMethodException ex) {
					//捕捉没有找到方法异常,直接返回给定的方法
					return method;
				}
			}
			else {
				//反射获取targetClass的方法名为method的方法名和参数类型数组为method的参数类型数组的方法对象
				Method specificMethod =
						ReflectionUtils.findMethod(targetClass, method.getName(), method.getParameterTypes());
                //如果specifiMethod不为null,就返回specifiMetho,否则返回给定的方法
				return (specificMethod != null ? specificMethod : method);
			}
		}
		catch (SecurityException ex) {
			// Security settings are disallowing reflective access; fall back to 'method' below.
			// 安全性设置不允许反射式访问；回到下面的'method'
		}
	}
	//直接返回给定的方法
	return method;
}
```

