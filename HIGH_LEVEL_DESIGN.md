# SMS-Multi-Provider 高层设计文档

> **愿景**：成为 Java 生态中最易用、可扩展的多厂商短信发送开源组件
> 
> **目标用户**：
> 1. **开发者**：直接引用 SDK 集成到业务系统
> 2. **平台方**：基于组件构建带 Web 管理的短信平台
> 3. **ISV/SaaS**：多租户场景下的短信能力提供商

---

## 1. 产品定位演进路线

```
┌─────────────────────────────────────────────────────────────────┐
│                     产品形态演进                                  │
├─────────────────────────────────────────────────────────────────┤
│  Phase 1: 核心组件 (sms-multi-provider)                         │
│  └── Java SDK，专注「发送能力」                                   │
│  └── 核心价值：屏蔽厂商差异、负载均衡、熔断重试                      │
├─────────────────────────────────────────────────────────────────┤
│  Phase 2: 管理平台 (sms-platform-web)                           │
│  └── 基于组件构建的 Web 管理平台                                   │
│  └── 核心价值：可视化管理、多租户、运营分析                          │
├─────────────────────────────────────────────────────────────────┤
│  Phase 3: 云原生平台 (sms-cloud)                                │
│  └── 独立部署的短信中台服务                                        │
│  └── 核心价值：Serverless、弹性伸缩、按量计费                        │
└─────────────────────────────────────────────────────────────────┘
```

**设计原则**：
- 底层组件「小而美」，专注核心能力
- 上层平台「大而全」，承载业务复杂度
- 两层之间通过「标准 SPI」衔接，可替换实现

---

## 2. 架构分层设计

### 2.1 整体架构图

```
┌──────────────────────────────────────────────────────────────────────────┐
│                           应用层 (Application)                            │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐  │
│  │ 电商订单系统   │  │ 金融风控系统   │  │ 营销推送平台   │  │ SaaS 服务商   │  │
│  └──────┬───────┘  └──────┬───────┘  └──────┬───────┘  └──────┬───────┘  │
└─────────┼────────────────┼────────────────┼────────────────┼──────────┘
          │                │                │                │
          └────────────────┴───────┬────────┴────────────────┘
                                 │
┌────────────────────────────────▼─────────────────────────────────────────┐
│                        SMS SDK (开源组件层)                               │
│  ┌─────────────────────────────────────────────────────────────────────┐ │
│  │                      sms-multi-provider                              │ │
│  │  ┌────────────┐  ┌────────────┐  ┌────────────┐  ┌────────────┐     │ │
│  │  │ sms-api    │  │sms-service │  │sms-facade  │  │sms-queue   │     │ │
│  │  │ (抽象接口)  │  │ (厂商实现)  │  │ (统一入口)  │  │ (消息队列)  │     │ │
│  │  └────────────┘  └────────────┘  └────────────┘  └────────────┘     │ │
│  │  ┌────────────┐  ┌────────────┐  ┌────────────┐                      │ │
│  │  │sms-load-   │  │sms-starter │  │sms-common  │                      │ │
│  │  │balance     │  │ (启动入口)  │  │ (工具类)    │                      │ │
│  │  └────────────┘  └────────────┘  └────────────┘                      │ │
│  └─────────────────────────────────────────────────────────────────────┘ │
└──────────────────────────────────────────────────────────────────────────┘
                                   │
          ┌────────────────────────┼────────────────────────┐
          │                        │                        │
          ▼                        ▼                        ▼
┌─────────────────┐    ┌─────────────────┐    ┌─────────────────┐
│   阿里云短信     │    │   天翼云短信     │    │   其他厂商...    │
└─────────────────┘    └─────────────────┘    └─────────────────┘
```

### 2.2 模块职责边界

