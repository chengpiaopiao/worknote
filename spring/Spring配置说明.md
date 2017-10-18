

## web.xml 


### Spring 配置文件拆分

在项目中，通常会将Spring的配置文件拆成几个，每一个对应不同的功能配置。在这里我们将Spring的配置文件拆分成 spring-config.xml 和 spring-servlet.xml。

- spring-servlet.xml 主要负责servlet相关的配置项。
- spring-config.xml 主要负责其他配置项。

### Spring 配置文件中属性值分离

PropertyPlaceholderConfigurer是个bean工厂后置处理器的实现，也就是 BeanFactoryPostProcessor接口的一个实现。PropertyPlaceholderConfigurer可以将上下文（配置文件）中的属性值放在另一个单独的标准java Properties文件中去，这里分三步说明：

1. 添加properties文件 resource.properties
```
connection.driver_class=com.mysql.jdbc.Driver
connection.url=jdbc\:mysql\://localhost\:3306/adserver?useUnicode\=true&characterEncoding\=utf-8&autoReconnect\=true
connection.username=root
connection.password=root
```
2. 在spring-config.xml文件中配置：
```
<bean id="propertyConfigurer"
	class="org.springframework.beans.factory.config.PropertyPlaceholderConfigurer">
	<property name="locations">
		<list>
			<value>classpath:resources.properties</value>
		</list>
	</property>
</bean>
```
3. 添加数据源，也是在spring-config.xml中添加：
```
<bean id="dataSource" class="org.logicalcobwebs.proxool.ProxoolDataSource">
	<property name="alias" value="proxoolDataSource" />
	<property name="driver" value="${connection.driver_class}" />
	<property name="driverUrl" value="${connection.url}" />
	<property name="user" value="${connection.username}" />
	<property name="password" value="${connection.password}" />
	<!-- 略 -->
</bean>
```

### 扫描注解

当前项目的包结构如下：
```
com.sumavision.ad
            ├─admgr
            │  ├─dao
            │  ├─model
            │  ├─service
            │  └─web.controller
            ├─framework
            │  ├─common
            │  ├─dao
            │  ├─model
            │  ├─service
            │  └─util
            ├─resourcemgr
            │  ├─dao
            │  ├─model
            │  ├─service
            │  └─web.controller
            ├─systemerror
            └─user
                ├─dao
                ├─model
                ├─service
                └─web
                    ├─controller
                    └─filter
```

#### spring-servlet.xml 中开启@Controller注解扫描
```
<!-- 开启controller注解支持 -->
<context:component-scan base-package="com.sumavision.ad.*.web.controller">
	<context:include-filter type="annotation"
		expression="org.springframework.stereotype.Controller" />
</context:component-scan>
```
- 要把最终的包写上，而不能写成 base-package="com.sumavision.ad"，因为这种写法对于include-filter来说，它都会扫描，而不仅仅扫描 @Controller.

- <context:component-scan> 在xml配置了这个标签后，spring可以自动去扫描base-package下面或者子包下面的java文件，如果扫描到有@Component @Controller@Service等这些注解的类，则把这些类注册为bean

#### spring-config.xml 中屏蔽@Controller注解扫描
```
<!-- 扫描注解Bean -->
<context:component-scan base-package="com.sumavision.ad">
	<!-- 不扫描Controller注解 -->
	<context:exclude-filter type="annotation"
		expression="org.springframework.stereotype.Controller" />
</context:component-scan>
```
- 扫描com.sumavision.ad包下的所有子类，不包含@Controller
- <context:include-filter> 可以搜索@Controller标签
- <context:exclude-filter> 搜索不到@Controller标签



### Spring validataion

spring-servlet.xml中
```
<!-- 会自动注册了validator ConversionService -->
<mvc:annotation-driven validator="validator"
	conversion-service="conversion-service" />

<!-- 以下 validator ConversionService 在使用 mvc:annotation-driven 会 自动注册 -->
<bean id="validator"
	class="org.springframework.validation.beanvalidation.LocalValidatorFactoryBean">
	<property name="providerClass" value="org.hibernate.validator.HibernateValidator" />
	<!-- 如果不加默认到 使用classpath下的 ValidationMessages.properties -->
	<property name="validationMessageSource" ref="messageSource" />
</bean>
<bean id="conversion-service"
	class="org.springframework.format.support.FormattingConversionServiceFactoryBean" />
```

spring-config.xml中配置国际化资源
```
<!-- 国际化的消息资源文件 -->
<bean id="messageSource"
	class="org.springframework.context.support.ReloadableResourceBundleMessageSource">
	<property name="basenames">
		<list>
			<!-- 在web环境中一定要定位到classpath 否则默认到当前web应用下找 -->
			<value>classpath:messages</value>
		</list>
	</property>
	<property name="defaultEncoding" value="UTF-8" />
	<property name="cacheSeconds" value="60" />
</bean>
```
- validationMessageSource属性：指定国际化错误消息从哪里取，此处使用spring-config.xml中定义的messageSource来获取国际化消息；如果此处不指定该属性，则默认到classpath下的ValidationMessages.properties取国际化错误消息。


