# 数据安全保护伞 (Data Security Umbrella)

数据安全保护伞面向企业敏感数据治理：以**数据库安全**（MySQL / ClickHouse）为主线，覆盖数据源与资产、分类分级策略、批量离线扫描（规则 + AI）及大盘概览；首页同时预留 **API / MQ 安全**模块入口，管理中心负责账号与角色。

## 项目概述

系统采用 **前端 + Manager（管理 API）+ Worker（扫描执行）** 拆分：Manager 对接前端配置与查询；Worker 消费 RabbitMQ 任务、抽样扫描库表列，按策略做规则命中与 Ollama LLM 评估，结果回写资产并落快照。推荐落地顺序见文末「交互界面展示」。

## 核心功能

### 1. 数据库安全（已落地）
- **数据源**：登记 MySQL / ClickHouse 实例，测试连通性
- **数据资产**：按实例 → 库 → 表 → 列下钻；规则扫描与 AI 扫描标签、敏感等级、样例与人工复核
- **策略管理**：按引擎维护 `policy_code`、分类规则、规则表达式与 AI 规则，支持规则检测 / AI 流式试跑
- **批量离线扫描**：周期或立即执行；任务与实例共用表，以 `database_type` 区分引擎；可停止实例并失效 Redis 版本号以丢弃在途 MQ 消息
- **扫描结果**：表级 / 字段级快照（默认约 180 天，以表 TTL 为准）
- **概览**：MySQL / ClickHouse 分路由大盘，支持刷新统计

### 2. 管理中心
- yml 预置超级管理员登录
- 账号管理（`/admin-center/account`）与角色说明（`/admin-center/role`，内置 ADMIN / OPERATOR）
- 产品权限：DATABASE / API / MQ

### 3. 其它产品模块（入口已预留）
- **API 安全保护伞**、**MQ 安全保护伞**：首页可进入，能力持续扩展中

## 技术架构

### 前端技术栈
- **框架**: React 19 + TypeScript
- **UI**: Ant Design 6、Ant Design Icons
- **路由**: React Router 7
- **构建**: Vite 8

### 后端技术栈
- **框架**: Spring Boot 3.2（Java 17）
- **模块**: `server-core` / `server-manager`（默认 8080）/ `server-worker`（默认 8081）
- **ORM**: MyBatis Plus 3.5
- **元数据库**: MySQL 8
- **扫描目标**: MySQL、ClickHouse（JDBC）
- **消息与缓存**: RabbitMQ、Redis
- **规则引擎**: Aviator；**AI**: LangChain4j + Ollama
- **其它**: Lombok、Fastjson2

## 系统架构

```
┌──────────────────────────────────────────────────────────────────┐
│                         前端 (React + Vite)                       │
│  首页 │ 数据库安全(数据源/资产/策略/批量任务/概览) │ 管理中心 │ API/MQ │
└─────────────────────────────┬────────────────────────────────────┘
                              │ HTTP POST API
┌─────────────────────────────▼────────────────────────────────────┐
│                    Manager (Spring Boot :8080)                    │
│         账号/策略/数据源/资产查询 │ 任务下发 │ 概览统计              │
└───────────────┬─────────────────────────────┬────────────────────┘
                │ RabbitMQ                    │ Redis（会话/任务版本等）
┌───────────────▼─────────────────────────────┴────────────────────┐
│                    Worker (Spring Boot :8081)                     │
│     离线规则扫描 │ 离线 AI 扫描（独立线程池）│ 快照写入              │
└───────────────┬─────────────────────────────┬────────────────────┘
                │                             │
        MySQL / ClickHouse 业务库      元数据 MySQL + 快照表
                │
           Ollama（可选，AI 规则）
```

## 接口开发规范

### ⚠️ 重要限制

本系统后端接口严格遵循以下规范，所有开发者必须遵守：

1. **禁止使用 RESTful 风格**
   - 所有接口统一使用 `POST` 方法
   - 不得使用 `GET`、`PUT`、`DELETE` 等 RESTful HTTP 方法

2. **禁止将参数写在 URL 路径中**
   - 所有参数必须放在请求体（Request Body）中
   - 不得使用路径参数（如 `/api/resource/{id}`）
   - 不得使用查询参数（如 `/api/resource?id=123`）

