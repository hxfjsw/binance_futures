# 奇澳智能量化系统（CiaoAI）仿制开发文档

> **用途**：本文档用于指导开发团队仿制一套与 CiaoAI 同功能、同交互的币安合约量化交易平台。
> **分析来源**：基于 `https://binance_futures.ciaolab.io/` 实际系统体验 + 网络抓包

---

## 一、技术栈选型建议

| 层级 | 推荐方案 | 说明 |
|------|----------|------|
| **前端** | React 18 + TypeScript + Vite | 主流方案，组件化开发 |
| **UI 组件库** | Ant Design 5.x（暗色主题） | 系统大量使用 AntD 的 Modal/Form/Table/Select/Switch/Tabs |
| **图表** | TradingView Lightweight Charts / KLineChart | 工作面板 K 线图 |
| **状态管理** | Zustand / Redux Toolkit | 多账户、多机器人状态管理 |
| **HTTP 请求** | Axios | 统一拦截、Token 注入 |
| **WebSocket** | 原生 WebSocket / Socket.IO | 实时盘口深度、K线推送 |
| **后端** | Node.js + Express / NestJS | 或 Go + Gin（高性能） |
| **数据库** | PostgreSQL + Redis | PG 存配置/日志，Redis 缓存行情/机器人状态 |
| **任务队列** | BullMQ (Redis) / RabbitMQ | 机器人策略执行队列 |
| **币安API** | node-binance-api / ccxt | 封装币安合约 API |

---

## 二、前端页面结构 & 路由设计

```
/                          → 登录页（Login）
/dashboard                 → 控制面板（Dashboard）
/robot-deployment          → 机器人部署（Robot Lab）
/work-panel                → 工作面板（Trading Desk）
/risk-management           → 风控（Risk Control）
/user/settings             → 用户-后台设置（User Admin）
/user/logs                 → 用户-日志管理（Login Logs）
```

### 2.1 布局组件（Layout）

所有内部页面共享一个 `MainLayout`：

```
┌────────────────────────────────────────────────────────┐
│ TopBar: 账户信息 | 参数设置 | 导入API Key | 语言 | 交易对 │
├────────┬───────────────────────────────────────────────┤
│        │                                               │
│ 侧边栏  │              主内容区（RouterView）            │
│  Logo   │                                               │
│  导航菜单│                                               │
│  导入   │                                               │
│  API Key│                                               │
│        │                                               │
└────────┴───────────────────────────────────────────────┘
```

**侧边栏导航菜单**：
- 控制面板（图标：仪表盘）
- 机器人部署（图标：机器人）
- 工作面板（图标：交易）
- 风控（图标：盾牌）
- 用户（图标：人物，展开子菜单：后台设置 / 日志管理）

---

## 三、后端 API 接口设计（已抓包验证 + 推断）

### 3.1 已确认的接口（来自网络抓包）

| 接口 | 方法 | 说明 |
|------|------|------|
| `/api/openapi/account?apiKey={apiKey}` | GET | 获取单个账户资产信息 |
| `/api/openapi/allAccounts` | GET | 获取当前用户绑定的所有账户列表 |
| `/api/openapi/depth?symbol={symbol}&limit={limit}&hosts={host}` | GET | 获取订单簿深度（代理到币安） |
| `/api/openapi/getAllPositions` | GET | 获取所有账户持仓 |
| `/api/openapi/getIsRunningRobots?name={account}` | GET | 获取运行中的机器人列表 |
| `/api/openapi/getMatchedOrdersHandle` | GET | 获取对敲/匹配订单记录 |
| `/api/openapi/getTradeSettings?apiKey={apiKey}&symbol={symbol}` | GET | 获取交易参数设置（杠杆、持仓模式等） |
| `/api/openapi/historyOrders?symbol={symbol}&apiKey={apiKey}` | GET | 历史委托 |
| `/api/openapi/klines?symbol={symbol}&interval={interval}&startTime={t}&endTime={t}&limit={n}&hosts={host}` | GET | K线数据（代理到币安） |
| `/api/openapi/openOrders?symbol={symbol}&apiKey={apiKey}` | GET | 当前委托 |
| `/api/openapi/trades?symbol={symbol}&limit={n}&hosts={host}` | GET | 成交记录（代理到币安） |

### 3.2 推断的接口（从功能推断）

