# 05-SpringBoot-Config

Spring Boot 通过提供合理的默认配置和灵活的覆盖机制，简化了 Spring 应用的配置管理。开发者可以通过 Java 配置类或外部配置文件（如 `application.properties`) 来定义应用程序的行为。

---

关键概要：

- **Spring Boot 配置灵活**：Spring Boot 提供多种配置方式，包括 Java 配置类和外部配置文件（如 `application.properties` 或 `application.yml`），以简化应用程序设置。
- **Java 配置类**：使用 `@Configuration` 或 `@SpringBootConfiguration` 注解定义配置类，替代传统的 XML 配置。
- **配置文件**：支持 `application.properties` 和 `application.yml`，后者因其层次结构更清晰而更受欢迎。
- **配置绑定**：通过 `@ConfigurationProperties` 注解将配置文件中的属性绑定到 Java 对象，支持参数绑定和构造器绑定。
- **外部化配置**：支持多种配置源（如环境变量、命令行参数），具有明确的优先级顺序。
- **高级功能**：包括导入额外配置、随机值配置、多文档配置、Profiles 环境管理、配置加密和配置迁移。
- **注意事项**：在同一位置同时存在 `application.properties` 和 `application.yml` 时，`.properties` 文件优先级更高。

---

## 配置类

Spring Boot 支持使用 `@Configuration` 或 `@SpringBootConfiguration` 注解定义配置类，替代传统的 XML 配置文件。

以下是一个简单的配置类示例：

```java
@Configuration
public class AppConfig {
    @Bean
    public RestTemplate restTemplate() {
        return new RestTemplate();
    }
}
```

`@Configuration` 是 Spring 框架的原生注解，而 `@SpringBootConfiguration` 是 Spring Boot 的专用注解，包含 `@Configuration`，并为 Spring Boot 应用提供额外功能。两者功能等效，推荐在 Spring Boot 中使用 `@Configuration` 以保持通用性。

## 导入配置

Spring Boot 允许通过以下方式导入其他配置：

- **导入 Java 配置类**：使用 `@Import` 注解导入其他配置类，适用于模块化配置管理。

```java
@Configuration
@Import({DataSourceConfig.class, RedisConfig.class})
public class MainConfig {
    // ...
}
```

- **导入 XML 配置**：使用 `@ImportResource` 注解导入传统 XML 配置文件，适用于需要兼容旧系统的情况。

```java
@Configuration
@ImportResource("classpath:legacy-config.xml")
public class MainConfig {
    // ...
}
```

如果配置类不在默认扫描路径下，可以使用 `@ComponentScan` 指定扫描包：

```java
@SpringBootApplication
@ComponentScan(basePackages = {"com.example.config"})
public class Application {
    public static void main(String[] args) {
        SpringApplication.run(Application.class, args);
    }
}
```

## 配置文件

Spring Boot 默认在以下位置查找 `application.properties` 或 `application.yml`：

| 位置 | 说明 | 
| -- | -- |
| `classpath:/` | 类路径根目录 |
| `classpath:/config/` | 类路径 config 子目录 |
| `file:./` | 当前工作目录 |
| `file:./config/` | 当前工作目录 config 子目录 |
| `file:./config/*/` | 当前工作目录 config 子目录的子目录 |

可以通过命令行参数自定义配置文件名称或位置：

```bash
java -jar myapp.jar --spring.config.name=myconfig --spring.config.location=classpath:/custom-config/
```
使用 `spring.config.additional-location` 可追加配置文件，而不覆盖默认位置。如果同一位置存在 `application.properties` 和 `application.yml`，`application.properties` 的属性优先级更高。

> 注意：`application.yml` 因其树状结构更清晰，推荐用于复杂配置，但不支持 `@PropertySource` 注解。

## 配置绑定

Spring Boot 提供 `@ConfigurationProperties` 注解，将配置文件中的属性绑定到 Java 对象，支持参数绑定和构造器绑定。

### 参数绑定

通过 setter 方法将配置文件中的属性绑定到 Java 对象。

配置文件示例：

```java
myapp:
  name: My Application
  version: 1.0.0
  enabled: true
  servers:
    - ip: 127.0.0.1
      path: /path1
  params:
    key: value
```

Java 类示例：

```java
@Component
@ConfigurationProperties(prefix = "myapp")
public class AppProperties {
    private String name;
    private String version;
    private boolean enabled;
    private List<Server> servers;
    private Map<String, String> params;

    public static class Server {
        private String ip;
        private String path;
        // getters and setters
    }
    // getters and setters
}
```

需在启动类上使用 `@EnableConfigurationProperties` 启用：

```java
@SpringBootApplication
@EnableConfigurationProperties(AppProperties.class)
public class Application {
    public static void main(String[] args) {
        SpringApplication.run(Application.class, args);
    }
}
```

### 构造器绑定

Spring Boot 3.0 支持通过构造器绑定属性，适合**不可变对象**。如果类只有一个构造器，无需使用 `@ConstructorBinding` 注解。

```java
@ConfigurationProperties(prefix = "myapp")
public class ImmutableAppProperties {
    private final String name;
    private final String version;

    public ImmutableAppProperties(String name, String version) {
        this.name = name;
        this.version = version;
    }
    // getters
}
```

### Bean 配置绑定

对于无法直接修改的第三方类，可以在 `@Bean` 方法上使用 `@ConfigurationProperties`：

