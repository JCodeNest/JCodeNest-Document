# 04-SpringBoot-Mvnd

在 Spring Boot 项目开发中，Maven 是主流的构建工具，但其传统构建速度较慢，特别是在大型多模块项目中。为提升构建效率，Apache 推出了 **maven-mvnd（Maven Daemon）**，结合 GraalVM 和守护进程技术显著优化构建性能。同时，Spring Boot 集成了 **Maven Wrapper（mvnw）**，提供免安装、版本灵活的 Maven 使用方式。本文将探讨如何在 Spring Boot 中使用 Maven、mvnd 和 mvnw，分析其原理、应用场景及实践方法。

## maven-mvnd 基本介绍

maven-mvnd 是一个增强型 Maven 构建工具，旨在通过借鉴 Gradle 和 Takari 的技术，提供更快的构建速度。它并非独立工具，而是对 Maven 的封装和优化，底层仍依赖 Maven 的核心功能。

> 开源地址： [https://github.com/apache/maven-mvnd/](https://github.com/apache/maven-mvnd/)

### mvnd 核心特性:

1. **守护进程模式**: 
    * mvnd 在后台运行一个长驻守护进程，处理多次构建请求，避免每次构建重新启动 JVM。
    * 守护进程缓存 Maven 插件类加载器，减少 JAR 文件的重复读取和解析。
    * JVM 的即时编译（JIT）优化代码在多次构建中复用，降低编译开销。
2. **GraalVM 支持**: mvnd 客户端使用 GraalVM 构建的本机可执行文件，启动速度快，内存占用低。
3. **并行构建**: 默认利用多核 CPU 并行构建模块，核心数由公式 `Math.max(Runtime.getRuntime().availableProcessors() - 1, 1)` 决定。例如，在 24 核机器上可高效利用 CPU 资源。

### mvnd 为什么更快？

- **无需重复启动 JVM**: 传统 Maven 每次构建需启动新的 JVM，而 mvnd 的守护进程保持运行，节省初始化时间。
- **插件缓存**: 插件 JAR 只解析一次，类加载器缓存复用。
- **JIT 优化**: GraalVM 的 JIT 编译生成高效本机代码，重复构建时直接可用。
- **并行处理**: 多模块项目并行构建，充分利用硬件资源。

官方测试效果图: 

![Img](https://user-images.githubusercontent.com/1826249/103917178-94ee4500-510d-11eb-9abb-f52dae58a544.gif)

### 安装与使用

**安装**: 参考 [mvnd 官方文档](https://github.com/apache/maven-mvnd/#how-to-install-mvnd) 获取最新安装方法。mvnd 内置 Maven，无需单独安装 Maven。

我这里是 MacOS, 直接使用 Homebrew 安装即可: 

```bash
brew install mvndaemon/homebrew-mvnd/mvnd
```

**验证版本**: 

```bash
mvnd --version
```

输出 mvnd 和内置 Maven 版本信息。

**附加选项**: 

- `--status`：列出当前运行的守护进程。
- `--stop`：停止所有守护进程。
- 查看更多选项：`mvnd --help`。

使用方式：mvnd 命令与 Maven 兼容，仅需将 mvn 替换为 mvnd，如：

```bash
mvnd clean install
```

## Maven Wrapper（mvnw）

Spring Boot 项目通过 [start.spring.io](start.spring.io) 生成时，默认包含 Maven Wrapper（mvnw），提供免安装、版本灵活的 Maven 使用方式，源自 maven-mvnd 项目。

**mvnw 核心特性:** 

* **免安装:** 无需手动安装 Maven，mvnw 自动下载指定版本的 Maven 到本地仓库（`$USER_HOME/.m2/wrapper`）。
* **版本灵活:** 通过配置文件指定 Maven 版本，适合多项目使用不同 Maven 版本的场景。
* **跨平台:** 包含 `mvnw`（Linux/macOS）和 `mvnw.cmd`（Windows）脚本。

**生成项目结构:** 

```plain
├── mvnw
├── mvnw.cmd
├── .mvn
│   └── wrapper
│       └── maven-wrapper.properties
├── pom.xml
└── src
    ├── main
    │   ├── java
    │   │   └── com/example/demo
    │   │       └── DemoApplication.java
    │   └── resources
    │       └── application.properties
    └── test
        └── java
            └── com/example/demo
                └── DemoApplicationTests.java
```

配置文件（`.mvn/wrapper/maven-wrapper.properties`）: 

```properties
distributionUrl=https://repo.maven.apache.org/maven2/org/apache/maven/apache-maven/3.9.9/apache-maven-3.9.9-bin.zip
```

更新 Maven 版本：修改 `distributionUrl` 指定所需版本，例如：

```properties
distributionUrl=https://repo.maven.apache.org/maven2/org/apache/maven/apache-maven/3.9.9/apache-maven-3.9.9-bin.zip
```

使用方式: (在项目根目录执行)

```bash
./mvnw clean install
```

mvnw 底层调用 Maven，自动下载指定版本到本地仓库（`$USER_HOME/.m2/wrapper`），无需手动安装。命令与 `mvn` 一致，支持所有 Maven 生命周期命令。

> **IDEA配置:** 
> 1. 打开 IDEA 的 Maven 设置（`File > Settings > Build, Execution, Deployment > Build Tools > Maven`）。
> 2. 设置 Maven home path 为项目中的 `.mvn/wrapper/maven-wrapper.properties` 指定的版本。
> 3. 确保项目根目录包含 `.mvn` 目录和 `mvnw` 脚本。

## mvnd 与 mvnw 对比

| 特性 | mvnd | mvnw |
| -- | -- | -- |
| 目的 | 增强 Maven 构建速度 | 提供免安装、版本灵活的 Maven 使用 |
| 底层实现 | 守护进程 + GraalVM | Maven 包装脚本，下载指定版本 |
| 性能 | 显著提升（约 3 倍） | 与传统 Maven 性能相同 |
| 适用场景 | 大型项目、频繁构建 | 多版本 Maven 管理、免安装场景 |
| Spring Boot 集成 | 需手动安装 | 默认包含在 Spring Boot 项目 |

**选择建议:** 

- **mvnd:** 推荐用于大型多模块项目或需要高频构建的场景，显著提升效率。
- **mvnw:** 适合快速启动项目、跨团队统一 Maven 版本或避免手动安装的场景，特别在 CI/CD 环境中实用。
- **结合使用:** Spring Boot 项目可先用 mvnw 初始化，结合 mvnd 加速构建。

## 快速实践：Spring Boot 项目集成 mvnd

以下以 Spring Boot 3.4.7 为例，展示如何结合 mvnd 构建项目。

1. **生成项目:** 使用 [start.spring.io](start.spring.io) 生成项目, 下载并解压项目，包含 mvnw 和 .mvn 目录。
2. **安装 mvnd:** 按 [官方指南](https://github.com/apache/maven-mvnd/#how-to-install-mvnd) 安装 mvnd。验证：`mvnd --version`。
3. **构建项目:** 

```bash
# 在项目根目录运行：
mvnd clean install

# 或使用 mvnw：
./mvnw clean install
```

5. **运行应用:** 

```bash
# 使用 mvnd 运行：
mvnd spring-boot:run
```

输出日志显示 Tomcat 启动于 8080 端口。

## 总结

**maven-mvnd** 通过守护进程、GraalVM 和并行构建显著提升 Maven 构建速度，特别适合大型 Spring Boot 项目。**Maven Wrapper（mvnw）** 提供免安装、版本灵活的 Maven 使用方式，与 Spring Boot 项目无缝集成。

开发者可根据项目规模和需求选择：

- **小型项目/快速上手:** 使用 mvnw，免配置直接构建。
- **大型项目/高频构建:** 采用 mvnd，优化性能。
- **混合使用:** 在 mvnw 项目中引入 mvnd，兼顾便捷性和效率。