| 模块 | 定位 | 开源暴露 | 平台扩展 |
|------|------|---------|---------|
| `sms-api` | 核心抽象 | ✅ Public API | 不可修改 |
| `sms-service` | 厂商实现 | ✅ 可扩展 | 可新增 Provider |
| `sms-facade` | 统一门面 | ✅ 主要入口 | 可包装增强 |
| `sms-load-balance` | 负载均衡 | ✅ 可配置 | 可自定义策略 |
| `sms-queue` | 异步队列 | ✅ 可选依赖 | 可替换实现 |
| `sms-starter` | 启动封装 | ✅ 开箱即用 | 可定制配置 |
| `sms-admin-api` | 【预留】管理接口 | ⚠️ 平台层 | Web 平台专用 |

---

## 3. 开源组件设计哲学

### 3.1 「渐进式」使用体验

```java
// Level 1: 极简入门（5分钟上手）
@Autowired
private SmsFacade smsFacade;

public void send() {
    SmsRequest request = SmsRequest.builder()
        .phoneNumber("13800138000")
        .templateCode("SMS_123456")
        .templateParams(Map.of("code", "1234"))
        .build();
    smsFacade.sendByTemplateSimple(request);
}

// Level 2: 细粒度控制（高级用法）
@Autowired
private LoadBalancerManager lbManager;

public void sendWithStrategy() {
    // 手动选择厂商
    SmsProvider provider = lbManager.getProvider();
    CompletableFuture<SmsResponse> future = provider.sendSms(request);
}

// Level 3: 自定义扩展（插件开发）
@Component
public class CustomProvider extends AbstractSmsProvider {
    @Override
    public CompletableFuture<SmsResponse> sendSms(SmsRequest request) {
        // 自定义实现
    }
}
```

### 3.2 「约定优于配置」

**零配置启动**：
```yaml
# 最小配置：只需配置密钥
sms:
  aliyun:
    access-key-id: ${ALI_KEY}
    access-key-secret: ${ALI_SECRET}
```

**自动装配原理**：
- 检测到阿里云配置 → 自动启用 AliyunSmsProvider
- 检测到多个厂商 → 自动启用负载均衡
- 检测到 Kafka 配置 → 自动启用队列

### 3.3 「可扩展点」设计

| 扩展点 | 接口 | 用途 |
|--------|------|------|
| 新增厂商 | `SmsProvider` | 接入小众短信服务商 |
| 负载均衡策略 | `LoadBalancerStrategy` | 自定义调度算法 |
| 消息队列 | `SmsProducer` | 替换为 RocketMQ 等 |
| 模板加载 | `TemplateConfiguration` | 从 Apollo/Nacos 加载 |
| 历史存储 | `SmsHistoryService` | 自定义存储（MongoDB等）|
| 回调处理 | `SmsCallbackHandler` | 自定义状态回调逻辑 |

---

## 4. 平台层扩展设计

### 4.1 组件与平台的边界

```
┌─────────────────────────────────────────────────────────────┐
│                    sms-platform-web (闭源/独立项目)           │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐          │
│  │  Web 管理端  │  │  多租户引擎  │  │  计费系统    │          │
│  │  (Vue/React)│  │  (隔离/配额) │  │  (成本核算)  │          │
│  └──────┬──────┘  └──────┬──────┘  └──────┬──────┘          │
│         │                │                │                  │
│  ┌──────▼────────────────▼────────────────▼──────┐          │
│  │         sms-admin-api (扩展模块)               │          │
│  │  ┌─────────────────────────────────────────┐   │          │
│  │  │  TenantService     租户生命周期管理      │   │          │
│  │  │  QuotaService      额度控制             │   │          │
│  │  │  AuditService      操作审计             │   │          │
│  │  │  AnalyticsService  统计分析             │   │          │
│  │  └─────────────────────────────────────────┘   │          │
│  └──────────────────┬─────────────────────────────┘          │
└─────────────────────┼────────────────────────────────────────┘
                      │
                      ▼ 标准 SPI 调用
┌─────────────────────────────────────────────────────────────┐
│              sms-multi-provider (开源组件)                    │
└─────────────────────────────────────────────────────────────┘
```

### 4.2 多租户预留设计

