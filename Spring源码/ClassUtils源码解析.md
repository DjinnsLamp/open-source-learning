# ClassUtils源码解析

## 类加载器

### _getDefaultClassLoader_

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

### _overrideThreadContextClassLoader_

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

### _forName_

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

