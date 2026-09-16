# 用户资料与 Redis 缓存服务

面向用户资料管理的后端练习，在注册登录之外实现联表查询、分页与 Redis 缓存，重点展示“查数据库、写入缓存、数据变更后失效”的处理过程。

> 仓库地址保留为 `Seeeeekiro/5`；这是学习实践项目，功能范围以源码为准。

## 已实现功能

- 用户注册、登录和分页查询。
- 通过 `sys_user LEFT JOIN user_info` 聚合用户详情。
- 详情缓存键为 `user:detail:{userId}`，TTL 为 10 分钟；缓存无法反序列化时删除并回源。
- 更新资料或删除用户后删除缓存；更新、删除方法带有事务注解。

## 技术栈

- Java 17、Spring Boot 3.5.11、Maven Wrapper。
- MyBatis-Plus、PostgreSQL。
- Spring Data Redis、Lombok、Hutool。

## 代码结构

```text
src/main/java/com/stu/helloserver/
├── controller/      用户接口
├── service/impl/    注册、登录与查询逻辑
├── mapper/          MyBatis-Plus 数据访问
├── entity/          数据库实体
├── dto/             请求对象
├── common/          统一响应和错误码
├── exception/       全局异常处理
└── config/          Web 配置
```

## 本地运行

### 1. 获取项目

```bash
git clone https://github.com/Seeeeekiro/5.git
cd 5
```

准备 JDK 17、PostgreSQL，以及可下载 Maven 依赖的网络。同时准备 Redis（当前默认 `127.0.0.1:6379`，数据库 0）。

### 2. 创建表

仓库原先未提供数据库初始化脚本。下面是根据当前实体及查询语句整理的**最小本地建表示例**，请在专用测试数据库中执行：

```sql
CREATE TABLE IF NOT EXISTS sys_user (
    id BIGSERIAL PRIMARY KEY,
    username VARCHAR(100) NOT NULL UNIQUE,
    password VARCHAR(255) NOT NULL
);
CREATE TABLE IF NOT EXISTS user_info (
    id BIGSERIAL PRIMARY KEY,
    user_id BIGINT NOT NULL UNIQUE REFERENCES sys_user(id),
    real_name VARCHAR(100),
    phone VARCHAR(32),
    address VARCHAR(255)
);
```

该示例不是生产迁移脚本；已有数据库请先核对字段和约束，不要直接覆盖现有表。

### 3. 配置连接

在 `src/main/resources/application.properties` 中设置自己的数据库连接；默认 URL 为 `jdbc:postgresql://localhost:5432/postgres`。推荐通过环境变量覆盖本地配置，避免将真实密码提交到仓库。

| 环境变量 | 用途 |
|---|---|
| `SPRING_DATASOURCE_URL` | PostgreSQL JDBC URL |
| `SPRING_DATASOURCE_USERNAME` | 数据库用户名 |
| `SPRING_DATASOURCE_PASSWORD` | 本地数据库密码 |
| `SPRING_DATA_REDIS_HOST` | Redis 主机 |
| `SPRING_DATA_REDIS_PORT` | Redis 端口 |
| `SPRING_DATA_REDIS_PASSWORD` | Redis 密码（实例需要时设置） |

### 4. 启动

Windows PowerShell：

```powershell
.\mvnw.cmd spring-boot:run
```

macOS / Linux：

```bash
sh mvnw spring-boot:run
```

默认接口地址为 `http://localhost:8080`。这是纯后端项目，不包含网页首页。

## 接口速查

| 方法 | 路径 | 说明 |
|---|---|---|
| POST | `/api/users` | 注册，JSON 包含 `username`、`password` |
| POST | `/api/users/login` | 登录，返回令牌字符串 |
| GET | `/api/users/{id}` | 根据用户 ID 查询用户名 |
| GET | `/hello` | 简单文本接口 |
| GET | `/api/users/page?pageNum=1&pageSize=5` | 用户分页查询 |
| GET | `/api/users/{id}/detail` | 查询用户及扩展资料，使用 Redis 缓存 |
| PUT | `/api/users/{id}/detail` | 更新资料，清理详情缓存 |
| DELETE | `/api/users/{id}` | 事务删除用户及资料，清理缓存 |

## 请求示例

以下命令适用于 macOS / Linux 的 shell；Windows 可使用 Postman 按相同路径和 JSON 请求体调用。

```bash
curl -X POST http://localhost:8080/api/users   -H 'Content-Type: application/json'   -d '{"username":"demo","password":"local-demo-only"}'

curl -X POST http://localhost:8080/api/users/login   -H 'Content-Type: application/json'   -d '{"username":"demo","password":"local-demo-only"}'
```

登录响应 `data` 已带有 `Bearer ` 前缀，后续请求直接把整个字符串放入 `Authorization`，不要重复添加前缀：

```bash
curl http://localhost:8080/api/users/1   -H 'Authorization: Bearer <登录返回的UUID>'
```

其中用户 ID 应替换为本地数据库中的实际值。

资料更新使用 `updateById`，请求 JSON 中必须传入真实的 `user_info.id`；路径里的 ID 是 `sys_user.id`，两者不能混淆。注册只创建 `sys_user` 记录，不会自动创建 `user_info`，联调前需自行插入对应资料行。

```json
{"id": 1, "realName": "演示用户", "phone": "", "address": ""}
```

建议验证：首次查询后出现 `user:detail:{userId}` 缓存；第二次查询命中缓存；更新资料后该键被删除，再次查询重新生成。

## 当前边界与后续完善

- 当前拦截器仅检查请求头是否以 `Bearer ` 开头，未持久化或验证 UUID 令牌，属于鉴权流程演示。
- 密码当前为明文保存及比对；正式使用前需实现密码哈希、输入校验及必要的授权控制。
- 分页接口返回用户实体，应改用不包含密码字段的响应 DTO；资料更新还需补齐新增资料路径及存在性检查。
- 数据库事务和 Redis 删除不属于同一事务，尚未提供严格的跨存储一致性保证。
- 当前配置中包含本地凭据字段。不要复用仓库中的示例或已有凭据；使用自己的环境变量，真实密钥如曾提交应先轮换。

## 验证与项目状态

本文依据仓库源码整理。尚未在本文档整理环境中连接数据库、启动服务或执行完整端到端测试；启动步骤和接口示例用于本地复现，不代表已经通过运行验收。

建议先按上述流程完成手动联调，再补充正常路径、非法输入和权限边界的自动化测试。

## 许可

当前仓库未提供 LICENSE 文件。公开可见不等于已授予开源使用许可；如需复用、分发或用于商业场景，请先联系仓库作者。
