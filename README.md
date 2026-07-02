# 拾光笔记 · shiguang-notes

> 基于 Spring Cloud Alibaba 微服务架构 + Vue 3 的仿小红书社交内容平台（学习项目）

拾光笔记是一个仿小红书的全栈社交内容平台，涵盖用户登录注册（短信验证码）、图文/视频笔记发布、点赞收藏、两级评论、关注粉丝、全文搜索等核心功能。后端采用 Spring Cloud Alibaba 微服务架构，前端为 Vue 3 单页应用。

---

## ✨ 核心特性

- 🔐 **认证鉴权**：短信验证码登录注册（阿里云 SMS）+ Sa-Token 网关统一鉴权，`userId` 在服务间透传
- 📝 **笔记发布**：图文 / 视频笔记，详情查询走 Caffeine + Redis + MySQL **三级缓存**
- ❤️ **点赞收藏**：Redis Lua 脚本去重 + ZSet/Roaring Bitmap 存储，RocketMQ 异步计数
- 💬 **两级评论**：一级评论 + 二级回复，支持热度排序
- 👥 **关注关系**：关注 / 粉丝关系链
- 🔍 **全文搜索**：Elasticsearch 检索，Canal 监听 MySQL Binlog 近实时同步
- 🆔 **分布式 ID**：Snowflake + Segment 双方案（美团 Leaf 思路）
- 🗓️ **数据一致性**：XXL-Job 定时任务 + BufferTrigger 微批处理 + 数据对齐服务

---

## 🧱 技术栈

### 后端

| 层级 | 技术 | 版本 | 说明 |
|------|------|------|------|
| 语言 | Java | 17 | JDK 17 |
| 框架 | Spring Boot | 3.0.2 | 基础框架 |
| 微服务 | Spring Cloud | 2022.0.0 | 服务治理 |
| 注册/配置 | Spring Cloud Alibaba (Nacos) | 2022.0.0.0 | 注册发现 + 配置中心 |
| 网关 | Spring Cloud Gateway | - | 路由转发 + 鉴权 |
| 服务调用 | OpenFeign | - | 声明式 RPC |
| 认证 | Sa-Token | 1.38.0 | 权限认证 |
| 数据库 | MySQL | 8.0 | 关系型存储 |
| ORM | MyBatis + Druid | - | SQL 映射 + 连接池 |
| 缓存 | Redis (Lettuce) + Caffeine | - | 分布式缓存 + 本地缓存 |
| 消息队列 | RocketMQ | 4.9.4 | 异步解耦 |
| 搜索引擎 | Elasticsearch | 7.3.0 | 全文检索 |
| 对象存储 | MinIO / 阿里云 OSS | 8.2.1 / 3.17.4 | 文件存储 |
| 分布式 ID | Snowflake + Segment | - | 美团 Leaf 方案 |
| 定时任务 | XXL-Job | 2.4.1 | 分布式调度 |
| 数据同步 | Canal | 1.1.7 | MySQL Binlog 监听 |
| 微批处理 | BufferTrigger | 0.2.21 | 批处理聚合 |

### 前端

| 层级 | 技术 | 版本 |
|------|------|------|
| 框架 | Vue | 3.5.13 |
| 构建工具 | Vite | 6.0.11 |
| 路由 | Vue Router | 4.5.0 |
| 状态管理 | Pinia + persistedstate | 3.0.1 |
| CSS | Tailwind CSS | 4.0.6 |
| HTTP | Axios | 1.7.9 |
| 动画 | GSAP | 3.12.7 |

---

## 🏗️ 系统架构

```
用户浏览器
    │
    ▼
┌──────────────┐
│  Vue 3 前端   │  Vite + Pinia + Tailwind CSS + GSAP  (5173)
└──────┬───────┘
       │ /api/*  (Vite 代理)
       ▼
┌──────────────────────────────┐
│  API Gateway (8000) + Sa-Token │  路由转发 + 统一鉴权 + 注入 userId
└──────┬───────────────────────┘
       │
       ▼
┌──────────────────────────────────────────────────────┐
│              微服务集群 (Nacos 注册发现)               │
│  auth · user · note · comment · user-relation         │
│  oss · count · search · kv · distributed-id · data-align │
└──────┬──────────┬──────────┬──────────┬──────────────┘
       ▼          ▼          ▼          ▼
    MySQL 8    Redis     Elasticsearch  RocketMQ
               (Lettuce)      7.3        4.9.4
                            MinIO / OSS · Canal · XXL-Job
```

---

## 📦 微服务模块

每个业务服务拆分为 `*-api`（Feign 接口定义）与 `*-biz`（业务实现），接口与实现解耦。

