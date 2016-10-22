# Spring-security-demo

---

## 最小配置

> 先对spring-security有个直观了解

### web.xml

> 添加spring-security filter  

``` xml
  <!-- 读取配置文件 -->
  <context-param>
    <param-name>contextConfigLocation</param-name>
    <param-value>classpath:conf/spring*.xml</param-value>
  </context-param>


  <!--spring-security-->
  <filter>
    <filter-name>springSecurityFilterChain</filter-name>
    <filter-class>org.springframework.web.filter.DelegatingFilterProxy</filter-class>
  </filter>

  <filter-mapping>
    <filter-name>springSecurityFilterChain</filter-name>
    <url-pattern>/*</url-pattern>
  </filter-mapping>

  <listener>
    <listener-class>org.springframework.web.context.ContextLoaderListener
    </listener-class>
  </listener>
```

### spring-security.xml

> 这里使用security为默认命名空间，使用自己的登录界面。username-parameter和password-parameter默认为j_username和j_password，这里改为username& 和password。在login.html中form中有所变化，action默认为j_spring_security_check

``` xml
<?xml version="1.0" encoding="UTF-8"?>

<beans:beans xmlns="http://www.springframework.org/schema/security"
             xmlns:beans="http://www.springframework.org/schema/beans"
             xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
             xsi:schemaLocation="http://www.springframework.org/schema/beans
		http://www.springframework.org/schema/beans/spring-beans-3.0.xsd
		http://www.springframework.org/schema/security
		http://www.springframework.org/schema/security/spring-security.xsd">

    <http auto-config="true">
        <intercept-url pattern="index.html" access="permitAll"/>
        <intercept-url pattern="login.html" access="permitAll" />
        <intercept-url pattern="/pages/*" access="hasRole('ROLE_USER')" />

        <form-login
                login-page="/login.html"
                default-target-url="/pages/welcome.html"
                authentication-failure-url="/login.html?error"
                login-processing-url="/j_spring_security_check"
                username-parameter="username"
                password-parameter="password" />

        <!-- logout-success-url:成功注销后跳转到的页面; -->
        <logout
                logout-url="/login"
                logout-success-url="/login.html"
                invalidate-session="true" />
                
        <!-- enable csrf protection -->
        <csrf disabled="true" />
    </http>

    <authentication-manager>
        <authentication-provider>
            <user-service>
                <user name="admin" password="1234" authorities="ROLE_USER" />
            </user-service>
        </authentication-provider>
    </authentication-manager>

</beans:beans>
```

### login.html

> 这里username和password已经是spring-security.xml中修改过后的

``` html
<form action="login" method="post">
    <input type="text" name="username">
    <input type="password" name="password">
    <input type="submit" value="Login">
</form>
```

## 相关说明

> * Authentication接口：表示用于认证信息。用户登录以前输入等相关信息封装成一个Authentication实现的对象。认证成功后生成信息更全面的Authentication对象。
> * SecurityContext：包含当前正在访问的所有用户信息。
> * SecurityContextHolder：使用ThreadLocal保存SecurityContext。
> * AuthenticationManager：验证Authentication的接口
> * ProviderManager：AuthenticationManager接口默认实现。将注册DaoAuthenticationProvider，委托AuthenticationProvider列表来验证Authentication。验证方式一般为根据username加载UserDetails，对于UserDetails密码与输入是否一致。
> * AuthenticationProvider：默认创建DaoAuthenticationProvider，内部使用UserDetailsService加载UserDetails。需要使用自己的认证方式时，可以使用自己实现的AuthenticationProvider；如果需要改变认证的用户信息来源，可以实现自己的UserDetailsService（如下都使用默认空间为security）。

``` xml
<authentication-manager>          
    <authentication-provider ref="myAuthenticationProvider"/>
</authentication-manager>

<authentication-manager>
    <authentication-provider user-service-ref="myUserDetailsService"/>
</authentication-manager>
```

> 一般用于数据都存在数据库中，用户信息通过UserDetailsService获取，需要配置数据源，下节是简单数据库的配置。


## 简单数据库配置

### spring-security.xml

>  这里只修改了数据源，而且没有对password加密。dataSource已经在spring配置文件中声明。

``` xml
   <http auto-config="true">
        <intercept-url pattern="index.html" access="permitAll"/>
        <intercept-url pattern="login.html" access="permitAll" />
        <intercept-url pattern="/pages/*" access="hasRole('ROLE_USER')" />

        <form-login
                login-page="/login.html"
                default-target-url="/pages/welcome.html"
                authentication-failure-url="/login.html?error"
                login-processing-url="/login"
                username-parameter="username"
                password-parameter="password" />

        <!-- logout-success-url:成功注销后跳转到的页面; -->
        <logout
                logout-url="/j_spring_security_logout"
                logout-success-url="/login.html"
                invalidate-session="true" />
                
        <!-- enable csrf protection -->
        <csrf disabled="true" />
    </http>

    <!--不使用数据库-->
<!--    <authentication-manager>
        <authentication-provider>
            <user-service>
                <user name="admin" password="1234" authorities="ROLE_USER" />
            </user-service>
        </authentication-provider>
    </authentication-manager>-->

    <!--使用数据库-->
    <authentication-manager alias="authenticationManager" >
        <authentication-provider>
            <jdbc-user-service data-source-ref="dataSource"/>
        </authentication-provider>
    </authentication-manager>
</beans:beans>
```

### 数据库表

> 由于这里使用的默认配置，必须使用以下数据库表。users用户存储所有用户（eg: admin admin 1）,authorities存储权限（eg:admin ROLE_USER）

``` sql
create table users(
	username varchar_ignorecase(50) not null primary key,
	password varchar_ignorecase(50) not null,
	enabled boolean not null
);

create table authorities (
	username varchar_ignorecase(50) not null,
	authority varchar_ignorecase(50) not null,
	constraint fk_authorities_users foreign key(username) references users(username)
);
create unique index ix_auth_username on authorities (username,authority);
```

> 默认情况下，jdbc-user-service将使用SQL语句“select username, password, enabled from users where username = ?”来获取用户信息；使用SQL语句“select username, authority from authorities where username = ?”来获取用户对应的权限。如果使用用户表为t_user，使用以下配置

``` xml
<authentication-manager>
    <authentication-provider>
        <jdbc-user-service data-source-ref="dataSource" users-by-username-query="select username,password, enabled from t_user where username = ?" />
    </authentication-provider>
</authentication-manager>
```

| 属性名| 说明|
|---|---|
|users-by-username-query|指定查询用户信息的SQL|
|authorities-by-username-query|指定查询用户权限的SQL|
|group-authorities-by-username-query|指定查询用户组权限的SQL|