| 接口 | 方法 | 说明 |
|------|------|------|
| `/api/auth/login` | POST | 登录：{account, password, agree} |
| `/api/auth/refresh` | POST | Token 刷新 |
| `/api/robot/config` | POST | 保存机器人配置 |
| `/api/robot/start` | POST | 启动机器人：{robotId, accountId, symbol} |
| `/api/robot/stop` | POST | 停止机器人：{robotId, accountId} |
| `/api/robot/list` | GET | 获取所有机器人配置列表 |
| `/api/trade/place` | POST | 下单：{symbol, side, type, quantity, price, apiKey} |
| `/api/trade/cancel` | POST | 撤单：{symbol, orderId, apiKey} |
| `/api/trade/batch` | POST | 批量下单/撤单 |
| `/api/trade/match` | POST | 手动对敲：{symbol, quantity, splits, mode} |
| `/api/account/settings` | POST | 保存账户参数设置 |
| `/api/account/importKey` | POST | 导入API Key：{label, apiKey, secret, domain} |
| `/api/risk/alert` | POST | 保存风控告警配置 |
| `/api/user/logs` | GET | 登录日志 |
| `/api/user/list` | GET | 用户列表（管理员） |
| `/api/notify/sendCode` | POST | 发送验证码（查看API Key时） |
| `/api/notify/verify` | POST | 验证验证码 |

---

## 四、数据模型设计

### 4.1 用户表（users）

```sql
CREATE TABLE users (
    id SERIAL PRIMARY KEY,
    account VARCHAR(100) UNIQUE NOT NULL,      -- ciaoaifuture@ciaocex.com
    password_hash VARCHAR(255) NOT NULL,
    name VARCHAR(50),                            -- 员工名称
    role VARCHAR(20) DEFAULT 'user',             -- admin / user
    bind_ip VARCHAR(50),                         -- 129.226.213.129
    created_at TIMESTAMP DEFAULT NOW(),
    last_login_at TIMESTAMP,
    last_login_ip VARCHAR(50)
);
```

### 4.2 API Key 账户表（api_keys）

```sql
CREATE TABLE api_keys (
    id SERIAL PRIMARY KEY,
    user_id INT REFERENCES users(id),
    label VARCHAR(50) NOT NULL,                  -- 账户1, 账户2...
    api_key VARCHAR(255) NOT NULL,               -- 币安API Key
    api_secret VARCHAR(255) NOT NULL,            -- 币安API Secret
    domain VARCHAR(255) DEFAULT 'https://fapi.binance.com',
    margin_type VARCHAR(20) DEFAULT 'cross',     -- isolated / cross (逐仓/全仓)
    position_mode VARCHAR(20) DEFAULT 'single',  -- single / dual (单向/双向)
    leverage INT DEFAULT 5,
    is_active BOOLEAN DEFAULT true,
    created_at TIMESTAMP DEFAULT NOW()
);
```

### 4.3 机器人配置表（robot_configs）

```sql
CREATE TABLE robot_configs (
    id SERIAL PRIMARY KEY,
    user_id INT REFERENCES users(id),
    api_key_id INT REFERENCES api_keys(id),
    robot_type VARCHAR(50) NOT NULL,             -- min_volume, random_volume, natural_volume, zone_volume, mid_price_volume, order_flash, near_depth, spread_depth, oscillation_up, oscillation_down, rapid_up, combo_follow, zone_oscillation, enhanced_zone, high_low_correct, ratio_oscillation, amm_market_making, bulldozer, smart_exit
    category VARCHAR(20) NOT NULL,               -- normal / strategy / ai / risk
    symbol VARCHAR(20) NOT NULL,                 -- TRXUSDT
    params JSONB NOT NULL DEFAULT '{}',          -- 各策略参数（见下文）
    status VARCHAR(20) DEFAULT 'stopped',        -- running / stopped / error
    start_mode VARCHAR(20) DEFAULT 'immediate',  -- immediate / scheduled
    price_mode VARCHAR(20) DEFAULT 'normal',     -- normal / safe
    description TEXT,
    created_at TIMESTAMP DEFAULT NOW(),
    updated_at TIMESTAMP DEFAULT NOW(),
    started_at TIMESTAMP,
    stopped_at TIMESTAMP
);
```

### 4.4 机器人参数 JSONB 结构示例