| 模块 | 端口 | 职责 |
|------|------|------|
| `xiaoha-framework` | - | 公共框架：登录上下文传播、操作日志 AOP、Jackson、通用工具 |
| `xiaohashu-gateway` | 8000 | API 网关 + Sa-Token 鉴权 |
| `xiaohashu-auth` | 8080 | 认证服务：登录 / 注册 / 注销、短信验证码 |
| `xiaohashu-user` | 8082 | 用户服务 |
| `xiaohashu-oss` | 8081 | 对象存储：MinIO / 阿里云 OSS |
| `xiaohashu-note` | 8086 | 笔记服务：发布 / 详情（三级缓存）/ 点赞收藏 |
| `xiaohashu-comment` | 8093 | 评论服务：两级评论 |
| `xiaohashu-user-relation` | 8087 | 关注 / 粉丝关系 |
| `xiaohashu-count` | 8090 | 计数服务：点赞 / 收藏 / 评论数 |
| `xiaohashu-search` | 8092 | 搜索服务：ES 全文检索 + Canal 同步 |
| `xiaohashu-kv` | 8084 | KV 存储：笔记 / 评论内容 |
| `xiaohashu-distributed-id-generator` | 8085 | 分布式 ID 生成 |
| `xiaohashu-data-align` | 8091 | 数据对齐服务 |

---

## 📁 目录结构

```
code/
├── backend/
│   ├── xiaohashu.sql                 # 数据库初始化脚本
│   └── xiaohashu2/                   # 后端微服务（Maven 多模块）
│       ├── xiaoha-framework/         # 公共框架
│       ├── xiaohashu-gateway/
│       ├── xiaohashu-auth/
│       ├── xiaohashu-user/
│       ├── xiaohashu-note/
│       └── ...                       # 其余业务服务
├── frontend/
│   └── xiaohashu-vue3/               # Vue 3 前端
├── 拾光笔记实现思路.md                # 全栈实现思路
└── 拾光笔记微服务学习指南.md          # 微服务实践学习指南
```

---

## 🚀 快速开始

### 环境依赖

- **JDK 17**、**Maven 3.6+**
- **MySQL 8.0**、**Redis**、**Nacos 2.x**
- **RocketMQ 4.9.4**、**Elasticsearch 7.3.0**、**MinIO**
- （可选）Canal 1.1.7、XXL-Job 2.4.1

### 启动步骤

1. **初始化数据库**：导入 `backend/xiaohashu.sql`
2. **启动中间件**：Nacos、Redis、RocketMQ、Elasticsearch、MinIO
3. **导入 Nacos 配置**：各服务的 `bootstrap.yml` 中声明了 Nacos 配置的 dataId，按需在 Nacos 控制台新建对应配置
4. **修改连接信息**：在各服务配置中填入本地 MySQL / Redis / OSS / SMS 等连接信息
5. **启动后端**：建议顺序为 Gateway → 基础服务（auth / user / oss）→ 业务服务（note / comment / count / search …）
6. **启动前端**：

   ```sh
   cd frontend/xiaohashu-vue3
   npm install
   npm run dev
   ```

   前端默认运行在 http://localhost:5173 ，通过 Vite 代理 `/api/*` 到网关 8000。

> 📌 各服务的详细配置项与业务实现思路，见下方文档。

---

## 🧭 核心技术亮点

- **三级缓存**：笔记详情查询 = Caffeine 本地缓存 → Redis 分布式缓存 → MySQL，降低 DB 压力
- **点赞去重 + 异步计数**：Redis Lua 脚本保证原子去重 → 发送 RocketMQ 消息 → Count 服务用 BufferTrigger 聚合后批量落库
- ** Canal + ES 搜索同步**：定时轮询 Canal 监听 `t_note` 表 Binlog 变更，近实时同步到 Elasticsearch 索引
- **分布式 ID**：Snowflake（workerId 经 ZooKeeper 分配）+ Segment 双 Buffer 号段模式，按业务场景选用
- **用户身份传播**：网关注入 `userId` → 各服务 Filter 提取到 `TransmittableThreadLocal` → Feign 拦截器自动透传

---

## 📚 文档

- [拾光笔记 —— 全栈实现思路](./拾光笔记实现思路.md)
- [拾光笔记 —— 微服务实践学习指南](./拾光笔记微服务学习指南.md)

---

## ⚠️ 声明

本项目为个人学习实践项目，仿照小红书的产品形态实现，**仅用于学习交流微服务技术**，不提供任何真实服务，与小红书官方无任何关联。仓库内所有数据均为本地测试数据。
