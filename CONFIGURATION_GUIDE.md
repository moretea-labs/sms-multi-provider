# SMS Multi Provider - 配置自动装配完成文档

## ✅ 已完成内容

### 1. 配置属性类（带校验）

#### 全局配置 - `SmsProperties`
位置：`sms-api/src/main/java/org/moretea/sms/api/config/SmsProperties.java`

```yaml
sms:
  load-balance:
    enabled: true           # 布尔值
    strategy: weighted      # 字符串：random / round-robin / weighted
  retry:
    enabled: true           # 布尔值
    immediate-limit: 3      # 整数，范围 0-10
    max-retry-times: 10     # 整数，范围 0-50
    fixed-delay: 5000       # 整数，范围 1000-60000（毫秒）
```

#### 阿里云配置 - `AliyunSmsProviderProperties`
位置：`sms-service/src/main/java/org/moretea/sms/service/provider/aliyun/AliyunSmsProviderProperties.java`

```java
@Validated
@ConfigurationProperties("sms.providers.aliyun")
public class AliyunSmsProviderProperties {
    @NotBlank private String accessKeyId;
    @NotBlank private String accessKeySecret;
    @NotBlank private String signName;
    // ...
}
```

#### 天翼云配置 - `CtyunSmsProviderProperties`
位置：`sms-service/src/main/java/org/moretea/sms/service/provider/tianyiyun/CtyunSmsProviderProperties.java`

```java
@Validated
@ConfigurationProperties("sms.providers.ctyun")
public class CtyunSmsProviderProperties {
    @NotBlank private String accessKey;
    @NotBlank private String secretKey;
    @NotBlank private String signName;
    // ...
}
```

### 2. 自动配置入口

#### 主配置类 - `SmsMultiProviderAutoConfiguration`
位置：`sms-starter/src/main/java/org/moretea/sms/starter/SmsMultiProviderAutoConfiguration.java`

```java
@AutoConfiguration
@EnableConfigurationProperties(SmsProperties.class)
@ComponentScan(basePackages = {
    "org.moretea.sms.facade",
    "org.moretea.sms.load.balance",
    "org.moretea.sms.service"
})
@Import({SmsServiceAutoConfiguration.class, SmsAsyncConfiguration.class})
```

#### Spring Boot 自动装配文件
- `META-INF/spring.factories` - 传统方式
- `META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports` - Spring Boot 2.7+ 推荐

### 3. 厂商自动配置 - `SmsServiceAutoConfiguration`
位置：`sms-service/src/main/java/org/moretea/sms/service/config/SmsServiceAutoConfiguration.java`

特性：
- ✅ 条件装配：`@ConditionalOnProperty` 根据配置决定是否启用
- ✅ 缺失bean保护：`@ConditionalOnMissingBean` 避免重复注册
- ✅ 阿里云：当存在 `dysmsapi20170525.AsyncClient` 且配置了 `access-key-id` 时启用
- ✅ 天翼云：当配置了 `access-key` 时启用

### 4. 负载均衡配置读取

`LoadBalancerManager` 现在从 `SmsProperties` 读取配置：

```java
@Autowired
private SmsProperties smsProperties;

@PostConstruct
public void init() {
    this.userConfigStrategyName = smsProperties.getLoadBalance().getStrategy();
    if (smsProperties.getLoadBalance().isEnabled()) {
        // ...
    }
}
```

## 📋 完整配置示例

### application.yml
```yaml
sms:
  # 负载均衡配置
  load-balance:
    enabled: true
    strategy: weighted    # random / round-robin / weighted

  # 重试配置
  retry:
    enabled: true
    immediate-limit: 3      # 失败后立即换厂商重试次数
    max-retry-times: 10     # 最大重试次数
    fixed-delay: 5000       # 延迟重试间隔（毫秒）

  # 厂商配置
  providers:
    aliyun:
      enabled: true
      access-key-id: ${ALIYUN_ACCESS_KEY_ID}
      access-key-secret: ${ALIYUN_ACCESS_KEY_SECRET}
      sign-name: 阿里云短信测试
      weight: 10

    ctyun:
      enabled: true
      access-key: ${CTYUN_ACCESS_KEY}
      secret-key: ${CTYUN_SECRET_KEY}
      sign-name: 天翼云短信测试
      weight: 5
```

### 最小配置
```yaml
sms:
  providers:
    aliyun:
      access-key-id: xxx
      access-key-secret: xxx
      sign-name: 测试签名
```

## 🔐 配置校验

当配置无效时，Spring Boot 启动会报错：

```
BindingException: 
  - sms.providers.aliyun.access-key-id: 阿里云 Access Key ID 不能为空
  - sms.providers.aliyun.access-key-secret: 阿里云 Access Key Secret 不能为空
```

## 🚀 使用方式

### 1. 添加依赖
```xml
<dependency>
    <groupId>org.moretea.sms</groupId>
    <artifactId>sms-starter</artifactId>
    <version>1.0.0-SNAPSHOT</version>
</dependency>
```

### 2. 配置 application.yml
见上文完整配置示例

### 3. 注入使用
```java
@Autowired
private SmsFacade smsFacade;

public void send() {
    SmsResponse response = smsFacade.send("13800138000", "测试短信");
}
```

## 📁 相关文件清单

| 文件 | 路径 | 说明 |
|------|------|------|
| SmsProperties | sms-api/src/main/java/.../api/config/ | 全局配置 |
| AliyunSmsProviderProperties | sms-service/.../aliyun/ | 阿里云配置 |
| CtyunSmsProviderProperties | sms-service/.../tianyiyun/ | 天翼云配置 |
| SmsMultiProviderAutoConfiguration | sms-starter/.../starter/ | 主自动配置 |
| SmsServiceAutoConfiguration | sms-service/.../config/ | 厂商自动配置 |
| spring.factories | sms-starter/src/main/resources/META-INF/ | 自动装配入口 |
| AutoConfiguration.imports | sms-starter/.../META-INF/spring/ | 新版自动装配 |

## ✅ 编译验证

```bash
$ mvn clean install -DskipTests

[INFO] sms-multi-provider ......................... SUCCESS
[INFO] sms-api .................................... SUCCESS
[INFO] sms-service ................................ SUCCESS
[INFO] sms-load-balance ........................... SUCCESS
[INFO] sms-facade ................................. SUCCESS
[INFO] sms-queue .................................. SUCCESS
[INFO] SMS Multi Provider Starter ................. SUCCESS

BUILD SUCCESS
```
