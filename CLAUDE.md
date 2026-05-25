# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## 项目概述

苍穹外卖（sky-take-out）- 一款外卖管理平台，包含管理端后端 API + 微信小程序前端。后端基于 Spring Boot 2.7.3，前端基于 uni-app/微信小程序框架。

## 项目结构

```
sky-take-out/                # Maven 多模块父工程
├── pom.xml                  # 父 POM，统一管理依赖版本
├── sky-common/              # 公共模块：工具类、常量、异常、DTO 基础、JSON 序列化配置
├── sky-pojo/                # 实体类(Entity)、数据传输对象(DTO)、视图对象(VO)
└── sky-server/              # 主服务模块：Controller/Service/Mapper/Config
mp-weixin/                   # 微信小程序前端（uni-app 编译后产物）
nginx-1.20.2/                # Nginx 配置文件
```

## 后端架构

### 技术栈
- Spring Boot 2.7.3 + MyBatis-PageHelper + MySQL + Druid 连接池
- Redis（缓存/会话）+ WebSocket（订单实时通知）
- JWT 认证（Admin 端 token + 用户端 authentication）
- AliOSS 文件存储 + 微信支付 + 百度地图
- Knife4j (Swagger 2) 接口文档
- AOP 自动填充（INSERT/UPDATE 的创建时间、更新时间、创建人、更新人）

### 模块分层
```
Controller (admin / user / notify)
  └─ Service (impl)
       └─ Mapper (*.xml)
```

- `controller/admin/` - 管理端接口（需 JwtTokenAdminInterceptor 拦截）
- `controller/user/` - 用户端接口（需 JwtTokenUsernterceptor 拦截）
- `controller/notify/` - 支付回调接口
- `config/` - WebMvc、Redis、AliOSS、WebSocket 配置
- `aspect/` + `annotation/` - AOP 自动填充字段
- `task/` - 定时任务（订单超时取消等）
- `webSocket/` - WebSocket 服务端（订单消息推送）

### 关键配置
- 入口: `sky-server/src/main/resources/application.yml`
- 环境配置: `application-dev.yml`（数据库、Redis、OSS、微信、百度地图等通过环境变量覆盖）
- JWT 签名在 yml 中硬编码（admin-secret-key: itcast, user-secret-key: itheima）
- 驼峰命名自动开启，Mapper XML 位于 `classpath:mapper/*.xml`

### 业务模块
| 模块 | 说明 |
|------|------|
| Employee | 员工管理（登录、分页、增删改、启用/禁用） |
| Category | 分类管理（菜品/套餐分类） |
| Dish | 菜品管理（含口味关联 DishFlavor） |
| Setmeal | 套餐管理（含关联菜品 SetmealDish） |
| Order | 订单管理（用户下单、管理端统计、超时取消） |
| ShoppingCart | 购物车 |
| User | 用户（微信登录、地址簿） |
| Report | 数据统计报表（营业额、销量排行、用户统计） |
| Workspace | 工作台（今日数据概览） |
| Shop | 店铺营业状态开关 |

## 常用命令

```bash
# 编译整个项目
mvn clean package -DskipTests

# 仅编译 sky-server
mvn clean package -pl sky-server -am -DskipTests

# 运行后端服务
mvn spring-boot:run -pl sky-server

# 启动测试（项目暂无测试代码）
mvn test -pl sky-server
```

## 开发注意事项

1. **自动填充**: Mapper 方法上加 `@AutoFill(OperationType.INSERT/UPDATE)` 注解，AOP 会自动填充 createTime/updateTime/createUser/updateUser
2. **上下文线程隔离**: `BaseContext.setCurrentId()` / `getCurrentId()` 通过 ThreadLocal 存储当前操作人 ID
3. **JWT 认证**: 管理端使用 `token` 字段，用户端使用 `authentication` 字段，分别由两个 Interceptor 拦截
4. **分页**: 使用 PageHelper 插件，Service 层需先调用 `PageHelper.startPage()`
5. **API 文档**: 访问 `/doc.html` 查看 Knife4j 接口文档（管理端 `/admin/**`，用户端 `/user/**`）
6. **微信小程序**: `mp-weixin/` 是 uni-app 编译后的产物，源码应在项目外的 uni-app 工程中