#### 随机刷量（random_volume）
```json
{
  "trade_amount_min": 1000,
  "trade_amount_max": 2000,
  "interval_min": 1,
  "interval_max": 2,
  "order_mode": "random",        // random / buy_first / sell_first
  "price_mode": "safe",          // normal / safe
  "start_mode": "immediate",     // immediate / scheduled
  "distribution": "continuous"    // continuous / random
}
```

#### 高抛低吸_修正（high_low_correct）
```json
{
  "max_sell_amount": 1000,
  "sell_trigger_price": 0.3,
  "max_buy_amount_usdt": 60,
  "buy_trigger_price": 0.2975,
  "buy_order_amount_min": 50,
  "buy_order_amount_max": 700,
  "buy_interval_min": 0.1,
  "buy_interval_max": 0.5,
  "sell_order_amount_min": 50,
  "sell_order_amount_max": 700,
  "sell_interval_min": 0.1,
  "sell_interval_max": 0.5,
  "start_mode": "immediate"
}
```

#### AMM 智能做市（amm_market_making）
```json
{
  "invest_usdt": 50,
  "invest_token": 10,
  "interval_seconds": 500,
  "price_upper": 0.5,
  "price_lower": 0.01,
  "grid_count": 5,               // 5, 10, 20, 30, 40
  "start_mode": "immediate"
}
```

#### 余额监控（balance_monitor）
```json
{
  "email_1": "",
  "email_2": "",
  "email_3": "",
  "balance_alert_interval_min": 20,
  "balance_change_alert_interval_min": 20,
  "balance_threshold": 100,
  "token_threshold": 10000,
  "start_mode": "immediate"
}
```

### 4.5 订单表（orders）

```sql
CREATE TABLE orders (
    id SERIAL PRIMARY KEY,
    user_id INT REFERENCES users(id),
    api_key_id INT REFERENCES api_keys(id),
    symbol VARCHAR(20) NOT NULL,
    order_id VARCHAR(100),          -- 币安订单ID
    client_order_id VARCHAR(100),    -- 自定义订单ID
    side VARCHAR(10) NOT NULL,       -- BUY / SELL
    type VARCHAR(20) NOT NULL,       -- LIMIT / MARKET / STOP / TAKE_PROFIT
    price DECIMAL(36,18),
    quantity DECIMAL(36,18),
    executed_qty DECIMAL(36,18) DEFAULT 0,
    status VARCHAR(20) DEFAULT 'NEW', -- NEW / PARTIALLY_FILLED / FILLED / CANCELLED / REJECTED
    source VARCHAR(50) DEFAULT 'manual', -- manual / robot_{robot_id}
    created_at TIMESTAMP DEFAULT NOW(),
    updated_at TIMESTAMP DEFAULT NOW()
);
```

### 4.6 登录日志表（login_logs）

```sql
CREATE TABLE login_logs (
    id SERIAL PRIMARY KEY,
    user_id INT REFERENCES users(id),
    account VARCHAR(100) NOT NULL,
    login_time TIMESTAMP DEFAULT NOW(),
    login_ip VARCHAR(50) NOT NULL,
    user_agent TEXT,
    status VARCHAR(20) DEFAULT 'success' -- success / failed
);
```

---

## 五、前端页面详细实现要点

### 5.1 登录页（/）

**布局**：左表单 + 右插画（全屏居中）

```
左面板：
- 标题："奇澳智能量化系统"
- 副标题："输入您的账户密码登录"
- 表单：
  - 账户名输入框（placeholder: 请输入账户名）
  - 密码输入框（placeholder: 请输入密码，带眼睛切换）
  - 复选框：我同意用户协议
  - 链接：点击查看用户协议
  - 登录按钮（disabled 到勾选协议）
- 底部：ENGLISH 切换 | 忘记密码？

右面板：金融插画（人物 + 电脑 + 金币 + 图表）

底部：免责声明文本（黄色小字）
```

**技术实现**：
- 使用 AntD Form + Input.Password + Checkbox + Button
- 登录成功后存储 JWT Token（localStorage）
- 重定向到 /dashboard

### 5.2 控制面板（/dashboard）

**顶部：账户矩阵表格**

```
列：标签 | apiKey（脱敏） | 域名 | 账户总净值 | 可用余额 | 未实现盈亏
行：账户1 ~ 账户N（支持横向滚动，当前可见7个账户）
```

