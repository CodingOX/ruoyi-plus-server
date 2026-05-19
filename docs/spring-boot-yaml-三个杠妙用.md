# Spring Boot YAML 中 `---` 的妙用：同一个文件，多个文档

> 你在看 RuoYi-Vue-Plus 配置文件时，有没有注意到满屏的 `---`？
> 以为 YAML 文件只能写一段配置？大错特错 — 三个横线，打开新世界。

## 一、问题：一个 YAML 文件能写几段配置？

大部分人接触的 YAML 配置长这样：

```yaml
server:
  port: 8080

spring:
  datasource:
    url: jdbc:mysql://localhost:3306/db
  redis:
    host: localhost
```

不管配置多复杂，都在一个文档里。当配置项多了以后，缩进越来越深，可读性越来越差。

## 二、标准 YAML 规范：文档分隔符

根据 [YAML 1.2 规范](https://yaml.org/spec/1.2.2/)：

> `---` 表示 **文档开始**，`...` 表示 **文档结束**。
> 一个 YAML 文件可以包含**多个 YAML 文档**，用 `---` 分隔。

```yaml
---  # 第一个文档
name: Alice

---  # 第二个文档
name: Bob
```

这不算什么冷知识，很多工具（如 `kubectl`）都在用多文档 YAML。但在 Spring Boot 中，这个特性被玩出了花。

## 三、Spring Boot 中的 `---` 实践

### 3.1 基础用法：分块定义不同模块

看 RuoYi-Vue-Plus 的 `application-dev.yml`：

```yaml
--- # 数据源配置
spring:
  datasource:
    type: com.zaxxer.hikari.HikariDataSource
    dynamic:
      datasource:
        master:
          url: jdbc:mysql://localhost:3306/ry-vue
          username: root
          password: root

--- # redis 单机配置
spring.data:
  redis:
    host: localhost
    port: 6379
    password: ruoyi123

--- # mail 邮件发送
mail:
  enabled: false
  host: smtp.163.com
  port: 465

--- # sms 短信配置
sms:
  config-type: yaml
  blends:
    config1:
      supplier: alibaba
      access-key-id: 您的accessKey
```

每个 `---` 后的块都是一个**独立的 YAML 文档**，Spring Boot 启动时会将它们**合并**成一个配置上下文。

### 3.2 带来的好处

**① 避免深层缩进**

不拆文档的话 Redis 配置要这样写：

```yaml
spring:
  data:
    redis:
      host: localhost
      port: 6379
```

拆开后：

```yaml
--- # redis
spring.data:
  redis:
    host: localhost
    port: 6379
```

`---` 后的文档默认缩进从第一列开始，且 Spring Boot 支持**键的路径式写法**（`spring.data.redis`），大大减少了缩进层级。

**② 按模块组织，注释清晰**

每个 `---` 后可以跟 `#` 注释说明当前块是哪个模块，阅读者一目了然：

```yaml
--- # 监控中心配置
--- # snail-job 配置
--- # 数据源配置
--- # redis 单机配置
--- # mail 邮件发送
--- # sms 短信 支持 阿里云 腾讯云 云片 ...
--- # 三方授权
```

比在同一个文档里到处插注释清晰得多。

**③ Profile 文件覆盖更优雅**

`application-dev.yml` 是 `application.yml` 的 profile 覆盖文件。Spring Boot 加载时：

1. 先加载 `application.yml` 中的所有 `---` 文档
2. 再加载 `application-dev.yml` 中的所有 `---` 文档
3. 同路径的 key，profile 文件覆盖主文件

可以精准覆盖某一块配置，而不影响其他块。

## 四、背后原理：Spring Boot 的 YAML 加载机制

Spring Boot 使用 SnakeYAML 解析 YAML 文件。`org.springframework.boot.env.YamlPropertySourceLoader` 会：

1. 按 `---` 分割 YAML 文件，得到多个 **YAML 文档（Document）**
2. 将每个文档扁平化为 `key=value` 的 Properties
3. 所有文档合并成一个 `PropertySource`
4. 后加载的 key 覆盖先加载的 key

示例：

```yaml
--- # 文档 1
app:
  name: my-app
  version: 1.0

--- # 文档 2
app:
  version: 2.0
```

最终结果是 `app.name=my-app`，`app.version=2.0`（文档 2 覆盖了文档 1 的 version）。

## 五、注意事项

### 5.1 不要混用不同 Profile 的配置

`---` 分隔的文档共享同一个 `spring.profiles.active` 上下文，不能在一个文件中用 `---` 来区分不同环境。不同环境的配置还是要通过 `application-{profile}.yml` 来区分。

### 5.2 注意文档顺序

后加载的文档会覆盖前面的，如果你在同一文件中有重复的 key，最后出现的生效。

### 5.3 兼容性问题

绝大多数 YAML 解析器都支持多文档，但如果你用 `@PropertySource` 手动加载 YAML 文件，需要确认解析器是否支持。

## 六、总结

| 问题 | 答案 |
|------|------|
| `---` 是 Spring Boot 特有的吗？ | **不是**，是标准 YAML 规范 |
| 有什么用？ | **按模块分块**，减少缩进，提高可读性 |
| Spring Boot 怎么处理？ | 拆成多个文档，**合并**为一个配置上下文 |
| 和 `application-{profile}.yml` 什么关系？ | `---` 处理的是**文件内部结构**，profile 处理的是**文件外部选择**，互不冲突 |

以前总觉得 YAML 就是比 properties 多个缩进，现在看来，三个杠带来的不仅是可读性，更是一种**配置即文档**的编码风格。

下次打开 RuoYi-Vue-Plus 的配置文件时，看到 `---`，你会心一笑——哦，这是另一个模块。
