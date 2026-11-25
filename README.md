# SMS Multi Provider

多厂商短信发送 SDK，让开发者在 5 分钟内发送第一条短信。

## 项目定位

- **SDK 而非平台**：提供发送能力，不提供管理界面
- **多厂商支持**：阿里云、天翼云，易于扩展更多厂商
- **开箱即用**：Spring Boot Starter 自动配置
- **生产就绪**：负载均衡、熔断降级、重试机制

## 快速开始

### 1. 添加依赖

```xml
<dependency>
    <groupId>org.moretea.sms</groupId>
    <artifactId>sms-starter</artifactId>
    <version>1.0.0-SNAPSHOT</version>
</dependency>
```

### 2. 配置短信密钥

```yaml
sms:
  providers:
    aliyun:
      enabled: true
      access-key-id: your-access-key
      access-key-secret: your-secret
      sign-name: 阿里云短信测试
      weight: 10
```

### 3. 发送短信

```java
@Autowired
private SmsFacade smsFacade;

// 最简单方式
SmsResponse response = smsFacade.send("13800138000", "您的验证码是1234");

// 使用模板（推荐）
SmsRequest request = SmsRequest.builder()
    .phoneNumber("13800138000")
    .templateCode("verification_code")
    .templateParams(Map.of("code", "1234"))
    .build();
SmsResponse response = smsFacade.sendTemplate(request);

// 检查结果
if (response.getSendStatus()) {
    System.out.println("发送成功！ID: " + response.getUniqueId());
}
```

## 项目结构

```
sms-multi-provider/
├── sms-api              # API 接口定义（核心抽象）
├── sms-service          # 服务商实现（阿里云、天翼云）
├── sms-load-balance     # 负载均衡（随机、轮询、加权）
├── sms-facade           # 门面层（对外 API）
├── sms-queue            # 消息队列支持（Kafka、RabbitMQ）
├── sms-starter          # Spring Boot Starter（自动配置）
└── sms-example          # 使用示例
```

## 核心特性

### 多厂商支持
- ✅ 阿里云短信
- ✅ 天翼云短信
- 🔄 易于扩展更多厂商

### 负载均衡
- 随机策略
- 轮询策略
- 加权策略（推荐）

### 高可用
- 熔断机制（失败阈值 + 恢复时间）
- 快速重试（立即切换服务商）
- 延迟重试（定时任务队列）

### 简洁 API

| 方法 | 说明 | 场景 |
|------|------|------|
| `send(phone, message)` | 直接发送 | 简单通知 |
| `sendTemplate(request)` | 模板发送 | 验证码、订单通知 |
| `sendAsync(batch)` | 异步批量 | 营销短信 |

## 完整配置示例

```yaml
sms:
  providers:
    aliyun:
      enabled: true
      access-key-id: ${ALIYUN_ACCESS_KEY_ID}
      access-key-secret: ${ALIYUN_ACCESS_KEY_SECRET}
      domain: dysmsapi.aliyuncs.com
      sign-name: 阿里云短信测试
      weight: 10
      
    ctyun:
      enabled: true
      access-key: ${CTYUN_ACCESS_KEY}
      secret-key: ${CTYUN_SECRET_KEY}
      sign-name: 天翼云短信
      weight: 5

  load-balance:
    enabled: true
    strategy: weighted

  retry:
    enabled: true
    immediate-limit: 3
    max-retry-times: 10
    fixed-delay: 5000
```

## 示例项目

详见 `sms-example/` 目录，包含：
- REST API 演示
- 业务代码演示
- 完整配置示例

## 开发计划

### MVP (当前)
- ✅ 阿里云、天翼云实现
- ✅ 负载均衡、重试机制
- ✅ Spring Boot Starter
- ✅ 使用示例

### v0.2.0
- 配置校验（@Validated）
- 历史记录持久化
- 监控指标（Micrometer）

### v0.3.0
- 腾讯云、华为云支持
- 通用 HTTP 适配器
- Maven Central 发布

## 构建

```bash
# 安装到本地 Maven 仓库
mvn clean install -DskipTests

# 运行示例
cd sms-example
mvn spring-boot:run
```

## License

MIT

## 作者

greyson