**中部：余额卡片**
- 图标 + 余额标题
- 可用 USDT: {value} | 冻结 USDT: {value}

**下部：机器人状态卡片（按分类分组）**

```
常规机器人：3列卡片
AI 机器人：3列卡片
风控机器人：3列卡片（控制面板页只显示部分，完整在风控页）
```

**卡片结构**：
```
┌──────────────────────────┐
│ 机器人名称          [配置参数]│
│ 描述为空                  │
│ ───────────────────────── │
│ 未运行              [开关] │
└──────────────────────────┘
```

**状态颜色**：
- 未运行：灰色开关
- 运行中：蓝色高亮卡片 + 蓝色开关 + 日期显示
- 查看按钮：运行中机器人显示"查看"而非"配置参数"

### 5.3 机器人部署页（/robot-deployment）

**完整策略列表（19种）**：

```
常规机器人（8个）：
├─ 最低维持刷量
├─ 随机刷量
├─ 自然刷量
├─ 区间刷量
├─ 中间价刷量
├─ 盘口闪烁
├─ 近端深度
└─ 分布深度

策略机器人（8个）：
├─ 振荡拉升
├─ 振荡下跌
├─ 极速拉升
├─ 组合跟随模式
├─ 区间振荡
├─ 增强区间振荡
├─ 高抛低吸_修正
└─ 比例震荡

AI 机器人（3个）：
├─ AMM 智能做市
├─ 推土机模式
└─ 智能出货
```

**配置参数弹窗（Modal）结构**：

根据机器人类型，弹窗内表单字段不同。通用字段：
- 价格模式：ToggleButton（一般模式 / 安全模式）
- 启动模式：ToggleButton（立即启动 / 定时启动）
- 机器人描述：TextArea
- 保存按钮

特殊字段按类型定义（见第4.4节 JSON 结构）。

**按钮组设计**：
- 二选一：使用 AntD Segmented / 自定义 ToggleButton 组件
- 三选一：同上
- 多选一（如档数 5/10/20/30/40）：横向按钮组
- 范围输入：两个 InputNumber + "~" 分隔符

### 5.4 工作面板（/work-panel）

**上部：K线图区域**
- 交易对标题 + O/H/L/C/V 数据
- 5分钟K线蜡烛图（绿色涨，红色跌）
- 时间轴：10:00 ~ 14:30
- 数据来源：/api/openapi/klines

**右侧：实时盘口深度（固定宽度约200px）**
- 卖盘（红色）：价格从高到低
- 买盘（绿色）：价格从低到高
- 显示：价格、数量、累计深度
- 数据源：/api/openapi/depth

**下部：交易操作区（Tabs）**

| Tab | 内容 |
|-----|------|
| **手动对敲** | 交易量输入 + 分笔数输入 + 随机值/中间值按钮 + 对敲按钮 + 结果表格（时间/价格/数量/分笔数） |
| **批量挂单** | 批量创建限价/市价单 |
| **集群挂单** | 多账户同时挂单 |
| **持仓** | 当前持仓列表 |
| **当前委托** | 未成交订单列表 |
| **历史委托** | 已成交/已取消订单列表 |

**右上角账户选择器**：下拉选择当前操作的账户（账户1 ~ 账户N）

### 5.5 风控页（/risk-management）

**风控机器人（5个）**：
```
├─ 余额监控
├─ 价格监控
├─ 余额定时报告
├─ TG 日常提醒
└─ TG 报警
```

**余额监控配置**：
- Email 1/2/3 输入框
- 金额定时提醒间隔（min）
- 金额变动提醒间隔（min）
- Balance 警戒值
- Token 警戒值

**通知渠道**：
- Email：SMTP 发送
- Telegram：Bot API 发送消息

### 5.6 用户页

**后台设置（/user/settings）**：
- 用户列表表格：账户 | 身份 | 员工名称
- 支持多选（复选框）

**日志管理（/user/logs）**：
- 登录日志表格：账户 | 登录时间 | 登录IP

---

## 六、核心机器人策略算法思路

### 6.1 随机刷量（Random Volume）

```python
while robot_running:
    amount = random.uniform(amount_min, amount_max)
    interval = random.uniform(interval_min, interval_max)
    
    if order_mode == "random":
        side = random.choice(["BUY", "SELL"])
    elif order_mode == "buy_first":
        side = "BUY"
    else:
        side = "SELL"
    
    place_market_order(symbol, side, amount)
    time.sleep(interval)
```