**组件层预留扩展点**：
```java
// sms-api 中预留上下文
public class SmsRequest {
    // ... 现有字段
    
    /**
     * 租户ID（平台层扩展使用）
     * 组件层不处理，透传给扩展点
     */
    private String tenantId;
    
    /**
     * 业务线标识（用于分组统计）
     */
    private String bizLine;
}

// 扩展接口支持多租户
public interface SmsHistoryService {
    /**
     * 保存发送记录
     * @param tenantId 租户ID（平台层传入）
     */
    void saveHistory(String tenantId, SmsHistoryEntity history);
}
```

**平台层实现**：
```java
@Service
public class TenantSmsService {
    @Autowired
    private SmsFacade smsFacade;
    
    public SmsResult sendForTenant(String tenantId, SmsRequest request) {
        // 1. 检查租户额度
        quotaService.checkQuota(tenantId);
        
        // 2. 设置租户上下文
        request.setTenantId(tenantId);
        
        // 3. 调用组件发送
        return smsFacade.send(request);
    }
}
```

### 4.3 管理接口规划

```java
/**
 * 平台层管理接口（非组件层）
 * 后续在 sms-platform-web 中实现
 */
@RestController
@RequestMapping("/admin/api/v1")
public class SmsAdminController {
    
    // 租户管理
    @PostMapping("/tenants")  // 创建租户
    @GetMapping("/tenants/{id}/quota")  // 查询额度
    @PostMapping("/tenants/{id}/quota") // 调整额度
    
    // 短信记录查询
    @GetMapping("/records")   // 分页查询（带租户隔离）
    @GetMapping("/records/{id}") // 详情
    @PostMapping("/records/{id}/retry") // 失败重发
    
    // 统计报表
    @GetMapping("/statistics/send")  // 发送统计
    @GetMapping("/statistics/provider") // 厂商占比
    @GetMapping("/statistics/cost")    // 成本分析
    
    // 配置管理
    @GetMapping("/templates")  // 模板列表
    @PostMapping("/templates") // 新增模板
    @GetMapping("/providers")  // 厂商配置
    @PutMapping("/providers/{name}/weight") // 调整权重
}
```

---

## 5. 技术决策记录 (ADR)

### ADR-001: 为什么选择多模块而非单体

**状态**: Accepted

**背景**: 需要平衡「开箱即用」与「按需引入」

**决策**: 
- 使用 Maven 多模块
- 提供 `sms-starter` 统一启动模块
- 队列模块为可选依赖

**后果**:
- ✅ 用户可按需引入（不用 Kafka 就不引）
- ✅ 清晰的分层边界
- ⚠️ 版本管理复杂度增加

### ADR-002: 为什么是 Spring Boot Starter 而非独立 SDK

**状态**: Accepted

**背景**: Java 生态中 Spring Boot 是主流

**决策**: 优先支持 Spring Boot 自动装配，同时暴露核心 API

**后果**:
- ✅ 符合主流开发习惯
- ✅ 配置自动提示（spring-boot-configuration-processor）
- ⚠️ 对非 Spring 项目不够友好（未来可考虑独立客户端）

### ADR-003: 历史记录存储抽象

**状态**: Proposed

**背景**: 不同场景存储需求不同（MySQL/MongoDB/ES）

**决策**: 定义 `SmsHistoryService` 接口，默认提供 JPA 实现

**后果**:
- ✅ 存储层可替换
- ✅ 组件层不依赖具体存储
- ⚠️ 需要定义清晰的实体模型

### ADR-004: 多租户实现层级

**状态**: Proposed

**背景**: 租户隔离是平台需求，非组件需求

**决策**: 组件层预留字段透传，平台层实现隔离逻辑

**后果**:
- ✅ 组件保持简洁
- ✅ 平台可自定义隔离策略（按库/按表/按字段）
- ⚠️ 需要良好的扩展点设计

---

## 6. 开源治理规划

### 6.1 版本策略

```
主版本.次版本.修订号
│     │      │
│     │      └── Bug 修复
│     └───────── 功能新增（向后兼容）
└────────────── 不兼容变更
```

**发布节奏**:
- Patch：按需发布（Bug 修复）
- Minor：每月发布（功能迭代）
- Major：年度发布（架构升级）

### 6.2 社区建设