- Spring 配置国际化资源有两种方式，参考这里：[spring 配置国际化资源的两种方式](http://blog.csdn.net/home_zhang/article/details/52489901)



### 首页重定向与静态资源映射

spring-servlet.xml中
```
<mvc:view-controller path="/" view-name="forward:/index" />
<!-- 当在web.xml 中 DispatcherServlet使用 <url-pattern>/</url-pattern> 映射时，能映射静态资源 -->
<mvc:default-servlet-handler />
<!-- 静态资源映射 -->
<mvc:resources mapping="/images/**" location="/images/" />
<mvc:resources mapping="/css/**" location="/css/" />
<mvc:resources mapping="/js/**" location="/js/" />
```

### 默认视图解析

```
<bean id="defaultViewResolver"
	class="org.springframework.web.servlet.view.InternalResourceViewResolver">
	<property name="contentType" value="text/html" />
	<property name="prefix" value="/" />
	<property name="suffix" value=".jsp" />
	<property name="viewClass"
		value="org.springframework.web.servlet.view.JstlView" />
</bean>
```


### 文件上传

```
<!-- 文件上传相关 -->
<bean id="multipartResolver"
	class="org.springframework.web.multipart.commons.CommonsMultipartResolver">
	<!--one of the properties available;the maximum file size in bytes -->
	<property name="maxUploadSize" value="2048000000" />
</bean>
```

### hibernate 整合

[整合之道--Spring4整合Hibernate5](http://blog.csdn.net/frankcheng5143/article/details/50634487)

hibernate的配置主要在hibernate.cfg.xml中，而hibernate.cfg.xml文件的主要作用就是配置了一个session-factory

1. 在session-factory中主要通过property配置一些数据库的连接信息，我们知道，Spring通常会将这种数据库连接用dataSource来表示，这样一来，hibernate.cfg.xml文件中的所有跟数据库连接的都可以干掉了，直接用spring的dataSource，而dataSource也可以用c3p0、dbcp等。

2. 在session-factory中通过property除了配置一些数据库的连接信息之外，还有一些hibernate的配置，比如方言、自动创建表机制、格式化sql等，这些信息也需要配置起来。

3. 还有最关键的一个持久化类所在路径的配置


> Spring对hibernate的整合就是将上述三点通过spring配置起来，而hibernate最关键的sessionFactroy就是spring的一个bean

spring-config.xml 中 sessionFactory bean的配置：
```
<bean id="sessionFactory" class="org.springframework.orm.hibernate5.LocalSessionFactoryBean">
	<property name="dataSource" ref="dataSource" />
	<property name="packagesToScan">
		<list>
			<value>com.sumavision.ad</value>
		</list>
	</property>
	<property name="hibernateProperties">
		<props>
			<prop key="hibernate.dialect">${hibernate.dialect}</prop>
			<prop key="hibernate.show_sql">${hibernate.show_sql}</prop>
			<prop key="hibernate.format_sql">${hibernate.format_sql}</prop>
			<prop key="hibernate.use_sql_comments">${hibernate.use_sql_comments}</prop>
			<prop key="hibernate.query.substitutions">${hibernate.query.substitutions}</prop>
			<prop key="hibernate.default_batch_fetch_size">${hibernate.default_batch_fetch_size}</prop>
			<prop key="hibernate.max_fetch_depth">${hibernate.max_fetch_depth}</prop>
			<prop key="hibernate.generate_statistics">${hibernate.generate_statistics}</prop>
			<prop key="hibernate.bytecode.use_reflection_optimizer">${hibernate.bytecode.use_reflection_optimizer}</prop>

			<prop key="hibernate.cache.use_second_level_cache">${hibernate.cache.use_second_level_cache}</prop>
			<prop key="hibernate.cache.use_query_cache">${hibernate.cache.use_query_cache}</prop>
			<prop key="hibernate.cache.region.factory_class">${hibernate.cache.region.factory_class}</prop>
			<prop key="net.sf.ehcache.configurationResourceName">${net.sf.ehcache.configurationResourceName}</prop>
			<prop key="hibernate.cache.use_structured_entries">${hibernate.cache.use_structured_entries}</prop>

			<prop key="hibernate.cache.use_second_level_cache">${hibernate.cache.use_second_level_cache}</prop>
			<prop key="hibernate.jdbc.fetch_size">${hibernate.jdbc.fetch_size}</prop>
			<prop key="hibernate.jdbc.batch_size">${hibernate.jdbc.batch_size}</prop>
			<prop key="hibernate.connection.autocommit">${hibernate.connection.autocommit}</prop>
		</props>
	</property>
</bean>
```
通过bean的配置可以看出该bean就是hibernate的sessionFactroy 
因为它指向了org.springframework.orm.hibernate5.LocalSessionFactoryBean

在这个bean中主要配置了上面说的三点：

1. 数据源dataSource, ``` <property name="dataSource" ref="dataSource" /> ```
2. hibernate的配置，包括方言，输出sql等, ``` <property name="hibernateProperties"> ```
3. 持久化类的位置，通过包进行扫描, ``` <property name="packagesToScan"> ```