3. **统一返回格式**
   - 所有接口返回统一的 `Result` 对象
   - 包含 `code`（状态码）、`message`（消息）、`data`（数据）字段

### 接口示例

#### 正确的接口设计
```java
// ✅ 正确：使用 POST 方法，参数在请求体中
@PostMapping("/get-by-id")
public Result<DataSource> getById(@RequestBody Map<String, Object> params) {
    Long id = Long.parseLong(params.get("id").toString());
    DataSource dataSource = dataSourceService.getById(id);
    return Result.success(dataSource);
}
```

#### 错误的接口设计
```java
// ❌ 错误：使用 GET 方法
@GetMapping("/{id}")
public Result<DataSource> getById(@PathVariable Long id) {
    // ...
}

// ❌ 错误：使用 PUT 方法
@PutMapping
public Result<Boolean> update(@RequestBody DataSource dataSource) {
    // ...
}

// ❌ 错误：使用 DELETE 方法
@DeleteMapping("/{id}")
public Result<Boolean> delete(@PathVariable Long id) {
    // ...
}

// ❌ 错误：使用查询参数
@GetMapping("/page")
public Result<IPage<DataSource>> getPage(
    @RequestParam Integer current,
    @RequestParam Integer size) {
    // ...
}
```

### 前端调用示例

```typescript
// ✅ 正确：使用 POST 方法，参数在请求体中
const response = await request('/api/data-source/get-by-id', {
    method: 'POST',
    body: JSON.stringify({ id: 123 }),
});

// ❌ 错误：使用 GET 方法，参数在 URL 中
const response = await request('/api/data-source/123', {
    method: 'GET',
});
```

### 规范原因

1. **安全性**：所有接口使用 POST 方法，减少攻击面
2. **一致性**：统一的接口风格，便于维护和理解
3. **扩展性**：便于接口版本控制和参数扩展
4. **监控**：便于日志记录和请求追踪

## 快速开始

### 环境要求
- Node.js 18+
- Java 17+
- Maven 3.8+
- MySQL 8.0+（元数据）
- Redis、RabbitMQ（任务与会话/版本校验）
- 可选：Ollama（AI 规则评估）、可连通的 MySQL / ClickHouse 业务实例（扫描目标）

### 安装步骤

1. **克隆项目**
   ```bash
   git clone https://github.com/JavaScalaDeveloper/data-sec-umbrella.git
   cd data-sec-umbrella
   ```

2. **配置并启动后端**
   ```bash
   cd data-sec-umbrella-server
   # 分别配置 manager / worker 的 application.yml：MySQL、Redis、RabbitMQ、超级管理员、Ollama 等
   mvn clean install
   mvn -pl data-sec-umbrella-server-manager spring-boot:run
   # 另开终端启动 Worker（执行离线扫描，默认 8081）：
   mvn -pl data-sec-umbrella-server-worker spring-boot:run
   ```

3. **启动前端**
   ```bash
   cd data-sec-umbrella-front
   npm install
   npm run dev
   ```

4. **访问**
   - 前端: http://localhost:3000（以 Vite 控制台为准）
   - Manager API: http://localhost:8080

推荐操作路径：首页 → 管理中心配置账号 → 策略 / 数据源 / 资产 → 批量扫描 → 概览（详见下方「交互界面展示」）。

## 使用指南

1. **管理中心**：超级管理员登录后创建业务账号，分配 ADMIN / OPERATOR 与产品权限  
2. **策略**：按 MySQL / ClickHouse 配置分类规则与 AI 规则，并做规则检测  
3. **数据源与资产**：登记实例并扫描，下钻查看库表列敏感标签  
4. **批量任务**：配置采样与 `policy_code`，立即运行或周期调度；在实例页查看快照  
5. **概览**：按引擎查看统计并刷新指标  

## 项目结构

