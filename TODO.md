# SMS-Multi-Provider 开发 TODO（对齐版）

> **定位**: Java 开源 SDK，专注短信发送能力，不做业务管理平台
> **当前主线**: 从 MVP 可用进入发布质量建设
> **更新时间**: 2026-03-08

---

## Phase 1: 稳定性收敛（v0.1.1，1 周）

**目标**: 形成稳定可复现的 MVP

### P0 - 阻塞发布
- [x] 全仓库 `mvn clean package` 通过并记录构建结果（2026-03-08 18:02 +08:00）
- [x] 实现全局异常处理（`@ControllerAdvice`，`sms-example` 已落地）
- [ ] 完成核心链路冒烟回归（阿里云 / 天翼云 / 加权 / 重试）
- [ ] 从零验证 README 快速开始可跑通

### 已完成（现状）
- [x] `sms-starter` 启动模块
- [x] 阿里云 Provider
- [x] 天翼云 Provider（HTTP + HMAC-SHA256）
- [x] 自动装配（`spring.factories` + `AutoConfiguration.imports`）
- [x] 配置类 `@ConfigurationProperties` + `@Validated`
- [x] 负载均衡（random / round-robin / weighted）
- [x] 重试机制（立即重试 + 延迟重试）
- [x] 示例工程 `sms-example`

---

## Phase 2: 质量建设（v0.2.0，1-2 周）

**目标**: 补齐可维护性与可观测性

### P1 - 发布前应完成
- [ ] 核心单元测试
  - [ ] `ProviderManager`
  - [ ] `LoadBalancerManager`
  - [ ] `WeightedLoadBalancerStrategy`
  - [ ] `SmsTrulySender`
- [ ] Micrometer 指标
  - [ ] `sms_send_total`
  - [ ] `sms_send_success`
  - [ ] `sms_send_fail`
  - [ ] `sms_send_duration`
- [ ] Actuator 健康检查（provider 可用性）
- [ ] 统一日志字段规范（`traceId` / `provider` / `uniqueId`）

---

## Phase 3: 发布准备（v0.3.0，1 周）

**目标**: 达到可对外发布最低标准

### P1/P2
- [ ] GitHub Actions CI/CD
  - [ ] 编译
  - [ ] 单测
  - [ ] 代码风格检查
- [ ] 文档补齐
  - [ ] `CHANGELOG.md`
  - [ ] `CONTRIBUTING.md`
  - [ ] Provider SPI 扩展指南
- [ ] 版本策略（SemVer）
- [ ] Maven Central 发布流程与校验

---

## Phase 4: 生态扩展（v1.0.0+，持续）

### P2
- [ ] 腾讯云 Provider
- [ ] 华为云 Provider
- [ ] 通用 HTTP Provider Adapter（配置驱动）
- [ ] 扩展点模板化文档（Provider / LoadBalancer / HistoryService）

---

## 不做清单（范围控制）

- [ ] Web 管理后台
- [ ] 多租户计费系统
- [ ] 用户/权限平台能力
- [ ] 统计报表 UI

原则：本仓库只提供 SDK 发送能力。
