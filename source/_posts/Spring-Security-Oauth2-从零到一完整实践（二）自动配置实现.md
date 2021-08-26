---
layout: post
title: Spring Security Oauth2 从零到一完整实践（二）自动配置实现
date: 2021-04-19 15:00:24
tags:
---

<p>

<!-- more -->

前面我们学习了四种授权模式的两种，因为那两种分别满足了方便和安全，已经能够胜任大多数情况，我们从简开始，先来用最简单的方式开始。

{% note info no-icon %}
github 地址：[spring-security-oauth2-demo](https://github.com/cdwxw/spring-security-oauth2-demo)

博客地址：[isee.wang](/)
{% endnote %}

## 系列文章

1. [较为详细的学习 oauth2 的四种模式其中的两种授权模式](/Spring-Security-Oauth2-从零到一完整实践（一）/)
2. spring boot oauth2 自动配置实现
3. [spring security oauth2 授权服务器配置](/Spring-Security-Oauth2-从零到一完整实践（三）授权服务器/)
4. [spring security oauth2 资源服务器配置](/Spring-Security-Oauth2-从零到一完整实践（四）资源服务器/)
5. [spring security oauth2 自定义授权模式（手机、邮箱等）](/Spring-Security-Oauth2-从零到一完整实践（五）自定义授权模式（手机、邮箱等）/)
6. [spring security oauth2 踩坑记录](/Spring-Security-Oauth2-从零到一完整实践（六）踩坑记录/)

## spring boot oauth2 自动配置实现

spring boot 最大一个特点就是 **约定大于配置，去繁就简** 。既然如此，他自然也提供了一套 oauth2 的自动化配置，我们先来实验他完成的自动化配置看看效果。

首先创建我们的 module 如下：

![](/images/Spring-Security-Oauth2-从零到一完整实践（二）自动配置实现/1.png)
![](/images/Spring-Security-Oauth2-从零到一完整实践（二）自动配置实现/2.png)
![](/images/Spring-Security-Oauth2-从零到一完整实践（二）自动配置实现/3.png)

{% note warning %}
注意：这个过程以后不再截图演示。
{% endnote %}

在 pom.xml 中添加如下依赖

```xml
<dependencies>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-security</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.security.oauth.boot</groupId>
        <artifactId>spring-security-oauth2-autoconfigure</artifactId>
        <version>${spring.boot.version}</version>
    </dependency>
</dependencies>
```

## spring security 保护的资源

默认情况下，我们加入了 spring security 的依赖，他会保护我们的资源。现在添加启动类如下：

```java
@SpringBootApplication
public class Application {
    public static void main(String[] args) {
        SpringApplication.run(Application.class, args);
    }
}
```

直接启动，控制台如下

![](/images/Spring-Security-Oauth2-从零到一完整实践（二）自动配置实现/4.png)

然后访问 http://127.0.0.1:8080 ，如下

![](/images/Spring-Security-Oauth2-从零到一完整实践（二）自动配置实现/5.png)

使用用户名 user 密码为控制台打印的那一串登录即可，成功后如下：

![](/images/Spring-Security-Oauth2-从零到一完整实践（二）自动配置实现/6.png)

但是现在默认的是 spring security，我们接下来来实验一下 oauth2 保护我们的资源

## spring security oauth2 保护资源

我们在前面提到过 oauth2 的几种角色，我们现在一步一步的来。在下面的授权服务器与资源服务器，我们将他们存在同一个应用之中使用，先以最快速的方式学习与了解，后面再来考虑分离的问题。

## 授权服务器

首先第一步是授权服务器，因为它是我们获取与请求凭证的地方，我们需要他来给我们下发令牌凭证，如何开启呢？需要一个启动注解@EnableAuthorizationServer 添加在启动类上即可。

同时为了方便测试，我们添加一个 ResourceController 来设置一个资源访问路径如下：

![](/images/Spring-Security-Oauth2-从零到一完整实践（二）自动配置实现/7.png)

这个时候我们再启动，然后去访问就会发现不需要登录了

![](/images/Spring-Security-Oauth2-从零到一完整实践（二）自动配置实现/8.png)

因为现在 spring security 已经不再去管理你的应用了，**然而现在你只配置了授权服务器，他不会保护你的应用程序的，所以不需要登录了。我们暂时不管，现在授权服务器的任务是验证身份并下发令牌，我们来测试一下。**

为了方便查看路径，我们开启 debug 日志，以便更好的理解整个过程；添加 application.yml 以及内容如下：

```yaml application.yml
logging:
  level:
    org:
      springframework:
        security: DEBUG
```

然后运行，你会看到如下画面：

![](/images/Spring-Security-Oauth2-从零到一完整实践（二）自动配置实现/9.png)

在我们启动的时候为我们自动生成的了一些东西：

- 用户密码
- 客户端 id
- 客户端密码
- 添加了七个路径
- 使用权限表达式设置访问权限

### 授权码模式

spring security oauth 授权服务器默认开启授权码模式。那么按照我们前面说的，**授权码模式是在授权端点 /oauth/authorize 请求授权码**，路径应该如下：

```
http://localhost:8080/oauth/authorize?response_type=code&client_id=b6a2f868-cba4-4c44-bfec-4cde082e979f&redirect_uri=http://example.com&scope=all
```

{% note warning no-icon %}
**为什么回调地址是 http://example.com ？**

因为我们现在没有任何应用，需要一个页面来接收回调后的授权码，所以随便找了一个。
{% endnote %}

访问后会出现如下错误

![](/images/Spring-Security-Oauth2-从零到一完整实践（二）自动配置实现/10.png)

这是因为我们没有配置 spring security 造成的，所以需要回去配置一下，使用默认配置即可，添加一个配置类如下：

![](/images/Spring-Security-Oauth2-从零到一完整实践（二）自动配置实现/11.png)

重启启动后，你会发现，现在的网页又不能访问了，全都提示需要登录了，暂时不管。

**我们使用新的 client id 去请求授权**，他会自动跳转到登录页面了，如下：

![](/images/Spring-Security-Oauth2-从零到一完整实践（二）自动配置实现/12.png)

这里的步骤是在授权服务器上面的，就像我们点击第三方登录的 qq 的时候，是跳转到 腾讯 自己的登录页面的。用户名 user ，密码为控制台生成的，登录后如下：

![](/images/Spring-Security-Oauth2-从零到一完整实践（二）自动配置实现/13.png)

它提示我们 **至少为客户端注册一个回调地址** ，我们请求授权的时候传递了一个回调地址了，这里为什么还需要一个呢？这个很容易理解，**因为你传递过来的回调地址授权服务器不知道是否合法，可能会在传输的中途被篡改，所以在授权服务器里面需要你注册一个回调地址，与你传递过来的进行对比，如果匹配才会携带授权码进行回调。**这样就有效避免中途被篡改的问题了，所以现在我们需要去注册一个回调地址，在 application.yml 中配置：

![](/images/Spring-Security-Oauth2-从零到一完整实践（二）自动配置实现/14.png)

然后重新启动，再次携带新的客户端id进行访问：

![](/images/Spring-Security-Oauth2-从零到一完整实践（二）自动配置实现/15.png)

当我们确认授权了以后，这个授权流程也就完毕了，也就相当于前面 **角色中的抽象流图的 AB 完成了** ，我们看看得到的授权码

![](/images/Spring-Security-Oauth2-从零到一完整实践（二）自动配置实现/16.png)

**接下来我们需要使用此授权码去完成请求令牌的操作也就是前面说到的第二个请求**，我们需要 postman 接口测试工具：

![](/images/Spring-Security-Oauth2-从零到一完整实践（二）自动配置实现/17.png)

当我们设置好授权信息以后他会为我们自动添加一个请求头

![](/images/Spring-Security-Oauth2-从零到一完整实践（二）自动配置实现/18.png)

请求头的添加方式就是前面提到的 **客户端加密** 的那一部分，不再赘述。然后我们设置 **第二个请求的请求参数** 如下：

![](/images/Spring-Security-Oauth2-从零到一完整实践（二）自动配置实现/19.png)

这样我们就获取到令牌了，到这一步，也就相当于前面 角色中的抽象流图的 CD 也完成了 。这就是授权码模式获取令牌的两个请求的过程。

### 密码模式

接下来我们来试一下 **密码模式** 来获取令牌，就像前面所说，他只有一个请求即可，所以我们只要用 postman 携带参数请求一下就好了。

![](/images/Spring-Security-Oauth2-从零到一完整实践（二）自动配置实现/20.png)

相比起来，密码模式就简单太多啦！但是用户名密码是在客户端那里的，而不是在授权服务器这边的，所以只能是完全信得过的应用才能够使用！

### 快速自定义

**所谓快速自定义，就是我们不需要写代码，通过配置文件即可完成自定义。**对于 oauth2 客户端，提供了如下配置让我们快速自定义：

![](/images/Spring-Security-Oauth2-从零到一完整实践（二）自动配置实现/21.png)

当然，只能提供一个客户端使用。我们后面再来详细学习如何自定义

### 配置用户

其实用户就是用的 spring security 的用户，但是由于不能够直接在配置文件中指定用户的密码了，所以我们需要建一个 UserService 的实现类。不过在那之前，我们需要配置一个密码加密器，让我们的密码得到保障，而不是明文传输，spring 5 以后这个是必须指定的。

```java
@Bean
public PasswordEncoder passwordEncoder() {
    return new BCryptPasswordEncoder();
}
```

BCrypt 不可逆的加密算法，无法通过解密密文得到明文，和其他对称或非对称加密方式不同的是，不是直接解密得到明文，也不是二次加密比较密文，而是把明文和存储的密文一块运算得到另一个密文，如果这两个密文相同则验证成功。对于同一个密码，每次加密出来是完全不同的，所以安全性很可靠。

下面的用户我们用最快捷的方式来进行创建，创建两个内存用户：

```java
@Bean
@Override
protected UserDetailsService userDetailsService() {
    InMemoryUserDetailsManager manager = new InMemoryUserDetailsManager();
    manager.createUser(User.withUsername("user")
                   .password(passwordEncoder().encode("123456"))
                   .authorities("ROLE_USER").build());
    manager.createUser(User.withUsername("admin")
                   .password(passwordEncoder().encode("admin"))
                   .authorities("ROLE_ADMIN").build());
    return manager;
}
```

![](/images/Spring-Security-Oauth2-从零到一完整实践（二）自动配置实现/22.png)

获取令牌看看

![](/images/Spring-Security-Oauth2-从零到一完整实践（二）自动配置实现/23.png)

## 资源服务器

现在我们取到了 token，我们来尝试访问一下被保护的资源，使用浏览器访问：

![](/images/Spring-Security-Oauth2-从零到一完整实践（二）自动配置实现/24.png)

你会发现同样需要你登录，因为现在是由 spring security 进行资源保护的。那么我们看看携带 token 使用 postman 测试一下会怎样呢？

![](/images/Spring-Security-Oauth2-从零到一完整实践（二）自动配置实现/25.png)

会发现是 401，也就是令牌是无效的，原因就是因为现在资源的保护是由传统的 spring security 来进行保护的。接下来我们就要配置我们的资源服务器。

同授权服务器一样，资源服务器的启动也只需要一个注解就可以了：@EnableResourceServer，启动类添加此注解如下

```java
@SpringBootApplication
@EnableAuthorizationServer
@EnableResourceServer
public class Application {
    public static void main(String[] args) {
        SpringApplication.run(Application.class, args);
    }
}
```

启动下应用，通过浏览器看看

![](/images/Spring-Security-Oauth2-从零到一完整实践（二）自动配置实现/26.png)

你会发现已经不能够登录了。现在重新用密码模式请求下 token，截图省略，然后获取 token 后去请求我们受保护的资源试一试：

![](/images/Spring-Security-Oauth2-从零到一完整实践（二）自动配置实现/27.png)

现在携带正确的 token 就可以请求到数据了，这就是已经由 spring security oauth 来进行资源保护了。

对于资源服务器的自定义配置，目前只有一个地方，就是资源的 id ，如下：

![](/images/Spring-Security-Oauth2-从零到一完整实践（二）自动配置实现/28.png)

如果两者不相同会抛出如下异常：

```json
{
    "error": "access_denied",
    "error_description": "Invalid token does not contain resource id (resource-id)"
}
```

其他的配置我们后面再说，因为他主要涉及到与授权服务器的分离的情况。

## 其他

除了获取 token 和请求以外，她还可以配置一些默认的实现。

### 解析 token

我们需要在配置文件中添加如下配置：

```yaml
security:
  oauth2:
    authorization:
      # 允许使用 /oauth/check_token 端点
      check-token-access: isAuthenticated()
```

![](/images/Spring-Security-Oauth2-从零到一完整实践（二）自动配置实现/29.png)

熟悉 SqEL 表达式的同学应该知道 isAuthenticated() 的意思，它允许此端点的访问，重启后获取新的 token，来访问试一试，参数前面已经说过不再赘述：

![](/images/Spring-Security-Oauth2-从零到一完整实践（二）自动配置实现/30.png)

这样就成功解析了我们的 token 信息！

{% note warning %}
使用默认配置的情况下且不增加类的情况下，我们是没有办法刷新 token 的。
{% endnote %}

## 总结

这个部分是我们最基础的部分，也是最为简单的部分，使用 spring boot oauth 的自动配置完成了简单的授权服务器和资源服务器的配置， 通过这两个服务器的配置就可以快速搭建起来 oauth2 的授权流程，为我们省掉了很多麻烦事儿，当然，自动配置有好处也有坏处，由于他自动帮我们配置好了很多，能满足很多的小型应用的需求了。但是要求总是在变化的，所以有些不符合我们要求的地方我们需要去自己自定义的，下面我们就要进入 spring security oauth2 完整自定义配置环节，分为两个部分，一个授权服务器，一个资源服务器的配置。
