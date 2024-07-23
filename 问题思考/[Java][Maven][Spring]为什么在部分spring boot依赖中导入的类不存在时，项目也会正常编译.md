# 背景
    在各spring boot第三方Starter项目中，都存在一些autoconfiguration项目，其中对于部分自动配置类中，通过 @ConditionalOnClass 验证类是否存在情况，当个人项目中未依赖这些类(或所对应的包)时,项目编译不会报错。而个人项目中随便import一个不存在的类，项目就无法编译。


例如，`org.mybatis.spring.boot:mybatis-spring-boot-start`依赖中的`JdbcRepositoriesAutoConfiguration`这个配置类中，项目中没有`spring-jdbc`这个依赖包，但是编译是成功的，不会有编译错误
```java
package org.springframework.boot.autoconfigure.data.jdbc;

// 不存在的依赖类，即没有导入 spring-data-jdbc相关包
import org.springframework.data.jdbc.repository.config.AbstractJdbcConfiguration;
import org.springframework.data.jdbc.repository.config.EnableJdbcRepositories;
import org.springframework.data.jdbc.repository.config.JdbcRepositoryConfigExtension;

// ... 其它存在的依赖

@AutoConfiguration(after = { JdbcTemplateAutoConfiguration.class, DataSourceTransactionManagerAutoConfiguration.class })
@ConditionalOnBean({ NamedParameterJdbcOperations.class, PlatformTransactionManager.class })
@ConditionalOnClass({ NamedParameterJdbcOperations.class, AbstractJdbcConfiguration.class })
@ConditionalOnProperty(prefix = "spring.data.jdbc.repositories", name = "enabled", havingValue = "true",
		matchIfMissing = true)
public class JdbcRepositoriesAutoConfiguration {

	@Configuration(proxyBeanMethods = false)
	@ConditionalOnMissingBean(JdbcRepositoryConfigExtension.class)
	@Import(JdbcRepositoriesRegistrar.class)
	static class JdbcRepositoriesConfiguration {

	}

	@Configuration(proxyBeanMethods = false)
	@ConditionalOnMissingBean(AbstractJdbcConfiguration.class)
	static class SpringBootJdbcConfiguration extends AbstractJdbcConfiguration {

	}

}
```

# 调查
1. javac编译器在编译时，会检查import的类是否存在，如果不存在，就会报错
2. javac编译时加载依赖jar包，会自动使用，而不会对依赖jar进行重复编译
3. mvn项目打包时，会根据依赖所属时期决定是否将依赖打包进项目包中：如`complier`只能在编译器使用(默认，不会打包进项目包)，`runtime`可以在运行时使用(即需要将依赖打包进项目包中)
4. Jvm类加载器机制，由父层查找，如果父层没有，则由子层加载，如果子层没有，则报错

# 解答
* spring boot 相关项目在编译时所有依赖都是准确存在的，并且部分依赖因为是`complier`，并不会被传递，只有在编译期间使用，故而不会打包进项目
* 本地项目依赖这些springboot项目，编译时只会将依赖的spring boot等jar包直接导入，而不会再次编译依赖的spring boot内容，故而即使本地没有再依赖spring boot所需的jar包，也不会报错
* 在运行时，spring boot项目有自己定义的loader类加载器，springboot根据@ConditionalXXX一系列验证判断是否需要使用这些class，如果依赖不存在，则不会使用，故而即使缺少依赖，启动也不会报错

总结：
* maven打包时会判断依赖是否加入到项目结果包中
* spring boot项目启动时，有自定义的类加载器，根据@ConditionalXXX决定是否加载该类