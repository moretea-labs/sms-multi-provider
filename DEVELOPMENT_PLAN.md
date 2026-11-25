# 短信平台开发计划（更新版）

## 📋 项目概览

- **项目名称**: `sms-multi-provider`
- **定位**: Java 开源短信发送 SDK（发送能力，不做管理平台）
- **技术栈**: Spring Boot 2.7.18 + Java 17 + Maven 多模块
- **文档更新时间**: 2026-03-08
- **当前阶段**: MVP 已可用，进入「发布质量提升」阶段

---

## 🏗️ 当前真实状态（基于仓库代码）

### 模块结构

```text
sms-multi-provider/
├── sms-api              # 核心抽象与配置对象
├── sms-service          # 厂商实现（阿里云、天翼云）
├── sms-load-balance     # 负载均衡策略（随机/轮询/加权）
├── sms-facade           # 统一发送门面
├── sms-queue            # 队列重试支撑
├── sms-starter          # Spring Boot 自动配置
└── sms-example          # 示例工程
```

### 功能完成度

| 模块 | 状态 | 说明 |
|------|------|------|
| 多厂商抽象接口 | ✅ 完成 | `SmsProvider` 抽象稳定 |
| 阿里云发送 | ✅ 完成 | 生产级主实现 |
| 天翼云发送 | ✅ 完成 | 已实现 HTTP + HMAC-SHA256 签名 |
| 自动装配 | ✅ 完成 | `spring.factories` + `AutoConfiguration.imports` |
| 配置体系 | ✅ 基本完成 | `@ConfigurationProperties` + `@Validated` |
| 负载均衡 | ✅ 完成 | random / round-robin / weighted |
| 熔断降级 | ✅ 完成 | `ProviderManager` 管理可用集 |
| 重试机制 | ✅ 完成 | 立即重试 + 延迟重试 |
| 示例工程 | ✅ 完成 | `sms-example` 可演示接入流程 |
| 统一异常处理 | ⏳ 待完成 | 缺少 `@ControllerAdvice` |
| 单元测试体系 | ⏳ 待完成 | 当前 `src/test` 基本为空 |
| 监控指标 | ⏳ 待完成 | 未接入 Micrometer 关键指标 |
| 发布工程化 | ⏳ 待完成 | CI/CD、发布流程、质量门禁未完整 |

---

## 🎯 目标调整

从「补齐基础功能」切换为「可发布质量建设」。

### 新主目标

1. `v0.1.x` 稳定版本：可编译、可运行、可用示例可复现
2. `v0.2.0` 开发者友好版本：可观测、可扩展、文档可落地
3. `v0.3.0` 可发布版本：质量门禁、CI/CD、发布流程完整

---

## 🚀 分阶段开发计划

## Phase 1: 稳定性收敛（1 周）

**目标**: 清理发布阻塞项，形成稳定 MVP

### P0（阻塞发布）

- [x] 全仓库执行并通过：`mvn clean package`（2026-03-08 18:02 +08:00）
- [x] 增加全局异常处理：`@ControllerAdvice`（`sms-example` 已实现）
- [ ] 核心链路冒烟回归（阿里云、天翼云、加权策略、重试）
- [ ] 对 README 的快速开始进行一次从零验证

### 产出物

- 构建通过记录
- 异常码与错误响应规范
- MVP 冒烟测试记录

### 最新构建记录

- 结果：`BUILD SUCCESS`（7/7 模块通过）
- 命令：`mvn clean package`
- 时间：2026-03-08 18:02:19 +08:00
- 备注：存在外部仓库元数据拉取告警（DNS/缓存），但不影响本次构建成功

---

## Phase 2: 质量建设（1-2 周）

**目标**: 补齐可维护性与可观测性

### P1（发布前应完成）

- [ ] 核心单元测试：`ProviderManager`、`LoadBalancerManager`、`WeightedLoadBalancerStrategy`、`SmsTrulySender`
- [ ] 接入基础监控指标（Micrometer）
  - [ ] `sms_send_total`
  - [ ] `sms_send_success`
  - [ ] `sms_send_fail`
  - [ ] `sms_send_duration`
- [ ] 增加 Actuator 健康检查（至少 provider 可用性）
- [ ] 统一日志字段规范（traceId / provider / uniqueId）

### 产出物

- 可重复执行的测试套件
- Prometheus 可采集指标
- 健康检查端点说明

---

## Phase 3: 发布准备（1 周）

**目标**: 达到对外发布最低标准（v0.3.0）

### P1/P2

- [ ] CI/CD（GitHub Actions）
  - [ ] 编译
  - [ ] 单测
  - [ ] 代码风格检查
- [ ] 文档完善
  - [ ] `CHANGELOG.md`
  - [ ] `CONTRIBUTING.md`
  - [ ] Provider 扩展指南（SPI）
- [ ] 版本策略落地（SemVer）
- [ ] Maven Central 发布脚本与校验流程

### 产出物

- 自动化流水线
- 发布文档与发布 checklist
- 首个可对外标记版本

---

## Phase 4: 生态扩展（持续迭代）

**目标**: 提升厂商覆盖和扩展效率

### P2

- [ ] 腾讯云 Provider
- [ ] 华为云 Provider
- [ ] 通用 HTTP Provider Adapter（配置优先，减少编码量）
- [ ] 扩展点文档模板化（Provider / LoadBalancer / HistoryService）

---

## ✅ 不做范围（保持项目边界）

以下不纳入本仓库近期计划：

- Web 管理后台
- 多租户计费系统
- 用户/权限平台能力
- 报表可视化 UI

原则：本项目只做「短信发送 SDK 能力」。

---

## 📌 当下执行优先级（从今天开始）

1. 先完成 `mvn clean package` 的稳定通过与问题清单归档
2. 落地 `@ControllerAdvice` 统一异常处理
3. 补 4 个核心类单测，形成第一批质量门禁
4. 接入最小可用监控指标（计数 + 耗时）

---

## 📅 里程碑（更新）

| 版本 | 目标 | 时间建议 | 状态 |
|------|------|---------|------|
| v0.1.1 | 稳定 MVP（异常 + 构建 + 冒烟） | 1 周内 | 进行中 |
| v0.2.0 | 可观测 + 测试基础 | +1~2 周 | 未开始 |
| v0.3.0 | 可发布版本（CI/CD + 文档 + 发布流程） | +1 周 | 未开始 |
| v1.0.0 | 厂商生态与高级能力完善 | 后续迭代 | 规划中 |

---

## 风险与应对

- **风险**: 厂商接口变更导致行为漂移  
  **应对**: 增加 provider 级集成测试与 mock 回归

- **风险**: 缺测试导致重构回归  
  **应对**: 先覆盖核心路径，再扩大覆盖面

- **风险**: 发布流程不稳定  
  **应对**: 先做内部试发布，再外部发布

---

*版本: v2.0（基于 2026-03-08 仓库现状更新）*