```
data-sec-umbrella/
├── data-sec-umbrella-front/              # 前端（Vite + React）
│   ├── public/images/交互图/             # README 用交互截图
│   ├── src/
│   │   ├── pages/
│   │   │   ├── Home.tsx                  # 产品首页
│   │   │   ├── AdminCenter.tsx           # 管理中心（账号/角色）
│   │   │   ├── DatabaseSecurity.tsx      # 数据库安全（策略/数据源等）
│   │   │   ├── data-asset/               # MySQL / ClickHouse 资产
│   │   │   ├── task-management/          # 批量离线任务与实例
│   │   │   ├── overview/                 # 大盘概览
│   │   │   ├── ApiSecurity.tsx           # API 模块入口
│   │   │   └── MqSecurity.tsx            # MQ 模块入口
│   │   ├── routes/                       # 路由
│   │   └── services/                     # API 封装
│   └── package.json
├── data-sec-umbrella-server/             # 后端多模块
│   ├── data-sec-umbrella-server-core/    # 实体、Mapper、Service、规则/AI 检查器
│   ├── data-sec-umbrella-server-manager/ # 管理端 API（默认 8080）
│   └── data-sec-umbrella-server-worker/  # 扫描 Worker（默认 8081）
└── README.md
```

## 贡献指南

我们欢迎社区贡献！请遵循以下步骤：

1. Fork 本项目
2. 创建特性分支 (`git checkout -b feature/AmazingFeature`)
3. 提交更改 (`git commit -m 'Add some AmazingFeature'`)
4. 推送到分支 (`git push origin feature/AmazingFeature`)
5. 提交 Pull Request

## 许可证

本项目采用 [Apache 2.0 许可证](LICENSE)。

## 联系我们

- 项目主页: https://github.com/JavaScalaDeveloper/data-sec-umbrella
- 问题反馈: https://github.com/JavaScalaDeveloper/data-sec-umbrella/issues
- 邮箱: 544789628@qq.com

## 交互界面展示

交互截图位于 `data-sec-umbrella-front/public/images/交互图/`（按「管理中心」「数据安全保护伞」分子目录）。下面按**推荐落地顺序**串联主要页面；图片路径为相对仓库根目录，便于在 GitHub / 本地预览。

### 0. 产品首页：选择模块入口

打开前端后首先进入**首页**：居中展示产品品牌与说明，并以卡片形式列出四大模块——**数据库安全保护伞**、**API 安全保护伞**、**MQ 安全保护伞**、**管理中心**。业务功能需登录后使用；未登录时可在右上角进入登录。后续步骤从「管理中心」配置账号开始，再进入「数据库安全保护伞」完成策略、数据源、扫描与概览。

**首页**：点击对应卡片的「进入模块」即可跳转；管理中心进入后需使用 yml 预置的超级管理员账密登录，数据库等业务模块则使用在管理中心创建的账号。

![首页](data-sec-umbrella-front/public/images/交互图/首页.png)

### 1. 管理中心：超级管理员登录、账号与角色

超级管理员**账号与密码**在管理端 **`application.yml`**（或等价配置）中预置；仅用于进入「管理中心」，与业务侧登录态分离。侧栏拆成两个独立路由：**`/admin-center/account`（账号管理）** 与 **`/admin-center/role`（角色管理）**，浏览器地址随菜单切换。完成登录后，可为团队创建账号并绑定角色与 **DATABASE / API / MQ** 等产品权限。

**管理员登录页**：输入预置超级管理员用户名、密码后提交，服务端校验通过后写入管理端会话（前端本地亦会保存令牌，便于刷新后仍保持登录）。

![管理员登录](data-sec-umbrella-front/public/images/交互图/管理中心/管理员登录.png)

**新建或编辑账号（弹窗）**：填写登录名、初始密码（新建时）、角色代码（**ADMIN** / **OPERATOR**）、多选产品权限及启用状态；超级管理员可维护他人账号，普通管理员仅能做只读或受限操作（以接口权限为准）。

![账号管理-配置](data-sec-umbrella-front/public/images/交互图/管理中心/账号管理-配置.png)

**账号列表**：分页展示已创建账号，可看到角色、产品权限标签、启用/禁用、创建与修改审计字段；支持从列表发起编辑、重置密码或删除（删除需二次确认）。路由为 `/admin-center/account`。

![账号管理列表](data-sec-umbrella-front/public/images/交互图/管理中心/账号管理列表.png)

