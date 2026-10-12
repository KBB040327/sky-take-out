# 苍穹外卖（sky-take-out）

> 🔗 **项目地址**：https://github.com/KBB040327/sky-take-out
>
> 技术栈：Java 8 · SpringBoot · MyBatis · MySQL · Redis · WebSocket · Spring Task · Apache POI

一个餐饮外卖平台的后端系统，分为**管理端**（商家/员工使用）和**用户端**（顾客点餐使用）两套接口。

> **项目说明**：本项目参考开源教程《苍穹外卖》实现，主要用于系统学习 SpringBoot 企业级开发的完整流程（分层架构、AOP、缓存、消息推送、定时任务、第三方支付对接等）。
> 在此基础上我做了一些自己的改造，见下方「我的改造」一节。

---

## 技术栈

| 分类 | 技术 |
|---|---|
| 语言 / 框架 | Java 8、SpringBoot 2.7、MyBatis |
| 数据库 | MySQL 8（Druid 连接池）|
| 缓存 | Redis |
| 实时通信 | WebSocket |
| 定时任务 | Spring Task |
| 工具 | Lombok、Knife4j（接口文档）、Apache POI（Excel 导出）、阿里云 OSS、微信支付 |

---

## 功能模块

### 管理端

| 模块 | 说明 |
|---|---|
| 员工管理 | 登录、分页查询、新增、编辑、启用/禁用 |
| 分类管理 | 菜品/套餐分类的增删改查 |
| 菜品管理 | 菜品增删改查、起售停售、口味管理、图片上传（OSS）|
| 套餐管理 | 套餐增删改查、起售停售、关联菜品 |
| 订单管理 | 订单搜索、各状态订单统计、接单/拒单/取消/派送/完成 |
| 数据统计 | 营业额统计、用户统计、订单统计、销量排名（Excel 导出）|
| 工作台 | 今日运营数据概览 |
| 店铺管理 | 营业状态设置 |

### 用户端

| 模块 | 说明 |
|---|---|
| 微信登录 | 小程序授权登录，JWT 签发与校验 |
| 浏览 | 分类、菜品、套餐浏览与搜索 |
| 购物车 | 加购、减购、清空、查看 |
| 下单 | 提交订单、订单支付、历史订单查询、取消、再来一单 |
| 地址簿 | 收货地址的增删改查与默认地址 |
| 支付回调 | 微信支付结果通知处理 |

---

## 快速开始

### 1. 环境要求

- JDK **8**
- Maven 3.6+
- MySQL **8.x**（3306）
- Redis（6379）

### 2. 初始化数据库

```sql
CREATE DATABASE sky_take_out DEFAULT CHARACTER SET utf8mb4;
```

导入表结构与初始数据（SQL 文件在项目资料中），然后启动项目。

### 3. 配置连接信息（重要）

**所有密码和密钥都通过环境变量注入，没有写在配置文件里。**

```powershell
# Windows PowerShell
setx MYSQL_PASSWORD      "你的MySQL密码"
setx REDIS_PASSWORD      "你的Redis密码"
setx ALIYUN_OSS_ENDPOINT "oss-cn-beijing.aliyuncs.com"
setx ALIYUN_OSS_ACCESS_KEY_ID     "你的AccessKeyId"
setx ALIYUN_OSS_ACCESS_KEY_SECRET "你的AccessKeySecret"
setx ALIYUN_OSS_BUCKET   "你的Bucket名"
setx WECHAT_APPID        "你的小程序AppId"
setx WECHAT_SECRET       "你的小程序密钥"
setx WECHAT_MCHID        "你的商户号"
setx WECHAT_MCH_SERIAL_NO "你的商户证书序列号"
setx WECHAT_API_V3_KEY   "你的APIv3密钥"
setx WECHAT_NOTIFY_URL       "支付回调地址"
setx WECHAT_REFUND_NOTIFY_URL "退款回调地址"
```

```bash
# macOS / Linux：用 export 代替 setx
```

> **设置后需要重启 IDEA 或终端才会生效。**
> 只做本地开发的话，前两个（MySQL、Redis）是必须的，其他可以留空。

### 4. 启动

```bash
mvn spring-boot:run -pl sky-server
```

服务端口：**8080**
接口文档：`http://localhost:8080/doc.html`

---

## 关键技术实现

### 1. 公共字段自动填充（AOP + 自定义注解）

`create_time` / `update_time` / `create_user` / `update_user` 这类字段每个 insert/update 都要写一遍。
通过自定义注解 `@AutoFill` + AOP 切面，在方法执行前统一反射填充，业务代码里完全不用管。

### 2. 菜品缓存

用 Redis 缓存菜品数据，key 按分类维度组织。
当菜品发生增删改时，**主动清理相关缓存**（而不是等过期），保证数据一致性。

### 3. 来单提醒（WebSocket）

- 商家端与后端建立 WebSocket 长连接，按 `sid`（店铺id）区分不同商家
- 用户支付成功后，通过 WebSocket **主动推送**新订单消息给对应商家，前端弹窗 + 语音播报
- 用户点击「催单」时，同样推送提醒给商家

### 4. 订单超时处理（Spring Task）

定时任务每分钟扫描一次：
- **超时未支付**的订单 → 自动取消、回补库存、记录取消原因
- **一直未接单**的订单 → 自动取消、退款

### 5. 营业数据统计

用 `GROUP BY` + 条件聚合做统计，并通过 Apache POI 导出 Excel 报表。
重点是处理「日期区间」和「各种状态」的聚合口径。

### 6. 微信支付对接

微信小程序登录 → 下单 → 调用微信支付统一下单 → 接收支付回调 → 更新订单状态。
涉及 JWT 令牌校验、支付回调验签、幂等处理。

---

## 我的改造

> 这一节记录我在教程基础上自己做的改动，以及过程中踩到的坑。

- [ ] 待补充：例如「把定时轮询改成延迟队列处理超时订单」
- [ ] 待补充：例如「部署到云服务器时遇到的问题及排查过程」

---

## 项目结构

```
sky-take-out/
├── sky-common/         公共模块（常量、工具类、异常、结果封装）
├── sky-pojo/           实体模块（Entity、DTO、VO）
└── sky-server/         业务模块
    └── src/main/java/com/sky/
        ├── annotation/ 自定义注解（@AutoFill）
        ├── aspect/     AOP 切面（公共字段填充、缓存清理）
        ├── config/     配置类（WebMvc、Redis、OSS、WebSocket、Knife4j）
        ├── constant/   常量
        ├── context/    ThreadLocal 上下文
        ├── controller/ 接口层（admin 管理端 / user 用户端）
        ├── exception/  全局异常处理
        ├── handler/    消息处理器
        ├── interceptor/ 拦截器（JWT 校验）
        ├── mapper/     数据访问层
        ├── service/    业务逻辑层
        ├── task/       定时任务
        ├── utils/      工具类
        └── websocket/  WebSocket 服务
```

---

## 安全说明

本项目**不在代码中保存任何密钥**，所有敏感配置通过环境变量注入。
如果你 clone 之后想要跑起来，请按上面「配置连接信息」一节设置环境变量。

---

## 后续计划

- [ ] 用 RabbitMQ 延迟队列替代定时任务轮询处理超时订单
- [ ] 引入 Spring Cache 注解式缓存
- [ ] 补充单元测试
- [ ] 用 Docker Compose 一键部署
