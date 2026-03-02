# 需求实施计划

- [ ] 1. Edge Gateway 与版本兼容控制（需求5-AC1,3）
  - 实现 `/api/tv/*` 聚合路由，注入 `x-device-id`, `x-oriontv-version`, `Accept-Language` 至内部上下文，并接入速率限制配置。
  - 编写版本兼容矩阵校验逻辑，对不受支持版本返回 426，响应含 `minSupportedVersion`, `downloadUrl`, `message`，并在响应头输出 `Sunset/Deprecation`。
  - 集成错误规范化模块，将 Edge 层捕获的所有异常转为统一 `error.code`, `message`, `details`。
- [ ]* 1.1 Edge Gateway 合同测试
  - 使用 Supertest/Pact 覆盖版本校验、错误映射与 `Sunset` 头部，确保 426/429/4xx 响应格式稳定。

- [ ] 2. Schema Adapter & Catalog API 对齐（需求1-AC1/2/3/4, 设计-数据模型, 正确性1）
  - 扩展 `ContentMapper` 以输出 `heroMedia`, `availability`, `watchState`, `schemaVersion`，并在内部缺失字段时填充 `null`。
  - 新增 `/api/tv/catalog` 与 `/api/tv/catalog/{id}` 控制器，支持 `type`, `page`, `locale` 查询参数与 400 错误代码 `INVALID_CONTENT_TYPE`。
  - 在 Content Service 层实现 fallback schema 处理，确保内部新增字段能通过 `schemaVersion` 保护并与缓存层（Kvrocks）集成。
- [ ]* 2.1 Catalog 契约 & 属性测试
  - 基于 JSON Schema 生成契约测试，补充属性测试以验证 `availability.geo` 与 `schemaVersion` 在任意内容类型下均存在或为空数组。

- [ ] 3. Discovery/Search/Highlights 统一（需求2-AC1/2/3/4）
  - 扩展 search 聚合器，合并豆瓣/Youtube/网盘/短剧/Bangumi 结果并输出 `sections[].sourceBadge`, `language`, `sourceStatus` 字段。
  - 将分类与高亮 API 暴露为 `/api/tv/categories` 与 `/api/tv/highlights`，支持 `Accept-Language` 本地化与 hero trailer 过期控制。
  - 实现上游失败的 degrade 逻辑，捕获 timeout/错误并将 `section.sourceStatus` 标记为 `DEGRADED` 同时记录 metrics。
- [ ]* 3.1 搜索与分类集成测试
  - 利用 Supertest + mock 上游源验证 sections 拆分、本地化名称、`sourceStatus` degrade 行为与 highlights 条数下限。

- [ ] 4. Device Auth & Profile 投影（需求3-AC1/2/3/4, 设计-Device Session, 正确性3）
  - 在 Auth Service 中实现 `POST /api/tv/auth/device` 与 `POST /api/tv/auth/token` 流程，持久化 `device_code`, `user_code`, `interval`, 状态机并触发 OIDC Provider。
  - 新增 `device_session` kv/表结构，保存 `access_token`, `refresh_token`, `entitlements`, `maturityRatings`, `householdId`, 并实现 `GET /api/tv/profile`。
  - 添加登录失败计数与限流，10 分钟内 5 次失败返回 429 `retryAfter`，并与 Edge Gateway 速率限制联动。
- [ ]* 4.1 设备授权集成测试
  - 编排模拟设备轮询流程，验证 428 轮询响应、批准后的 token 下发、速率限制与 profile 投影完整性。

- [ ] 5. Playback Session & Fallback 控制（需求4-AC1/2/3/4, 设计-Playback Session, 正确性2/4）
  - 构建 `PlaybackMapper` 与 Session Service，处理 `POST /api/tv/playback/session` 输入，选择最优源并返回 `sessionId`, `sources`, `drm`, `bufferProfile`。
  - 实现 `PATCH /api/tv/playback/session/{sessionId}` Enricher，记录 `position`, `playbackState`, `bandwidth`，确保 200ms 内写入 Kvrocks，并强制 120s 心跳与 3 次缺失失效。
  - 添加播放源异常与 fallback 生成器，500ms 内返回 `fallbackSource`, `proxyUrl`, `directPlayable=false` 且遵守 deterministic `sourcePreference` 权重。
- [ ]* 5.1 Playback 幂等与 fallback 测试
  - 编写 property 测试验证相同 `sessionId+sequence` PATCH 幂等，以及当主源 403/CORS 时 fallback 顺序与代理 URL 一致。

- [ ] 6. Telemetry Dispatcher & 版本公告（需求5-AC2/3/4, 设计-Telemetry, 正确性5）
  - 构建 `TelemetryMapper` 校验单事件 2KB/批量 100 条限制，将 `POST /api/tv/telemetry` 数据写入队列（Kafka/NATS/Redis Stream）。
  - 实现队列不可用时的缓冲策略，返回 202 `status=buffered`, 将事件写入持久队列 15 分钟并定期 flush。
  - 在所有即将废弃的 `/api/tv/*` 响应添加 `Sunset`/`Deprecation` 头部，并记录版本公告时间戳。
- [ ]* 6.1 Telemetry 负载与缓冲测试
  - 使用 Jest/Mock 队列验证 2KB 限制、1 秒入队 SLA、缓冲状态与过期清理；可选进行 k6 压测脚本自检。

- [ ] 7. OrionTV 客户端适配（前端依赖于上述 API 完成后进行，需求1-5 全面覆盖）
  - 更新 React Native 数据层以消费新的 catalog/search/profile/playback API，建立与 `schemaVersion` 对齐的类型定义。
  - 调整设备登录 UI 与轮询逻辑，展示 428/429 错误，以及播放页对 fallback/proxy 提示处理。
  - 接入 telemetry 上报 SDK，批量发送事件、附带 `x-oriontv-version`，并处理 202 `status=buffered` 情况。
- [ ]* 7.1 OrionTV 端到端 UI 自动化
  - 编写 Detox/Appium 脚本覆盖登录、浏览、播放、遥测上报主路径，确保客户端与后端契约同步。

- [ ] 8. 检查点 - 确保所有测试通过,如有疑问请询问用户
  - 运行全部单元、属性、集成测试；验证 Edge/Schema/Auth/Playback/Telemetry/Client 端链路并回归关键 API。