**角色管理**：路由 `/admin-center/role`，以表格静态展示内置角色说明（当前为 **ADMIN**、**OPERATOR**）：角色代码、名称摘要、能力边界与产品范围。与新建账号时可选角色保持一致，便于对照授权；后续可扩展为可配置权限矩阵。

### 2. 策略管理：规则与分类策略

在「数据库安全保护伞 → 策略管理」中按引擎（如 MySQL / ClickHouse）维度维护策略；每条策略对应唯一的 **policy_code**，供离线任务、资产扫描等引用。

**策略列表**：支持按名称、创建人、敏感等级等条件筛选；列表中可快速区分数据库类型、策略启用情况，并进入编辑或删除（删除前需确认无关键任务依赖）。

![策略管理列表](data-sec-umbrella-front/public/images/交互图/数据安全保护伞/策略管理列表.png)

**策略配置（编辑弹窗）**：维护策略名称、描述、敏感等级与「隐藏样例」等基础信息；核心是**多条分类规则**（条件对象、条件类型、表达式、命中比例）以及汇总用的**规则表达式**与 **AI 规则**文本，并可填写样例数据做「规则检测」校验后再保存。

![策略管理配置](data-sec-umbrella-front/public/images/交互图/数据安全保护伞/策略管理配置.png)

### 3. 数据源与数据资产

先登记可连通的**数据源实例**（连接串、账号、类型等），管理端会探测连通性；资产扫描与离线任务均依赖此处配置，避免在业务页面硬编码连接信息。前端按引擎拆分路由（如 MySQL：`/database-security/data-source/mysql`、`/database-security/data-asset/mysql/{instance|database|table}`；ClickHouse 同理替换路径段），下列截图以 MySQL 为例，ClickHouse 交互与能力对齐。

**数据源配置（新建/编辑）**：选择数据源类型（如 MySQL、ClickHouse 等）、填写实例地址（`host:port` 形式）、库侧账号密码及说明；保存前可触发「测试连接」确认网络与凭据正确。

![数据源配置](data-sec-umbrella-front/public/images/交互图/数据安全保护伞/数据源配置.png)

**数据源列表**：按类型与关键字过滤已登记实例，查看连通性状态、创建/修改时间；从列表可再次编辑或作为后续资产发现、扫描任务的选取范围。

![数据源配置列表](data-sec-umbrella-front/public/images/交互图/数据安全保护伞/数据源配置列表.png)

**数据资产 · 实例维度**：以数据源实例为入口浏览其下挂资产规模（库、表数量或健康度等摘要，以后端返回为准），用于确认采集范围是否覆盖预期环境。

![资产管理-实例](data-sec-umbrella-front/public/images/交互图/数据安全保护伞/资产管理-实例.png)

**数据资产 · 数据库维度**：在选定实例下展开数据库列表，查看库名、描述及策略扫描产出的敏感等级、标签等聚合信息，便于先做库级治理。

![资产管理-数据库](data-sec-umbrella-front/public/images/交互图/数据安全保护伞/资产管理-数据库.png)

**数据资产 · 表维度**：进一步下钻到表列表，关注单表敏感等级、标签及人工打标状态，为列级审计或抽样扫描提供入口。

![资产管理-表](data-sec-umbrella-front/public/images/交互图/数据安全保护伞/资产管理-表.png)

**表详情 · 列级**：在表维度打开详情后，逐列展示数据类型、注释，以及**规则扫描 / AI 扫描**各自的敏感等级、标签与样例片段，支持对单列结论做人工复核。

![资产管理-表详情（列）](data-sec-umbrella-front/public/images/交互图/数据安全保护伞/资产管理-表详情（列）.png)

### 4. 批量离线扫描：任务配置

在「任务管理 → 批量任务」中按引擎（MySQL / ClickHouse）分别维护离线扫描任务：任务与实例共用表 `db_asset_scan_offline_job` / `db_asset_scan_offline_job_instance`，通过 **`database_type`** 区分引擎。配置页与实例页亦拆分 URL（如 `/database-security/task-management/batch/mysql/config` 与 `.../instances`，ClickHouse 对应 `.../batch/clickhouse/...`）。

