# SMS Multi Provider - MVP API 使用指南

## 快速开始

### 1. 添加依赖

```xml
<dependency>
    <groupId>org.moretea.sms</groupId>
    <artifactId>sms-starter</artifactId>
    <version>1.0.0-SNAPSHOT</version>
</dependency>
```

### 2. 配置 application.yml

```yaml
sms:
  providers:
    aliyun:
      access-key-id: your-access-key
      access-key-secret: your-secret
      sign-name: 阿里云短信测试
      weight: 10
    ctyun:
      access-key: your-access-key
      secret-key: your-secret
      sign-name: 天翼云短信
      weight: 5
  
  load-balance:
    enabled: true
    strategy: weighted
  
  retry:
    enabled: true
    immediate-limit: 3
```

### 3. 注入使用

```java
@Service
public class UserService {
    
    @Autowired
    private SmsFacade smsFacade;
    
    /**
     * 发送验证码（最简单方式）
     */
    public void sendVerificationCode(String phone, String code) {
        SmsResponse response = smsFacade.send(phone, "您的验证码是：" + code);
        
        if (Boolean.TRUE.equals(response.getSendStatus())) {
            System.out.println("发送成功，ID: " + response.getUniqueId());
        } else {
            System.out.println("发送失败: " + response.getMsg());
        }
    }
    
    /**
     * 使用模板发送（推荐）
     */
    public void sendWithTemplate(String phone, String templateCode, Map<String, String> params) {
        SmsRequest request = SmsRequest.builder()
                .phoneNumber(phone)
                .templateCode(templateCode)
                .templateParams(params)
                .build();
        
        SmsResponse response = smsFacade.sendTemplate(request);
        // 处理结果...
    }
    
    /**
     * 完整参数发送
     */
    public void sendWithFullParams() {
        SmsRequest request = SmsRequest.builder()
                .phoneNumber("13800138000")
                .templateCode("SMS_123456")
                .signName("自定义签名")  // 覆盖配置中的默认签名
                .templateParams(Map.of("code", "1234", "name", "张三"))
                .build();
        
        SmsResponse response = smsFacade.sendTemplate(request);
        System.out.println("状态: " + response.getSendStatus());
        System.out.println("消息: " + response.getMsg());
        System.out.println("追踪ID: " + response.getUniqueId());
        System.out.println("详情: " + response.getData());
    }
}
```

## API 对比

### 新版 API（推荐）

```java
// 最简方式
SmsResponse response = smsFacade.send("13800138000", "您的验证码是1234");

// 模板发送
SmsResponse response = smsFacade.sendTemplate(request);

// 异步批量
CompletableFuture<Void> future = smsFacade.sendAsync(batchRequest);
```

### 旧版 API（已废弃，但仍可用）

```java
// 已废弃，建议使用 sendTemplate
smsFacade.sendByTemplateSimple(request);

// 已废弃，建议使用 sendAsync
smsFacade.sendByTemplateMulti(batchRequest);
```

## 核心改进

| 改进项 | 旧版 | 新版 |
|--------|------|------|
| **结果返回** | `void`（无返回） | `SmsResponse`（完整结果） |
| **简洁 API** | 只有长方法名 | `send()` / `sendTemplate()` |
| **签名处理** | 需每次传入 | 支持配置默认签名 |
| **权重配置** | 从 Environment 读取 | 从 Provider.getWeight() 读取 |

## SmsResponse 结构

```java
public class SmsResponse<T> {
    private Boolean status;      // 请求是否成功（true=请求成功，false=请求失败）
    private Boolean sendStatus;  // 发送是否成功（true=发送成功，false=发送失败）
    private String msg;          // 响应消息
    private String uniqueId;     // 追踪ID
    private T data;              // 额外数据（如服务商返回的详情）
}
```

## 配置说明

### 最小配置

```yaml
sms:
  providers:
    aliyun:
      access-key-id: xxx
      access-key-secret: xxx
      sign-name: 测试签名
```

### 多厂商 + 负载均衡

```yaml
sms:
  providers:
    aliyun:
      access-key-id: xxx
      access-key-secret: xxx
      sign-name: 阿里云签名
      weight: 10          # 权重越高，被选中的概率越大
    ctyun:
      access-key: xxx
      secret-key: xxx
      sign-name: 天翼云签名
      weight: 5
  
  load-balance:
    enabled: true
    strategy: weighted    # random / round-robin / weighted
```

### 重试配置

```yaml
sms:
  retry:
    enabled: true
    immediate-limit: 3    # 立即重试次数（失败后立即换服务商重试）
    max-retry-times: 10   # 最大重试次数（包括延迟重试）
    fixed-delay: 5000     # 延迟重试间隔（毫秒）
```