```java
@Configuration
public class BeanConfig {

    @Bean
    @ConfigurationProperties(prefix = "myapp")
    public ThirdPartyClass thirdPartyClass() {
        return new ThirdPartyClass();
    }
}
```

### 配置验证

使用 JSR-303 验证注解（如 `@NotNull`）进行配置验证：

```java
@Component
@Validated
@ConfigurationProperties(prefix = "myapp")
public class ValidatedProperties {

    @NotNull
    private String name;
    // getters and setters
}
```

## 外部化配置与优先级

Spring Boot 支持多种配置源，按以下优先级顺序应用（从低到高）：

| 优先级 | 配置源 |
| -- | -- |
| 1 | 默认属性 |
| 2 | @PropertySource 注解 |
| 3 | 配置文件 |
| 4 | 命令行参数 |
| 5 | Java 系统属性 |
| 6 | 环境变量 |
| 7 | JNDI 参数 |
| 8 | Servlet 参数 |
| 9 | SPRING_APPLICATION_JSON |
| 10 | 测试参数 |
| 11 | Devtools 全局设置 |

命令行参数示例：

```bash
java -jar myapp.jar --server.port=9090
```

## 高级配置功能

### 导入额外配置

使用 `spring.config.import` 导入其他配置文件：

```yaml
spring:
  config:
    import:
      - optional:classpath:/extra-config.yml
```

### 随机值配置

使用 `${random.*}` 生成随机值：

```yaml
myapp:
  secret: ${random.value}
  uuid: ${random.uuid}
  number: ${random.int[1,100]}
```

### 多文档配置

在 YAML 文件中使用 `---` 分隔多个配置文档，结合 Profiles 实现环境特定配置：

```yaml
server:
  port: 8080
---
spring:
  config:
    activate:
      on-profile: dev
server:
  port: 8081
---
spring:
  config:
    activate:
      on-profile: prod
server:
  port: 8082
```

### Profiles

Profiles 提供环境特定配置，可通过 `spring.profiles.active` 激活：

```yaml
spring:
  profiles:
    active: dev
```

支持 Profile 分组：

```yaml
spring:
  profiles:
    group:
      main:
        - main1
        - main2
```

### 配置加密

使用 Jasypt 加密敏感信息：

```properties
jasypt.encryptor.password=secretkey myapp.username=ENC(encryptedUsername) myapp.password=ENC(encryptedPassword)
```

添加 Jasypt 依赖：

```xml
<dependency>
    <groupId>com.github.ulisesbocchio</groupId>
    <artifactId>jasypt-spring-boot-starter</artifactId>
    <version>3.0.5</version>
</dependency>
```

### 配置迁移

升级 Spring Boot 版本时，使用 `spring-boot-properties-migrator` 检测和替换废弃属性：

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-properties-migrator</artifactId>
    <scope>runtime</scope>
</dependency>
```

## 配置加载机制

Spring Boot 通过 `PropertySourceLoader` 接口加载配置文件，支持 `PropertiesPropertySourceLoader` 和 `YamlPropertySourceLoader`，分别处理 `.properties` 和 `.yml` 文件。

加载顺序如下：

1. 初始化配置处理
2. 处理无 Profiles 的配置
3. 处理带 Profiles 的配置

配置文件加载由 `EnvironmentPostProcessorApplicationListener` 监听器触发，监听 `ApplicationEnvironmentPreparedEvent` 事件。

## 结论

Spring Boot 的配置管理系统提供了从简单属性绑定到复杂环境配置的全面支持。

推荐以下最佳实践：

- **使用 YAML 格式**：层次结构清晰，适合复杂配置。
- **利用 Profiles**：管理开发、测试、生产环境。
- **加密敏感信息**：使用 Jasypt 或配置中心保护数据。
- **外部化配置**：通过命令行参数或环境变量提高灵活性。
- **定期检查废弃属性**：升级版本时使用配置迁移工具。

---

参考资料: 

- [Using application.yml vs application.properties in Spring Boot](https://www.baeldung.com/spring-boot-yaml-vs-properties)
- [What Happens When You Use Both application.properties and application.yml in a Spring Boot Project?](https://medium.com/%40Mohd_Aamir_17/what-happens-when-you-use-both-application-properties-and-application-yml-in-a-spring-boot-project-d506b8f7a6a8)
- [Benefits of Spring Boot's application.yml file with examples](https://www.theserverside.com/blog/Coffee-Talk-Java-News-Stories-and-Opinions/yaml-vs-properties-yml-application-spring-boot-configuration-difference-compare-value-map-list)
- [Externalized Configuration](https://docs.spring.io/spring-boot/docs/1.1.0.M1/reference/html/boot-features-external-config.html)
- [Clarify precedence between yml and properties files on the same level](https://github.com/spring-projects/spring-boot/issues/25121)
- [Externalized Configuration](https://docs.spring.io/spring-boot/docs/1.2.3.RELEASE/reference/html/boot-features-external-config.html)
- [Properties with Spring and Spring Boot](https://www.baeldung.com/properties-with-spring)
- [Properties & configuration](https://docs.spring.io/spring-boot/docs/1.2.0.M1/reference/html/howto-properties-and-configuration.html)
- [application.yml vs bootstrap.yml in Spring Boot](https://www.geeksforgeeks.org/difference-between-application-properties-and-bootstrap-properties/)












