# Sa-token源码解析

<style>
    .hljs-keyword,
    .hljs-selector-tag,
    .hljs-built_in,
    .hljs-name,
    .hljs-tag {

        color: #569CD6;

    }

    .hljs-string,
    .hljs-type,
    .hljs-built_in,
    .hljs-builtin-name,
    .hljs-selector-id,
    .hljs-selector-attr,
    .hljs-selector-pseudo,
    .hljs-addition,
    .hljs-variable,
    .hljs-template-variable {

        color: #CE9178;

    }



    .hljs-comment,
    .hljs-deletion,
    .hljs-meta {

        color: #6A9955;

    }

    .hljs-bullet,
    .hljs-quote,
    .hljs-number,
    .hljs-regexp,
    .hljs-literal,
    .hljs-link {

        color: #B5CEA8;

    }

    .hljs-code,
    .hljs-title,
    .hljs-section,
    .hljs-selector-class {

        color: #DCDCAA;

    }

    .hljs-keyword,
    .hljs-selector-tag,
    .hljs-built_in,
    .hljs-name,
    .hljs-tag {

        color: #569CD6;

    }

    .hljs-attr {

        color: #9CDCFE;

    }
    .cnblogs-markdown .hljs
    {
        background: #1E1E1E !important;
        background-image: linear-gradient(to top, #a18cd1 0%, #fbc2eb 100%);
    }
</style>

--------------

## Sa-token配置选项

Sa-token可以零启动配置和通过配置文件方式进行参数的配置，所有的配置选项如下，配置文件对应的参数被封装为`SaTokenConfig`，`SaTokenConfig`含有配置参数的默认值，配置文件中未出现的配置选项会直接使用默认值

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

`SaTokenConfigFactory`负责从`classpath:sa-token.properties`配置文件中读取参数，封装成`SaTokenConfig`，**在非IOC环境下不会使用到该类**。

## Token(令牌应该如何设计，包含哪些参数)

Token在权限认证模块中的作用是无可替代的，所有的认证流程基本都是基于token，在Sa-token项目中，token的常用信息被封装在`SaTokenInfo`中，大致内容如下:

```java
public class SaTokenInfo {
	/** token名称 */
	public String tokenName;
	/** token值 */
	public String tokenValue;
	/** 此token是否已经登录 */
	public Boolean isLogin;
	/** 此token对应的LoginId，未登录时为null */
	public Object loginId;
	/** LoginKey账号体系标识 */
	public String loginKey;
	/** token剩余有效期 (单位: 秒) */
	public long tokenTimeout;
	/** User-Session剩余有效时间 (单位: 秒) */
	public long sessionTimeout;
	/** Token-Session剩余有效时间 (单位: 秒) */
	public long tokenSessionTimeout;
	/** token剩余无操作有效时间 (单位: 秒) */
	public long tokenActivityTimeout;
	/** 登录设备标识 */
	public String loginDevice;
}
```

关于token的一些常量则被封装在`SaTokenConsts`中:

```java
public class SaTokenConsts{
    /* 如果token为本次请求新创建的，则以此字符串作为key存储在request中 */
    public static final String JUST_CREATED_SAVE_KEY = "JUST_CREATED_SAVE_KEY_";
    /*  */
}
```