### 6.2 高抛低吸_修正（High-Low Correct）

```python
while robot_running:
    current_price = get_market_price(symbol)
    
    # 卖出逻辑
    if current_price >= sell_trigger_price and token_balance > 0:
        amount = random.uniform(sell_amount_min, sell_amount_max)
        sell_amount = min(amount, token_balance, max_sell_amount)
        if sell_amount > 0:
            place_limit_order(symbol, "SELL", sell_amount, current_price)
            time.sleep(random.uniform(sell_interval_min, sell_interval_max))
    
    # 买入逻辑
    if current_price <= buy_trigger_price and usdt_balance > 0:
        amount = random.uniform(buy_amount_min, buy_amount_max)
        cost = amount * current_price
        if cost <= max_buy_usdt and cost <= usdt_balance:
            place_limit_order(symbol, "BUY", amount, current_price)
            time.sleep(random.uniform(buy_interval_min, buy_interval_max))
    
    time.sleep(0.5)
```

### 6.3 AMM 智能做市（AMM Market Making）

```python
# 等差网格做市
center_price = (price_upper + price_lower) / 2
grid_prices = [price_lower + i * (price_upper - price_lower) / (grid_count - 1) 
               for i in range(grid_count)]

# 每个档位分配等额资金
usdt_per_grid = invest_usdt / grid_count
token_per_grid = invest_token / grid_count

for price in grid_prices:
    if price < center_price:
        # 在低价区挂买单
        amount = usdt_per_grid / price
        place_limit_order(symbol, "BUY", amount, price)
    elif price > center_price:
        # 在高价区挂卖单
        amount = token_per_grid
        place_limit_order(symbol, "SELL", amount, price)

# 定时轮询：成交后重新挂单
while robot_running:
    check_and_rebalance_orders()
    time.sleep(interval_seconds)
```

### 6.4 余额监控（Balance Monitor）

```python
while robot_running:
    # 定时检查
    if time_since_last_balance_check >= balance_alert_interval * 60:
        balance = get_account_balance()
        if balance["usdt"] < balance_threshold:
            send_alert(f"Balance alert: USDT {balance['usdt']} < {balance_threshold}")
        if balance["token"] < token_threshold:
            send_alert(f"Token alert: Token {balance['token']} < {token_threshold}")
    
    # 变动检查
    if abs(balance_change) > 0 and time_since_last_change >= change_alert_interval * 60:
        send_alert(f"Balance changed: {balance_change}")
    
    time.sleep(10)
```

### 6.5 手动对敲（Match Trading）

```python
def manual_match(symbol, total_amount, split_count, mode):
    """
    mode: "random" = 随机值对敲, "mid" = 中间值对敲
    """
    amounts = split_total(total_amount, split_count)
    
    records = []
    for i, amount in enumerate(amounts):
        price = get_market_price(symbol)
        
        if mode == "random":
            price = price * random.uniform(0.999, 1.001)
        
        # 同时下买单和卖单
        buy_order = place_limit_order(symbol, "BUY", amount, price)
        sell_order = place_limit_order(symbol, "SELL", amount, price)
        
        records.append({
            "time": now(),
            "price": price,
            "quantity": amount,
            "splits": i + 1
        })
    
    return records
```

---

## 七、币安 API 代理层设计

系统所有交易/行情请求都通过后端代理到币安，避免前端暴露 API Secret。

