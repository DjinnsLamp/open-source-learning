# Sa-token源码解析

--------------

## Sa-token配置选项

```yml
spring: 
    # sa-token配置
    sa-token: 
        # token名称 (同时也是cookie名称)
        #默认: satoken
        token-name: satoken
        # token有效期，单位s 默认30天, -1代表永不过期 
        # 默认: 30 * 24 * 60 * 60
        timeout: 2592000
        # token临时有效期 (指定时间内无操作就视为token过期) 单位: 秒
        # 默认: -1
        activity-timeout: -1
        # 是否允许同一账号并发登录 (为true时允许一起登录, 为false时新登录挤掉旧登录)
        # 默认: true 
        allow-concurrent-login: false
        # 在多人登录同一账号时，是否共用一个token (为true时所有登录共用一个token, 为false时每次登录新建一个token)
        # 默认： true 
        is-share: false
        # 是否尝试从请求体里面读取token
        # 默认: true
        is-read-body: true
        # 是否尝试从header里面读取token
        # 默认: true
        is-read-head: true
        # 是否尝试从cookie中读取文件
        # 默认: true
        is-read-cookie: true
        # token风格，可取值: uuid, simple-uuid, random-32, random-64, random-128, tik
        # 默认: uuid
        token-style: uuid
        # 默认dao层实现中，每次清理过期数据间隔的时间，单位：秒，-1代表不启动定时清理
        # 默认: 30
        data-refresh-period: 30
        # 获取[token专属session]时是否必须登录，如果配置为true，会在每次获取[token-session]时校验是否登录
        # 默认: true
        token-session-check-login: true
        # 是否在初始化配置的时候打印版本字符画
        # 默认: true
        is-v: true
```

## Sa-token关于session的设计

```java
//session model
public class SaSession implements Serializable {
	/** 此Session的id */
	private String id;
	/** 此Session的创建时间 */
	private long createTime;
	/** 此Session的所有挂载数据 */
	private Map<String, Object> dataMap = new ConcurrentHashMap<String, Object>();
    /** 此Session绑定的token sign列表 */
    private List<TokenSign> tokenSignList = new Vector<TokenSign>();
    ...
}
```

```java
//token sign model
public class TokenSign implements Serializable {
	/** token值 */
	private String value;
	/** 所在设备标识 */
	private String device;
    ...
}
```