# Requirements Document

## Introduction

为实现 LunaTV 增强版后端与 OrionTV 安卓电视前端的深度集成，需要定义一致的数据结构、认证机制与 API 契约。本需求文档采用 EARS 规范，确保双方系统在内容分发、交互及安全策略上达到完全匹配。

## Glossary

- **LunaTV Backend**: 当前使用 Next.js + Node.js 的聚合内容与播放服务端。
- **OrionTV Frontend**: 基于 React Native TVOS/Expo 的 Android/Apple TV 客户端。
- **Content Item**: 统一的影视或直播内容实体，包含描述、封面、可播放源等信息。
- **Schema Adapter**: LunaTV 中负责转换内部数据模型为 OrionTV 所需响应结构的中间层。
- **Device Session**: OrionTV 设备与 LunaTV 间的认证状态，含访问令牌、刷新令牌等。
- **Telemetry Bus**: 接收 OrionTV 端遥控交互、播放质量与错误事件的 LunaTV 监控通道。

## Requirements

### Requirement 1: Unified Content Catalog Contract

**User Story:** 作为使用 OrionTV 的观众，我希望所有影视、短剧、直播内容在电视端展示统一且完整的元数据，以便快速浏览与选择。

#### Acceptance Criteria

1. WHEN OrionTV 调用 `GET /api/tv/catalog?type={contentType}&page={page}&locale={locale}`, the LunaTV backend SHALL respond within 2 seconds with a JSON array whose objects包含字段 `id`, `type`, `title`, `synopsis`, `poster`, `heroMedia`, `resolution`, `tags`, `releaseYear`, `rating`, `availability`。
2. WHEN OrionTV 请求 `GET /api/tv/catalog/{id}`, the LunaTV backend SHALL include `episodes`, `sources`, `credits`, `watchState`, `aiHighlights`, `contentAdvisory` 字段，缺失数据以 `null` 表示。
3. WHILE LunaTV backend 存储新增元数据字段, the schema adapter SHALL preserve backward-compatible responses by填充未知字段为 `null` 并附带 `schemaVersion`。
4. IF OrionTV 提交未定义的 `contentType`, the LunaTV backend SHALL return HTTP 400 with error code `INVALID_CONTENT_TYPE` 与人类可读信息。

### Requirement 2: Discovery & Search Alignment

**User Story:** 作为希望快速找到资源的观众，我需要在 OrionTV 中使用统一的搜索与分类能力，并获取明确的来源状态。

#### Acceptance Criteria

1. WHEN OrionTV 调用 `GET /api/tv/search?q={keyword}&filters={...}`, the LunaTV backend SHALL 混合返回豆瓣、YouTube、网盘、短剧与 Bangumi 各类型结果，并以 `sections[]` 格式区分来源，确保每个结果含 `sourceBadge` 与 `language`。
2. WHEN OrionTV 请求 `GET /api/tv/categories`, the LunaTV backend SHALL 返回至少 12 个分类，包含 `slug`, `displayName`, `type`, `thumbnail`, `contentCount`, `isFeatured`，并根据 `Accept-Language` 头部提供本地化名称。
3. IF 任一上游采集源超时或失败, the LunaTV backend SHALL 标记对应 `section.sourceStatus=DEGRADED` 并仍返回其他成功的 section。
4. WHEN OrionTV 请求 `GET /api/tv/highlights`, the LunaTV backend SHALL 提供不少于 5 条 Hero 内容，每条含可播放预告片 URL 与过期时间戳。

### Requirement 3: Authentication & Authorization Federation

**User Story:** 作为登录用户，我希望通过 OrionTV 安全获取授权并同步个人偏好与权益。

#### Acceptance Criteria

1. WHEN OrionTV 调用 `POST /api/tv/auth/device` 并提供 `client_id`, the LunaTV backend SHALL 返回 `device_code`, `user_code`, `verification_uri_complete`, `expires_in` (600s) 与 `interval` 字段。
2. WHEN OrionTV 以 `device_code` 轮询 `POST /api/tv/auth/token`, the LunaTV backend SHALL 在授权完成时返回 `access_token`, `refresh_token`, `expires_in`, `scope`, `token_type=bearer`，并在未授权时返回 428 指示继续轮询。
3. WHILE Device Session 处于有效状态, the LunaTV backend SHALL 在 `GET /api/tv/profile` 响应中包含 `userId`, `householdId`, `entitlements`, `maturityRatings`, `continueWatching[]`。
4. IF 同一设备在 10 分钟内失败登录 5 次, the LunaTV backend SHALL 返回 HTTP 429 并在响应体中提供下一次允许尝试的 `retryAfter` 秒数。

### Requirement 4: Playback Session Continuity

**User Story:** 作为 OrionTV 观众，我希望播放过程中能自动选择可用源、保留进度并在异常时快速切换。

#### Acceptance Criteria

1. WHEN OrionTV 调用 `POST /api/tv/playback/session` 提供 `contentId`, `sourcePreference`, `deviceCapabilities`, the LunaTV backend SHALL 返回 `sessionId`, `sources[]`, `drm`, `startPosition`, `bufferProfile`，并选择最优源。
2. WHEN OrionTV 发送 `PATCH /api/tv/playback/session/{sessionId}` 包含 `position`, `playbackState`, `bandwidth`, the LunaTV backend SHALL 在 200ms 内持久化数据用于断点续播。
3. WHILE Session 处于活跃状态, the LunaTV backend SHALL 要求 OrionTV 至少每 120 秒发送一次 `heartbeat` 并在 3 个心跳缺失后自动失效会话。
4. IF 播放源返回 HTTP 403 或 CORS 错误, the LunaTV backend SHALL 在 500ms 内提供 `fallbackSource` 与 `proxyUrl`, 并设置 `directPlayable=false`。

### Requirement 5: Telemetry & Version Compatibility

**User Story:** 作为平台运维人员，我需要监控 OrionTV 客户端行为并确保客户端版本与 API 兼容。

#### Acceptance Criteria

1. WHEN OrionTV 在请求头中携带 `x-oriontv-version`, the LunaTV backend SHALL 根据兼容矩阵校验版本并在版本不受支持时返回 HTTP 426，响应体包含 `minSupportedVersion`, `downloadUrl`。
2. WHEN OrionTV 调用 `POST /api/tv/telemetry` 提交事件数组, the LunaTV backend SHALL 接受单事件最大 2KB 的 JSON 并在 1 秒内写入 Telemetry Bus 队列。
3. WHILE LunaTV backend 计划废弃任意 API, the LunaTV backend SHALL 在响应头添加 `Sunset` 与 `Deprecation` 信息, 提前至少 30 天公告。
4. IF Telemetry Bus 暂时不可用, the LunaTV backend SHALL 返回 HTTP 202 并将事件缓存于持久队列 15 分钟，同时在响应体中提供 `status=buffered`。