**任务列表**：展示任务名、扫描周期、启用状态等；可对单条任务执行「立即运行」生成实例，或停用周期调度，避免误扫生产。关联策略下拉仅展示**当前引擎**下的 `policy_code`。

![批量扫描任务管理列表](data-sec-umbrella-front/public/images/交互图/数据安全保护伞/批量扫描任务管理列表.png)

**任务配置（新建/编辑）**：配置采样条数与取样方式、是否启用规则/AI 扫描、扫描范围（全实例或指定实例）、时间窗口（全量/增量）及关联 **policy_code** 列表；保存后由 Worker 按队列消费执行。

![批量扫描任务配置](data-sec-umbrella-front/public/images/交互图/数据安全保护伞/批量扫描任务配置.png)

### 5. 执行扫描与查看实例结果

每次「执行」或周期触发会生成一条**任务实例**记录运行进度、成功/失败计数；实例完成后可查看落库的**快照**（表级、字段级事件），用于审计与排障。

**任务实例列表**：按运行状态、任务名过滤实例，查看进度百分比、起止时间；可停止运行中实例或打开详情查看快照与列级 JSON。

![批量扫描任务实例列表](data-sec-umbrella-front/public/images/交互图/数据安全保护伞/批量扫描任务实例列表.png)

**实例详情 · 快照总览**：在弹窗内切换「表级 / 字段级」快照 Tab，按唯一键、敏感等级、标签等条件过滤；快照数据有保留周期（如表 TTL 约 180 天），超期由存储层清理。

![批量扫描任务实例详情（快照）](data-sec-umbrella-front/public/images/交互图/数据安全保护伞/批量扫描任务实例详情（快照）.png)

**字段级快照**：展示扫描过程中产生的字段事件（时间、唯一键、敏感标签、样例等），便于对照策略命中情况与具体列值分布。

![批量扫描任务实例详情-字段快照](data-sec-umbrella-front/public/images/交互图/数据安全保护伞/批量扫描任务实例详情-字段快照.png)

**表级快照中的列详情 JSON**：对单条表级记录可展开「列详情」，查看结构化 JSON（列名、等级、标签、样例列表等），用于与线上一致性比对或导出分析。

![批量扫描任务实例详情-表的列详情](data-sec-umbrella-front/public/images/交互图/数据安全保护伞/批量扫描任务实例详情-表的列详情.png)

### 6. 大盘概览

**概览页**：在「数据库安全保护伞 → 概览」按引擎分 Tab（URL 区分，如 `/database-security/overview/mysql` 与 `/database-security/overview/clickhouse`），聚合展示资产体量、扫描覆盖、风险分布等图表与指标，并支持后端刷新统计；具体卡片以后端接口为准，便于快速掌握当前数据安全态势与趋势。截图以 MySQL 概览为例。

![概览](data-sec-umbrella-front/public/images/交互图/数据安全保护伞/概览.png)

## 快速启动脚本

### 前端一键启动

项目提供了便捷的前端启动脚本 `start.sh`，位于 `/data-sec-umbrella-front` 目录下。

#### 使用方法：

1. 进入前端项目目录：
   ```bash
   cd /Users/huang/Documents/Workspaces/data-sec-umbrella/data-sec-umbrella-front
   ```

2. 给脚本添加执行权限（首次使用）：
   ```bash
   chmod +x start.sh
   ```

3. 执行启动脚本：
   ```bash
   ./start.sh
   ```
   （脚本内部一般为 `npm run dev` / Vite；若与仓库脚本不一致，以 `package.json` 为准。）

#### 脚本功能：
- 检查 Node.js / npm
- 按需安装依赖
- 启动前端开发服务器并提示访问地址

## 更新日志

### v0.1.x（进行中）
- 数据库安全：MySQL / ClickHouse 数据源、资产、策略、批量离线扫描（规则 + AI）、快照与概览
- 管理中心：超级管理员、账号 / 角色分路由
- 架构：Manager + Worker、RabbitMQ、Redis、Ollama
- 前端：Vite + React 19；首页模块入口与页脚备案信息

### v0.1.0
- 初始版本与项目骨架