```typescript
// 币安合约 API 封装（Node.js）
class BinanceFuturesProxy {
    constructor(apiKey: string, apiSecret: string, baseURL: string = 'https://fapi.binance.com') {
        this.apiKey = apiKey;
        this.apiSecret = apiSecret;
        this.baseURL = baseURL;
    }

    async placeOrder(params: OrderParams) {
        const signature = this.sign(params);
        return axios.post(`${this.baseURL}/fapi/v1/order`, params, {
            headers: { 'X-MBX-APIKEY': this.apiKey }
        });
    }

    async getAccountInfo() {
        const params = { timestamp: Date.now() };
        params.signature = this.sign(params);
        return axios.get(`${this.baseURL}/fapi/v2/account`, { params, headers: { 'X-MBX-APIKEY': this.apiKey } });
    }

    async getDepth(symbol: string, limit: number = 40) {
        return axios.get(`${this.baseURL}/fapi/v1/depth`, { params: { symbol, limit } });
    }

    async getKlines(symbol: string, interval: string, limit: number = 500) {
        return axios.get(`${this.baseURL}/fapi/v1/klines`, { params: { symbol, interval, limit } });
    }

    async getTrades(symbol: string, limit: number = 1000) {
        return axios.get(`${this.baseURL}/fapi/v1/trades`, { params: { symbol, limit } });
    }

    async getOpenOrders(symbol: string) {
        const params = { symbol, timestamp: Date.now() };
        params.signature = this.sign(params);
        return axios.get(`${this.baseURL}/fapi/v1/openOrders`, { params, headers: { 'X-MBX-APIKEY': this.apiKey } });
    }

    async getAllOrders(symbol: string) {
        const params = { symbol, timestamp: Date.now() };
        params.signature = this.sign(params);
        return axios.get(`${this.baseURL}/fapi/v1/allOrders`, { params, headers: { 'X-MBX-APIKEY': this.apiKey } });
    }

    private sign(params: any): string {
        const queryString = new URLSearchParams(params).toString();
        return crypto.createHmac('sha256', this.apiSecret).update(queryString).digest('hex');
    }
}
```

---

## 八、关键组件清单

| 组件 | 说明 | 实现方式 |
|------|------|----------|
| `RobotCard` | 机器人状态卡片 | AntD Card + Switch + Button |
| `RobotConfigModal` | 机器人配置弹窗 | AntD Modal + Form + 动态字段 |
| `ToggleButtonGroup` | 二选一/三选一按钮组 | 自定义组件（仿 AntD Segmented） |
| `RangeInput` | 范围输入（min~max） | 两个 InputNumber + 波浪号 |
| `AccountTable` | 多账户表格 | AntD Table + 横向滚动 |
| `DepthPanel` | 实时盘口深度 | 右侧固定面板 + 自定义列表 |
| `KLineChart` | K线图 | TradingView Lightweight Charts |
| `TradeTabs` | 交易操作区 | AntD Tabs + 自定义内容 |
| `TopBar` | 顶部工具栏 | Flex布局 + Button + Select + Dropdown |
| `Sidebar` | 侧边导航 | 固定宽度 + Menu + 可展开子菜单 |
| `GlobalSettingsModal` | 全局参数设置 | AntD Modal + Form + Select + Switch |
| `ImportKeyModal` | 导入API Key | AntD Modal + Input + OTP验证 |

---

## 九、开发里程碑（MVP → 完整版）

### Phase 1: 基础框架（1-2周）
- [ ] 登录/注册/权限系统
- [ ] 主布局（侧边栏 + 顶部栏 + 内容区）
- [ ] 币安 API 代理层（基础行情接口）
- [ ] API Key 管理（增删查）

### Phase 2: 核心功能（2-3周）
- [ ] 控制面板（多账户列表 + 余额卡片）
- [ ] 机器人部署页（19种策略卡片 + 配置弹窗）
- [ ] 机器人引擎（至少实现：随机刷量、AMM做市、余额监控）
- [ ] 工作面板（手动对敲 + 盘口深度 + K线图）

### Phase 3: 完整功能（2-3周）
- [ ] 所有19种策略机器人算法实现
- [ ] 风控模块（价格监控 + TG通知）
- [ ] 批量挂单/集群挂单
- [ ] 持仓/委托管理
- [ ] 用户管理 + 登录日志

### Phase 4: 优化上线（1-2周）
- [ ] 策略回测/模拟盘
- [ ] 风控硬限制（止损、互锁）
- [ ] 多语言（中英切换）
- [ ] 部署 + 监控 + 告警

---

## 十、安全注意事项

1. **API Secret 绝不暴露给前端**：所有币安请求通过后端代理，签名在服务端完成
2. **JWT Token 安全**：HttpOnly Cookie 或 localStorage（推荐前者）
3. **验证码保护**：查看 API Key 时需二次验证（8位验证码）
4. **IP 绑定**：登录时记录 IP，提示用户绑定（可选强制绑定）
5. **敏感操作日志**：机器人启停、参数修改、订单操作全部记录审计日志
6. **Rate Limit**：币安 API 有严格限频，后端需做请求队列和限流控制

---

*文档基于实际系统逆向分析，API 路径和参数结构可直接参考实现。*
