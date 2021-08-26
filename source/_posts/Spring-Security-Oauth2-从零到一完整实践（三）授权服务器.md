---
layout: post
title: Spring Security Oauth2 从零到一完整实践（三）授权服务器
date: 2021-04-20 09:18:16
tags:
---

<p>

<!-- more -->

前面说了自动配置，现在就是来说自定义配置啦，这个是十分重要的一节，可以说 oauth2 的核心就是授权服务器了，所有的角色都是围绕着授权服务器而运作的，这里基本包含了资源服务器的所有配置。

{% note warning %}
注意：spring security oauth2 模块已经过期，见 [github](https://github.com/spring-projects/spring-security-oauth#-deprecation-notice-)。
{% endnote %}

{% note info no-icon %}
github 地址：[spring-security-oauth2-demo](https://github.com/cdwxw/spring-security-oauth2-demo)

博客地址：[isee.wang](/)
{% endnote %}

## 系列文章

1. [较为详细的学习 oauth2 的四种模式其中的两种授权模式](/Spring-Security-Oauth2-从零到一完整实践（一）/)
2. [spring boot oauth2 自动配置实现](/Spring-Security-Oauth2-从零到一完整实践（二）自动配置实现/)
3. spring security oauth2 授权服务器配置
4. [spring security oauth2 资源服务器配置](/Spring-Security-Oauth2-从零到一完整实践（四）资源服务器/)
5. [spring security oauth2 自定义授权模式（手机、邮箱等）](/Spring-Security-Oauth2-从零到一完整实践（五）自定义授权模式（手机、邮箱等）/)
6. [spring security oauth2 踩坑记录](/Spring-Security-Oauth2-从零到一完整实践（六）踩坑记录/)

## spring security oauth2 授权服务器

我们首先再次回顾下授权服务器的详细作用：

1. 客户端的验证与授权
2. 令牌的生成与发放
3. 令牌的校验与更新

所以我们以下的操作都会围绕 `客户端` 与 `令牌` 来完成。

{% note warning %}
注意：以下授权服务器全默认在 8000 端口运行！！！
{% endnote %}

现在我们需要进行的就是授权服务器配置实现，我们完成项目的初始化，和之前创建完全一样，创建完成后，**我们把 8080 端口修改为 8000 端口**，然后项目结构如下：

![](/images/Spring-Security-Oauth2-从零到一完整实践（三）授权服务器/1.png)

同时添加一下如下依赖

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

既然是授权服务器，那么我们也就不用把它注册为资源服务器了，因为我们不对外暴露任何资源，仅仅只是为了令牌的下发，不需要做资源保护。

在我们配置授权服务器之前，需要先进行我们前面遇到过的配置 spring security web 安全，复制一下上一次的配置，就不截图了，如下：

```java
package isee.wang.oauth.authorization.config;

import org.springframework.context.annotation.Bean;
import org.springframework.security.config.annotation.web.configuration.EnableWebSecurity;
import org.springframework.security.config.annotation.web.configuration.WebSecurityConfigurerAdapter;
import org.springframework.security.core.userdetails.User;
import org.springframework.security.core.userdetails.UserDetailsService;
import org.springframework.security.crypto.bcrypt.BCryptPasswordEncoder;
import org.springframework.security.crypto.password.PasswordEncoder;
import org.springframework.security.provisioning.InMemoryUserDetailsManager;

/**
 * @author <a href="http://isee.wang">isee.wang</a>
 * @date 2021/4/20 10:25
 */
@EnableWebSecurity
public class SecurityConfig extends WebSecurityConfigurerAdapter {

    /**
     * 密码加密方式，spring 5 后必须对密码进行加密
     *
     * @return BCryptPasswordEncoder
     */
    @Bean
    public PasswordEncoder passwordEncoder() {
        return new BCryptPasswordEncoder();
    }

    /**
     * 创建两个内存用户
     * 用户名 user 密码 123456 角色 ROLE_USER
     * 用户名 admin 密码 admin 角色 ROLE_ADMIN
     *
     * @return InMemoryUserDetailsManager
     */
    @Bean
    @Override
    public UserDetailsService userDetailsService() {
        InMemoryUserDetailsManager manager = new InMemoryUserDetailsManager();
        manager.createUser(User.withUsername("user")
                .password(passwordEncoder().encode("123456"))
                .authorities("ROLE_USER").build());
        manager.createUser(User.withUsername("admin")
                .password(passwordEncoder().encode("admin"))
                .authorities("ROLE_ADMIN").build());
        return manager;
    }

}
```

上一部分我们知道 spring-security-oauth2-autoconfigure 是自动配置的包，通过陪配置文件就可以完成一个授权服务器和资源服务器，现在我们需要来自定义他的授权服务器该怎么做呢？我们需要做的就是配置属于我们自己的 AuthorizationServerConfigurer了，当 spring 扫描到我们实现的配置以后，他就不回去自动配置 oauth2 了。为什么这么说呢？可以通过查看他的自动配置的源码你就会发现为什么，如下：

![](/images/Spring-Security-Oauth2-从零到一完整实践（三）授权服务器/2.png)

所以，如果我们配置了 AuthorizationServerConfigurer 的bean，它是不会执行自动配置的。我们现在需要自定义，所以就要来实现一下这个接口。当然，spring 提供了相应的适配器来供我们实现这个接口的，他就是 AuthorizationServerConfigurerAdapter，我们只要继承这个类即可。我们来看看里面的三个配置方法：

| 方法名 | 参数 | 描述 |
| - | - | - |
| configure | AuthorizationServerSecurityConfigurer | 配置授权服务器的安全信息，比如 ssl 配置、checktoken 是否允许访问，是否允许客户端的表单身份验证等。 |
| configure | ClientDetailsServiceConfigurer | 配置客户端的 service，也就是应用怎么获取到客户端的信息，一般来说是从内存或者数据库中获取，已经提供了他们的默认实现，你也可以自定义。 |
| configure | AuthorizationServerEndpointsConfigurer | 配置授权服务器各个端点的非安全功能，如令牌存储，令牌自定义，用户批准和授权类型。如果需要密码授权模式，需要提供 AuthenticationManager 的 bean。 |

所以为了方便，我们先在我们的 SecurityConfig 配置中创建一个 AuthenticationManager Bean，直接调用父类的方法获取即可，如下：

```java
/**
 * 认证管理
 *
 * @return 认证管理对象
 * @throws Exception 认证异常信息
 */
@Override
@Bean  // 重点是这行，父类并没有将它注册为一个 Bean
public AuthenticationManager authenticationManagerBean() throws Exception {
    return super.authenticationManagerBean();
}
```

接下来就是我们配置我们自己的授权服务器了，我们要完成如下的几种授权服务器配置

- 基于内存的客户端信息与令牌存储
- 基于 mysql 的客户端信息与令牌存储
- 基于 redis 的令牌存储
- 基于 jwt 的令牌生成与配置
- 授权服务器小扩展

{% note info no-icon %}
**以上可以自由组合，例如 mysql 客户端配合 redis 令牌存储等。**

**由于内容过多，防止由于依赖的问题导致不好运行查看效果，我每一种方式，都将它放在新的模块之中，模块的创建将会省略不写。分别为 内存、mysql、redis、jwt 四个模块**
{% endnote %}

不过在那之前，我们需要准备一个已经继承 AuthorizationServerConfigurerAdapter 的配置类，同时上面提到过，如果需要密码模式，我们要提供 AuthenticationManager 的 bean，所以我们在这里提前进行配置下，后面就不再进行赘述，如下：

```java
@Configuration
@RequiredArgsConstructor
@EnableAuthorizationServer
public class Oauth2AuthorizationServerConfig
    extends AuthorizationServerConfigurerAdapter {
  
    private final @NonNull AuthenticationManager authenticationManager;

    @Override
    public void configure(AuthorizationServerEndpointsConfigurer endpoints) throws Exception {
        endpoints.authenticationManager(this.authenticationManager);
    }
}
```

现在的项目结构如下：

![](/images/Spring-Security-Oauth2-从零到一完整实践（三）授权服务器/3.png)

{% note warning %}
注意，为了方便，后面的测试均使用密码模式进行测试！
{% endnote %}

## 基于内存的客户端信息与令牌存储

{% note default no-icon %}
代码参见项目模块 spring-security-oauth2-authorization
{% endnote %}

我们将在内存中存储和读取客户端信息以及下发的令牌信息：

- **优点**：速度快，读取速度和写入速度都很快，配置也极其方便。
- **缺点**：扩展性差，需要在代码中配置，重启应用后已经下发的令牌失效。
- **适用场景**：小型不易改变的应用，授权服务器和资源服务器一体的应用。

### 客户端信息

对于客户端信息的配置，你完全可以通过 `OAuth2AuthorizationServerConfiguration` 这个类学习到，对于客户端的配置我们主要实现对参数为 `ClientDetailsServiceConfigurer` 的方法配置，我们分来两个方式来学习：

1. 直接代码写死配置客户端信息
2. 读取配置文件中的客户端信息

#### 代码配置

我们需要以下几步完成配置

1. 构建内存存储的 ClientDetailsService 实现类（spring security oauth 已经提供）。
2. 利用构建出来的进行配置客户端。

所以我们先进行第一步，我们获取他的建造者：

```java
InMemoryClientDetailsServiceBuilder builder = clients.inMemory();
```

然后通过他构建一个内存客户端：

```java
builder
		// 构建一个 id 为 oauth2 的客户端
        .withClient("oauth2")
        // 设置她的密钥，加密后的
        .secret("$2a$10$wlgcx61faSJ8O5I4nLiovO9T36HBQgh4RhOQAYNORCzvANlInVlw2")
        // 设置允许访问的资源 id
        .resourceIds("oauth2")
        // 授权的类型
        .authorizedGrantTypes("password", "authorization_code", "refresh_token")
        // 可以授权的角色
        .authorities("ROLE_ADMIN", "ROLE_USER")
        // 授权的范围
        .scopes("all")
        // token 有效期
        .accessTokenValiditySeconds(Math.toIntExact(Duration.ofHours(1).getSeconds()))
        // 刷新 token 的有效期
        .refreshTokenValiditySeconds(Math.toIntExact(Duration.ofHours(1).getSeconds()))
        // 授权码模式的重定向地址
        .redirectUris("http://example.com");
```

看起来她配置的东西和我们在配置文件中写的东西是基本一致的，不过密码现在是加密后的了，如何获取呢？我是写了一个测试类如下：

```java
package isee.wang.oauth.authorization;

import org.junit.Test;
import org.springframework.security.crypto.bcrypt.BCryptPasswordEncoder;

/**
 * 获取加密后的密码
 *
 * @author <a href="http://isee.wang">isee.wang</a>
 * @date 2021/4/27 17:20
 */
public class PasswordTest {

    @Test
    public void password() {
        // 每次打印的结果都不一样，不影响
        System.out.println(new BCryptPasswordEncoder().encode("oauth2"));
    }
}
```

然后将打印的密码填入即可，**不过值得注意的是，她每次的加密结果都是不一样的。**现在的文件如下：

![](/images/Spring-Security-Oauth2-从零到一完整实践（三）授权服务器/4.png)

我们启动然后测试一下：

![](/images/Spring-Security-Oauth2-从零到一完整实践（三）授权服务器/5.png)

这个就从内存中存存储和读取客户端信息了，如果多个客户端呢？复制一遍就好啦

```java
    @Override
    public void configure(ClientDetailsServiceConfigurer clients) throws Exception {
        InMemoryClientDetailsServiceBuilder builder = clients.inMemory();
        builder
                // 构建一个 id 为 oauth2 的客户端
                .withClient("oauth2")
                // 设置她的密钥，加密后的
                .secret("$2a$10$8o7CwWaaMVvbUdnMwYs6IeWsUogIznSeWuP60zPiZHlY3OyCWywZC")
                // 设置允许访问的资源 id
                .resourceIds("oauth2")
                // 授权的类型
                .authorizedGrantTypes("password", "authorization_code", "refresh_token")
                // 可以授权的角色
                .authorities("ROLE_ADMIN", "ROLE_USER")
                // 授权的范围
                .scopes("all")
                // token 有效期
                .accessTokenValiditySeconds(Math.toIntExact(Duration.ofHours(1).getSeconds()))
                // 刷新 token 的有效期
                .refreshTokenValiditySeconds(Math.toIntExact(Duration.ofHours(1).getSeconds()))
                // 授权码模式的重定向地址
                .redirectUris("http://example.com");
        builder
                // 构建一个 id 为 test 的客户端
                .withClient("test")
                // 设置她的密钥，加密后的
                .secret("$2a$10$8o7CwWaaMVvbUdnMwYs6IeWsUogIznSeWuP60zPiZHlY3OyCWywZC")
                // 设置允许访问的资源 id
                .resourceIds("oauth2")
                // 授权的类型
                .authorizedGrantTypes("password", "authorization_code", "refresh_token")
                // 可以授权的角色
                .authorities("ROLE_ADMIN", "ROLE_USER")
                // 授权的范围
                .scopes("all")
                // token 有效期
                .accessTokenValiditySeconds(Math.toIntExact(Duration.ofHours(1).getSeconds()))
                // 刷新 token 的有效期
                .refreshTokenValiditySeconds(Math.toIntExact(Duration.ofHours(1).getSeconds()))
                // 授权码模式的重定向地址
                .redirectUris("http://example.com");
    }
```

亦或者完全使用链式结构如下：

```java
    @Override
    public void configure(ClientDetailsServiceConfigurer clients) throws Exception {
        clients.inMemory()
                .withClient("oauth2")
                .secret("$2a$10$8o7CwWaaMVvbUdnMwYs6IeWsUogIznSeWuP60zPiZHlY3OyCWywZC")
                .resourceIds("oauth2")
                .authorizedGrantTypes("password", "authorization_code", "refresh_token")
                .authorities("ROLE_ADMIN", "ROLE_USER")
                .scopes("all")
                .accessTokenValiditySeconds(Math.toIntExact(Duration.ofHours(1).getSeconds()))
                .refreshTokenValiditySeconds(Math.toIntExact(Duration.ofHours(1).getSeconds()))
                .redirectUris("http://example.com")
                .and()
                .withClient("test")
                .secret("$2a$10$8o7CwWaaMVvbUdnMwYs6IeWsUogIznSeWuP60zPiZHlY3OyCWywZC")
                .resourceIds("oauth2")
                .authorizedGrantTypes("password", "authorization_code", "refresh_token")
                .authorities("ROLE_ADMIN", "ROLE_USER")
                .scopes("all")
                .accessTokenValiditySeconds(Math.toIntExact(Duration.ofHours(1).getSeconds()))
                .refreshTokenValiditySeconds(Math.toIntExact(Duration.ofHours(1).getSeconds()))
                .redirectUris("http://example.com");
    }
```

#### 配置文件配置

对于配置文件配置其实他已经有了默认的实现了，但是只能对一个客户端进行配置，我们需要多个的时候怎么办呢？就需要我们来扩展了，这个实现其实很简单，就是一个配置类和一个循环的实现，我们来捋一下步骤。

1. 读取配置文件，多个客户端信息
2. 逐个配置客户端信息

先来书写配置类，使用 lombok 自动生成 get/set 等方法：

```java
@Data
@Configuration
@ConfigurationProperties("application.security.oauth")
public class ClientDetails {
    private List<BaseClientDetails> client;
}
```

书写配置文件：

```yaml
application:
  security:
    oauth:
      client[0]:
        registered-redirect-uri: http://example.com
        # 客户端 id
        client-id: client1
        # 客户端密钥
        client-secret: $2a$10$8o7CwWaaMVvbUdnMwYs6IeWsUogIznSeWuP60zPiZHlY3OyCWywZC
        # 授权范围
        scope: all
        # token 有效期
        access-token-validity-seconds: 6000
        # 刷新 token 的有效期
        refresh-token-validity-seconds: 6000
        # 允许的授权类型
        grant-type: authorization_code,password,refresh_token
        # 可以访问的资源 id
        resource-ids: oauth2
      client[1]:
        registered-redirect-uri: http://example.com
        # 客户端 id
        client-id: client2
        # 客户端密钥
        client-secret: $2a$10$8o7CwWaaMVvbUdnMwYs6IeWsUogIznSeWuP60zPiZHlY3OyCWywZC
        # 授权范围
        scope: all
        # token 有效期
        access-token-validity-seconds: 6000
        # 刷新 token 的有效期
        refresh-token-validity-seconds: 6000
        # 允许的授权类型
        grant-type: authorization_code,password,refresh_token
        # 可以访问的资源 id
        resource-ids: oauth2
```

为了防止混淆，我单独写了一个方法来配置，如下：

```java
    private void configClient(ClientDetailsServiceConfigurer clients) throws Exception {
        InMemoryClientDetailsServiceBuilder builder = clients.inMemory();
        for (BaseClientDetails client : clientDetails.getClient()) {
            ClientDetailsServiceBuilder<InMemoryClientDetailsServiceBuilder>.ClientBuilder clientBuilder =
                    builder.withClient(client.getClientId());
            clientBuilder
                    .secret(client.getClientSecret())
                    .resourceIds(client.getResourceIds().toArray(new String[0]))
                    .authorizedGrantTypes(client.getAuthorizedGrantTypes().toArray(new String[0]))
                    .authorities(
                            AuthorityUtils.authorityListToSet(client.getAuthorities())
                                    .toArray(new String[0]))
                    .scopes(client.getScope().toArray(new String[0]));
            if (client.getAutoApproveScopes() != null) {
                clientBuilder.autoApprove(
                        client.getAutoApproveScopes().toArray(new String[0]));
            }
            if (client.getAccessTokenValiditySeconds() != null) {
                clientBuilder.accessTokenValiditySeconds(
                        client.getAccessTokenValiditySeconds());
            }
            if (client.getRefreshTokenValiditySeconds() != null) {
                clientBuilder.refreshTokenValiditySeconds(
                        client.getRefreshTokenValiditySeconds());
            }
            if (client.getRegisteredRedirectUri() != null) {
                clientBuilder.redirectUris(
                        client.getRegisteredRedirectUri().toArray(new String[0]));
            }
        }
    }
```

最终如下：

![](/images/Spring-Security-Oauth2-从零到一完整实践（三）授权服务器/6.png)

然后运行测试一下两个客户端

![](/images/Spring-Security-Oauth2-从零到一完整实践（三）授权服务器/7.png)
![](/images/Spring-Security-Oauth2-从零到一完整实践（三）授权服务器/8.png)

这样也实现了效果

### 令牌存储

其实他默认的令牌存储就是使用到内存存储，所以我们无需配置～何以见得呢？我们来简单分析一下。

在前面我们说过 `AuthorizationServerConfigurer` 的三个配置方法，其中就有一个参数为 `AuthorizationServerEndpointsConfigurer` 类型的配置方法，它可以配置我们令牌信息，所以我们就要把目标放在他的上面看看，去找一找他是如何配置的。

他的核心配置类是 `AuthorizationServerEndpointsConfiguration`，这个类内容很多，我们只关注他是默认配置的为什么是内存的，首先找到一个工厂类：

![](/images/Spring-Security-Oauth2-从零到一完整实践（三）授权服务器/9.png)

我们跟进去看看：

![](/images/Spring-Security-Oauth2-从零到一完整实践（三）授权服务器/10.png)
![](/images/Spring-Security-Oauth2-从零到一完整实践（三）授权服务器/11.png)
![](/images/Spring-Security-Oauth2-从零到一完整实践（三）授权服务器/12.png)

这样我们就找到她是如何默认创建的了。

## 基于 mysql 的客户端信息与令牌存储

{% note default no-icon %}
代码参见项目模块 spring-security-oauth2-authorization-mysql

模块创建步骤省略
{% endnote %}

我们将在 mysql 中存储和读取客户端信息以及下发的令牌信息：

- **优点**：扩展性极高，不用修改代码与重启就可以完成客户端管理，安全性高。
- **缺点**：使用数据库速度过慢，多客户端高并发情况下可能会造成性能瓶颈
- **适用场景**：中大型项目，独立且完整的授权服务器。

在这之前你要添加如下的 mysql 和 jdbc 依赖

```xml
<dependency>
    <groupId>mysql</groupId>
    <artifactId>mysql-connector-java</artifactId>
</dependency>
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-jdbc</artifactId>
</dependency>
```

配置文件如下，我的 mysql 版本为 5.7 ，url 参数请自行修改

```yaml
server:
  port: 8000
spring:
  datasource:
    driver-class-name: com.mysql.jdbc.Driver
    url: jdbc:mysql://127.0.0.1:3306/auth?useUnicode=true&characterEncoding=UTF-8&useOldAliasMetadataBehavior=true&autoReconnect=true&serverTimezone=UTC
    username: root
    password: 123456
    # 用来初始化数据库的，如果不存在表就自动创建
    initialization-mode: ALWAYS
    schema: classpath:ddl.sql
```

导入 [官方提供](https://github.com/spring-projects/spring-security-oauth/blob/master/spring-security-oauth2/src/test/resources/schema.sql) 的 h2 的表，由于官方使用的是 h2 的数据库，有些字段类型不对，我修改成 mysql 的后如下：

```mysql
-- used in tests that use MYSQL
create table if not exists oauth_client_details (
  client_id VARCHAR(256) PRIMARY KEY,
  resource_ids VARCHAR(256),
  client_secret VARCHAR(256),
  scope VARCHAR(256),
  authorized_grant_types VARCHAR(256),
  web_server_redirect_uri VARCHAR(256),
  authorities VARCHAR(256),
  access_token_validity INTEGER,
  refresh_token_validity INTEGER,
  additional_information VARCHAR(4096),
  autoapprove VARCHAR(256)
) ENGINE=InnoDB DEFAULT CHARSET=utf8 ROW_FORMAT=DYNAMIC;

create table if not exists oauth_client_token (
  token_id VARCHAR(256),
  token BLOB,
  authentication_id VARCHAR(256) PRIMARY KEY,
  user_name VARCHAR(256),
  client_id VARCHAR(256)
) ENGINE=InnoDB DEFAULT CHARSET=utf8 ROW_FORMAT=DYNAMIC;

create table if not exists oauth_access_token (
  token_id VARCHAR(256),
  token BLOB,
  authentication_id VARCHAR(256) PRIMARY KEY,
  user_name VARCHAR(256),
  client_id VARCHAR(256),
  authentication BLOB,
  refresh_token VARCHAR(256)
) ENGINE=InnoDB DEFAULT CHARSET=utf8 ROW_FORMAT=DYNAMIC;

create table if not exists oauth_refresh_token (
  token_id VARCHAR(256),
  token BLOB,
  authentication BLOB
) ENGINE=InnoDB DEFAULT CHARSET=utf8 ROW_FORMAT=DYNAMIC;

create table if not exists oauth_code (
  code VARCHAR(256), authentication BLOB
) ENGINE=InnoDB DEFAULT CHARSET=utf8 ROW_FORMAT=DYNAMIC;

create table if not exists oauth_approvals (
	userId VARCHAR(256),
	clientId VARCHAR(256),
	scope VARCHAR(256),
	status VARCHAR(10),
	expiresAt TIMESTAMP,
	lastModifiedAt TIMESTAMP
) ENGINE=InnoDB DEFAULT CHARSET=utf8 ROW_FORMAT=DYNAMIC;


-- customized oauth_client_details table
create table if not exists ClientDetails (
  appId VARCHAR(256) PRIMARY KEY,
  resourceIds VARCHAR(256),
  appSecret VARCHAR(256),
  scope VARCHAR(256),
  grantTypes VARCHAR(256),
  redirectUrl VARCHAR(256),
  authorities VARCHAR(256),
  access_token_validity INTEGER,
  refresh_token_validity INTEGER,
  additionalInformation VARCHAR(4096),
  autoApproveScopes VARCHAR(256)
) ENGINE=InnoDB DEFAULT CHARSET=utf8 ROW_FORMAT=DYNAMIC;
```

先给大家介绍一下几张表的具体意思和结构：

oauth_client_details ===> 客户端信息

| 列名 | 类型 | 描述 |
| - | - | - |
| client_id（主键） | VARCHAR(256) | 主键,必须唯一,不能为空. 用于唯一标识每一个客户端(client); 在注册时必须填写(也可由服务端自动生成). 对于不同的grant_type,该字段都是必须的. 在实际应用中的另一个名称叫appKey,与client_id是同一个概念. |
| resource_ids | VARCHAR(256) | 客户端所能访问的资源id集合,多个资源时用逗号(,)分隔 |
| client_secret | VARCHAR(256) | 用于指定客户端(client)的访问密匙; 在注册时必须填写(也可由服务端自动生成). 对于不同的grant_type,该字段都是必须的. 在实际应用中的另一个名称叫appSecret,与client_secret是同一个概念. |
| scope | VARCHAR(256) | 指定客户端申请的权限范围,可选值包括read,write,trust;若有多个权限范围用逗号(,)分隔,如: “read,write”. |
| authorized_grant_types | VARCHAR(256) | 指定客户端支持的grant_type,可选值包括authorization_code,password,refresh_token,implicit,client_credentials,若支持多个grant_type用逗号(,)分隔,如: “authorization_code,password”.在实际应用中,当注册时,该字段是一般由服务器端指定的,而不是由申请者去选择的, |
| web_server_redirect_uri | VARCHAR(256) | 客户端的重定向URI,可为空, 当grant_type为authorization_code或implicit时, 在Oauth的流程中会使用并检查与注册时填写的redirect_uri是否一致. |
| authorities | VARCHAR(256) | 指定客户端所拥有的Spring Security的权限值,可选, 若有多个权限值,用逗号(,)分隔, 如: "ROLE_ADMIN" |
| access_token_validity | INTEGER | 设定客户端的access_token的有效时间值(单位:秒),可选, 若不设定值则使用默认的有效时间值(60 * 60 * 12, 12小时). |
| refresh_token_validity | INTEGER | 设定客户端的refresh_token的有效时间值(单位:秒),可选, 若不设定值则使用默认的有效时间值(60 * 60 * 12, 12小时). |
| additional_information | VARCHAR(4096) | 这是一个预留的字段,在Oauth的流程中没有实际的使用,可选,但若设置值,必须是JSON格式的数据,在实际应用中, 可以用该字段来存储关于客户端的一些其他信息 |
| autoapprove | VARCHAR(256) | 设置用户是否自动Approval操作, 默认值为 ‘false’, 可选值包括 ‘true’,‘false’, ‘read’,‘write’.该字段只适用于grant_type="authorization_code"的情况,当用户登录成功后,若该值为’true’或支持的scope值,则会跳过用户Approve的页面,直接授权. |

oauth_client_token ===> 客户端系统中存储从服务端获取的 token 数据

| 字段名 | 字段类型 | 描述 |
| - | - | - |
| token_id | VARCHAR(256) | 从服务器端获取到的access_token的值. |
| token | BLOB | 这是一个二进制的字段, 存储的数据是OAuth2AccessToken.java对象序列化后的二进制数据. |
| authentication_id | VARCHAR(256) | 该字段具有唯一性, 是根据当前的username(如果有),client_id与scope通过MD5加密生成的. 具体实现请参考DefaultClientKeyGenerator.java类. |
| user_name | VARCHAR(256) | 登录时的用户名 |
| client_id | VARCHAR(256) | 客户端 id |

oauth_access_token ===> 生成的 token 数据

| 字段名 | 字段类型 | 描述 |
| - | - | - |
| token_id | VARCHAR(256) | 从服务器端获取到的access_token的值. |
| token | BLOB | 存储将OAuth2AccessToken.java对象序列化后的二进制数据, 是真实的AccessToken的数据值. |
| authentication_id | VARCHAR(256) | 该字段具有唯一性, 其值是根据当前的username(如果有),client_id与scope通过MD5加密生成的. |
| user_name | VARCHAR(256) | 登录时的用户名, 若客户端没有用户名(如grant_type=“client_credentials”),则该值等于client_id |
| client_id | VARCHAR(256) | 客户端 id |
| authentication | BLOB | 存储将 OAuth2Authentication 对象序列化后的二进制数据. |
| refresh_token | VARCHAR(256) | 该字段的值是将refresh_token的值通过MD5加密后存储的. |

oauth_refresh_token ===> 刷新 token

| 字段名 | 字段类型 | 描述 |
| - | - | - |
| token_id | VARCHAR(256) | 该字段的值是将refresh_token的值通过MD5加密后存储的. |
| token | BLOB | 存储将OAuth2RefreshToken.java对象序列化后的二进制数据. |
| authentication | BLOB | 存储将OAuth2Authentication.java对象序列化后的二进制数据. |

oauth_code ===> 服务端生成的 code 值

| 字段名 | 字段类型 | 描述 |
| - | - | - |
| code | VARCHAR(256) | 存储服务端系统生成的code的值(未加密). |

oauth_approvals ===> 授权同意信息

| 字段名 | 字段类型 | 描述 |
| - | - | - |
| userId | VARCHAR(256) | 用户 id |
| clientId | VARCHAR(256) | 客户端 id |
| scope | VARCHAR(256) | 请求的范围 |
| status | VARCHAR(10) | 授权的状态 |
| expiresAt | TIMESTAMP | 时间 |
| lastModifiedAt | TIMESTAMP | 最后修改的时间 |

最后一张 ClientDetails 是我们要自定义他的 表 的情况，在我们需要自定义的时候使用，但是目前我们暂时不去自定义，所以无用。

所以你现在的项目结构应该如下

![](/images/Spring-Security-Oauth2-从零到一完整实践（三）授权服务器/13.png)

记得启动测试一下，确定不报错。

接下来我们就是来进行配置了，同样的，分为客户端信息配置和令牌配置

### 客户端信息

同样，对于客户端的配置我们主要实现对参数为 ClientDetailsServiceConfigurer 的方法进行配置，我们需要完成以下两步：

1. 构建一个 jdbc 的 ClientDetailsService，通过他来链接数据库。
2. 将它配置进 ClientDetailsServiceConfigurer 之中。

我们首先先来配置一个 jdbc 的 ClientDetailsService ，非常简单，因为他已经提供了默认的实现了的，构建方式如下：

```java
// 数据源
private final @NonNull DataSource dataSource;

/**
 * 声明 ClientDetails实现
 *
 * @return ClientDetailsService
 */
@Bean
public ClientDetailsService clientDetails() {
    return new JdbcClientDetailsService(dataSource);
}
```

然后将他配置进 ClientDetailsServiceConfigurer 之中，如下：

```java
@Override
public void configure(ClientDetailsServiceConfigurer clients) throws Exception {
    clients.withClientDetails(clientDetails());
}
```
