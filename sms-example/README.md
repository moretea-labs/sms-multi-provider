# SMS Multi Provider 使用示例

## 快速开始

### 1. 安装 SDK 到本地 Maven 仓库

```bash
cd ..
mvn clean install -DskipTests
```

### 2. 配置短信密钥

编辑 `application.yml`：

```yaml
sms:
  providers:
    aliyun:
      enabled: true
      access-key-id: 你的阿里云AccessKey
      access-key-secret: 你的阿里云Secret
      sign-name: 你的短信签名
      weight: 10
```

或使用环境变量：

```bash
export ALIYUN_ACCESS_KEY_ID=your-key
export ALIYUN_ACCESS_KEY_SECRET=your-secret
```

### 3. 启动应用

```bash
mvn spring-boot:run
```

### 4. 测试发送

#### 4.1 检查健康状态

```bash
curl http://localhost:8080/actuator/health
```

#### 4.2 简单发送

```bash
curl "http://localhost:8080/demo/send/simple?phone=13800138000&message=hello"
```

响应示例：

```json
{
  "success": false,
  "status": false,
  "message": "发送失败",
  "uniqueId": "1709834567890-1234"
}
```

#### 4.3 模板发送

```bash
curl -X POST http://localhost:8080/demo/send/template \
  -H "Content-Type: application/json" \
  -d '{
    "phone": "13800138000",
    "templateCode": "verification_code",
    "params": {"code": "123456"}
  }'
```

#### 4.4 批量发送

```bash
curl -X POST http://localhost:8080/demo/send/batch \
  -H "Content-Type: application/json" \
  -d '{
    "templateCode": "verification_code",
    "phones": ["13800138000", "13800138001"],
    "params": {"code": "123456"}
  }'
```

## 在业务代码中使用

```java
@Service
public class UserService {
    
    @Autowired
    private SmsFacade smsFacade;
    
    public void sendVerificationCode(String phone, String code) {
        // 方式1：直接发送
        SmsResponse response = smsFacade.send(phone, "验证码：" + code);
        
        // 方式2：使用模板（推荐）
        SmsRequest request = SmsRequest.builder()
                .phoneNumber(phone)
                .templateCode("verification_code")
                .templateParams(Map.of("code", code))
                .build();
        SmsResponse response = smsFacade.sendTemplate(request);
        
        // 处理结果
        if (response.getSendStatus()) {
            System.out.println("发送成功！");
        }
    }
}
```

## 注意事项

1. **未配置密钥时**：服务可以启动，但发送会失败（请查看日志）
2. **模板配置**：需要在 `templates.json` 中定义模板
3. **多厂商**：配置多个厂商后，会自动按权重负载均衡
4. **重试机制**：发送失败会自动重试（可配置）

## 项目结构

```
sms-example/
├── src/main/java/org/moretea/sms/example/
│   ├── SmsExampleApplication.java          # 启动类
│   ├── controller/
│   │   └── SmsDemoController.java          # REST API 演示
│   └── service/
│       └── SmsDemoService.java             # 业务代码演示
├── src/main/resources/
│   ├── application.yml                     # 应用配置
│   └── templates.json                      # 短信模板配置
└── pom.xml                                 # Maven 配置
```