| 渠道 | 用途 |
|------|------|
| GitHub Issues | Bug 报告、功能请求 |
| GitHub Discussions | 使用问答、技术讨论 |
| 官方文档站 | 完整文档、API 参考 |
| 示例仓库 | quick-start、最佳实践 |
| 博客/推文 | 版本发布、案例分享 |

### 6.3 贡献指南

**代码规范**:
- 遵循阿里巴巴 Java 开发手册
- 单元测试覆盖率 > 80%
- 新增厂商必须提供 Mock 测试

**PR 流程**:
1. Fork & Branch (`feature/xxx` or `fix/xxx`)
2. 编写代码 + 测试
3. 更新文档
4. 提交 PR（触发 CI）
5. Code Review（至少 1 个 Approval）
6. Merge to main

---

## 7. 路线图规划

### Phase 1: 核心稳定（v1.0.0）- 2个月
- [ ] 完成阿里云、天翼云双厂商实现
- [ ] 数据库持久化层实现
- [ ] REST API 完整暴露
- [ ] 基础监控指标（Micrometer）
- [ ] 完整单元测试覆盖
- [ ] 官方文档初版

**发布目标**: 可被第三方项目引用

### Phase 2: 生态完善（v1.1.0~v1.3.0）- 3个月
- [ ] 新增 3+ 厂商（腾讯云、华为云、网易云信）
- [ ] 插件化架构实现（SPI 加载）
- [ ] 通用 HTTP 适配器（简化小众厂商接入）
- [ ] 多语言 SDK 规划（Go/Node.js 客户端）
- [ ] 示例项目（Spring Boot、Spring Cloud）

**发布目标**: 形成初步生态

### Phase 3: 平台就绪（v2.0.0）- 3个月
- [ ] Admin API 标准化
- [ ] 多租户接口预留实现
- [ ] 前端管理界面（开源）
- [ ] 云原生部署方案（K8s Helm Chart）
- [ ] 性能基准测试报告

**发布目标**: 可作为短信中台基础

### Phase 4: 商业化探索（v3.0.0+）
- [ ] SaaS 版本（托管服务）
- [ ] 高级功能（AI 内容审核、智能路由）
- [ ] 企业级支持（SLA、技术支持）

---

## 8. 设计检查清单

### 开源友好度
- [ ] 开箱即用（5分钟内跑通 Demo）
- [ ] 文档完善（Quick Start、API 文档、FAQ）
- [ ] 示例代码（覆盖主流场景）
- [ ] 版本兼容性（清晰的升级指南）
- [ ] 问题响应（Issue 模板、自动回复）

### 技术可扩展性
- [ ] 清晰的 SPI 扩展点
- [ ] 配置化优于编码
- [ ] 向后兼容保证
- [ ] 依赖最小化
- [ ] 性能可配置（线程池、连接池）

### 平台化预留
- [ ] 多租户字段透传
- [ ] 业务线隔离支持
- [ ] 审计日志扩展点
- [ ] 计费数据暴露
- [ ] 管理接口标准化

---

## 9. 关键术语表

| 术语 | 定义 |
|------|------|
| Provider | 短信服务商（阿里云、天翼云等） |
| Template | 短信模板（含占位符） |
| Strategy | 负载均衡策略（随机、轮询、加权） |
| Circuit Breaker | 熔断机制（失败达到一定阈值暂停使用） |
| Tenant | 租户（平台化场景中的客户单位） |
| BizLine | 业务线（同一租户内的业务分组） |
| SPI | Service Provider Interface，Java 扩展机制 |
| Facade | 门面模式，统一对外接口 |

---

## 10. 参考资料

- [阿里云短信服务 API](https://help.aliyun.com/product/44282.html)
- [Spring Boot Starter 开发规范](https://docs.spring.io/spring-boot/docs/current/reference/html/features.html#features.developing-auto-configuration)
- [Micrometer 监控指标规范](https://micrometer.io/docs/concepts)
- [开源项目治理最佳实践](https://opensource.guide/)

---

*文档版本: v1.0*  
*创建时间: 2026-03-03*  
*维护者: Greyson*  
*更新记录: 初始版本*
