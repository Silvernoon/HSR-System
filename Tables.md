# 高铁票务在线预订与分配系统数据库表头（MySQL 8）

## 1. 设计说明

- 数据库：`MySQL 8`
- 主键：统一使用 `BIGINT`
- 金额：统一使用 `DECIMAL(10,2)`
- 日期：使用 `DATE`
- 时间戳：使用 `DATETIME`
- 管理员与普通用户共用 `account` 表，通过 `role` 区分
- 一张订单可包含多张车票
- 车票按区间售卖，库存按相邻经停区间维护
- 交易数据默认不做物理删除，以状态流转为主

---

## 2. 数据表一览

| 表名 | 中文说明 |
| --- | --- |
| `account` | 账号表 |
| `passenger` | 乘车人表 |
| `station` | 车站表 |
| `train` | 列车表 |
| `train_stop` | 列车经停站表 |
| `seat_type` | 座席类型表 |
| `carriage` | 车厢表 |
| `seat` | 座位表 |
| `train_run` | 列车开行实例表 |
| `run_fare` | 开行票价表 |
| `run_leg_inventory` | 区间库存表 |
| `ticket_order` | 订单表 |
| `ticket` | 车票表 |
| `payment_record` | 支付记录表 |
| `refund_record` | 退票记录表 |
| `change_record` | 改签记录表 |
| `waitlist_order` | 候补订单表 |
| `notification_message` | 通知消息表 |

---

## 3. 数据表表头

### 3.1 `account` 账号表

| 字段名 | 类型 | 约束 | 说明 |
| --- | --- | --- | --- |
| `id` | `BIGINT` | `PK`, `AUTO_INCREMENT` | 账号主键 |
| `username` | `VARCHAR(50)` | `NOT NULL`, `UNIQUE` | 登录用户名 |
| `password_hash` | `VARCHAR(255)` | `NOT NULL` | 密码哈希值 |
| `phone` | `VARCHAR(20)` | `NOT NULL`, `UNIQUE` | 手机号 |
| `real_name` | `VARCHAR(50)` | `NULL` | 账号真实姓名 |
| `role` | `ENUM('USER','ADMIN')` | `NOT NULL`, `DEFAULT 'USER'` | 账号角色 |
| `status` | `VARCHAR(20)` | `NOT NULL`, `DEFAULT 'ACTIVE'` | 账号状态 |
| `created_at` | `DATETIME` | `NOT NULL` | 创建时间 |
| `updated_at` | `DATETIME` | `NOT NULL` | 更新时间 |

### 3.2 `passenger` 乘车人表

| 字段名 | 类型 | 约束 | 说明 |
| --- | --- | --- | --- |
| `id` | `BIGINT` | `PK`, `AUTO_INCREMENT` | 乘车人主键 |
| `account_id` | `BIGINT` | `NOT NULL`, `FK -> account.id` | 所属账号 |
| `name` | `VARCHAR(50)` | `NOT NULL` | 乘车人姓名 |
| `id_type` | `VARCHAR(20)` | `NOT NULL` | 证件类型 |
| `id_no` | `VARCHAR(30)` | `NOT NULL` | 证件号码 |
| `passenger_type` | `VARCHAR(20)` | `NOT NULL` | 乘客类型，如成人/儿童 |
| `phone` | `VARCHAR(20)` | `NULL` | 联系电话 |
| `status` | `VARCHAR(20)` | `NOT NULL`, `DEFAULT 'ACTIVE'` | 乘车人状态 |
| `created_at` | `DATETIME` | `NOT NULL` | 创建时间 |

补充约束：

- `UNIQUE(id_type, id_no)`

### 3.3 `station` 车站表

| 字段名 | 类型 | 约束 | 说明 |
| --- | --- | --- | --- |
| `id` | `BIGINT` | `PK`, `AUTO_INCREMENT` | 车站主键 |
| `station_code` | `VARCHAR(20)` | `NOT NULL`, `UNIQUE` | 车站编码 |
| `station_name` | `VARCHAR(50)` | `NOT NULL` | 车站名称 |
| `city_name` | `VARCHAR(50)` | `NOT NULL` | 所属城市 |
| `pinyin` | `VARCHAR(100)` | `NULL` | 拼音检索字段 |
| `status` | `VARCHAR(20)` | `NOT NULL`, `DEFAULT 'ACTIVE'` | 车站状态 |

### 3.4 `train` 列车表

| 字段名 | 类型 | 约束 | 说明 |
| --- | --- | --- | --- |
| `id` | `BIGINT` | `PK`, `AUTO_INCREMENT` | 列车主键 |
| `train_no` | `VARCHAR(20)` | `NOT NULL`, `UNIQUE` | 车次号，如 `G101` |
| `train_type` | `VARCHAR(20)` | `NOT NULL` | 列车类型 |
| `origin_station_id` | `BIGINT` | `NOT NULL`, `FK -> station.id` | 始发站 |
| `destination_station_id` | `BIGINT` | `NOT NULL`, `FK -> station.id` | 终到站 |
| `sale_status` | `VARCHAR(20)` | `NOT NULL` | 售卖状态 |

### 3.5 `train_stop` 列车经停站表

| 字段名 | 类型 | 约束 | 说明 |
| --- | --- | --- | --- |
| `id` | `BIGINT` | `PK`, `AUTO_INCREMENT` | 经停记录主键 |
| `train_id` | `BIGINT` | `NOT NULL`, `FK -> train.id` | 所属列车 |
| `station_id` | `BIGINT` | `NOT NULL`, `FK -> station.id` | 经停车站 |
| `stop_index` | `INT` | `NOT NULL` | 站序，从小到大排列 |
| `arrive_time` | `TIME` | `NULL` | 到站时间，始发站可为空 |
| `depart_time` | `TIME` | `NULL` | 发车时间，终点站可为空 |
| `day_offset` | `INT` | `NOT NULL`, `DEFAULT 0` | 相对开车日偏移天数 |
| `stay_minutes` | `INT` | `NULL` | 停站分钟数 |

补充索引：

- `INDEX(train_id, stop_index)`
- `INDEX(station_id)`

### 3.6 `seat_type` 座席类型表

| 字段名 | 类型 | 约束 | 说明 |
| --- | --- | --- | --- |
| `id` | `BIGINT` | `PK`, `AUTO_INCREMENT` | 座席类型主键 |
| `seat_type_code` | `VARCHAR(20)` | `NOT NULL`, `UNIQUE` | 座席编码 |
| `seat_type_name` | `VARCHAR(50)` | `NOT NULL` | 座席名称 |
| `class_level` | `INT` | `NOT NULL` | 舱等层级 |

### 3.7 `carriage` 车厢表

| 字段名 | 类型 | 约束 | 说明 |
| --- | --- | --- | --- |
| `id` | `BIGINT` | `PK`, `AUTO_INCREMENT` | 车厢主键 |
| `train_id` | `BIGINT` | `NOT NULL`, `FK -> train.id` | 所属列车 |
| `carriage_no` | `VARCHAR(10)` | `NOT NULL` | 车厢号 |
| `seat_type_id` | `BIGINT` | `NOT NULL`, `FK -> seat_type.id` | 对应座席类型 |
| `seat_count` | `INT` | `NOT NULL` | 座位数量 |

补充约束：

- `UNIQUE(train_id, carriage_no)`

### 3.8 `seat` 座位表

| 字段名 | 类型 | 约束 | 说明 |
| --- | --- | --- | --- |
| `id` | `BIGINT` | `PK`, `AUTO_INCREMENT` | 座位主键 |
| `carriage_id` | `BIGINT` | `NOT NULL`, `FK -> carriage.id` | 所属车厢 |
| `seat_no` | `VARCHAR(10)` | `NOT NULL` | 座位号，如 `05A` |
| `row_no` | `INT` | `NULL` | 行号 |
| `col_no` | `VARCHAR(5)` | `NULL` | 列号，如 `A/B/C` |
| `position_code` | `VARCHAR(20)` | `NULL` | 座位位置偏好，如靠窗/靠过道 |
| `seat_type_id` | `BIGINT` | `NOT NULL`, `FK -> seat_type.id` | 座席类型 |

补充约束：

- `UNIQUE(carriage_id, seat_no)`

### 3.9 `train_run` 列车开行实例表

| 字段名 | 类型 | 约束 | 说明 |
| --- | --- | --- | --- |
| `id` | `BIGINT` | `PK`, `AUTO_INCREMENT` | 开行实例主键 |
| `train_id` | `BIGINT` | `NOT NULL`, `FK -> train.id` | 对应列车 |
| `run_date` | `DATE` | `NOT NULL` | 开行日期 |
| `run_status` | `ENUM('NORMAL','DELAYED','CANCELLED','CLOSED')` | `NOT NULL`, `DEFAULT 'NORMAL'` | 开行状态 |
| `sale_start_at` | `DATETIME` | `NOT NULL` | 起售时间 |
| `sale_end_at` | `DATETIME` | `NOT NULL` | 停售时间 |
| `created_at` | `DATETIME` | `NOT NULL` | 创建时间 |

补充约束：

- `UNIQUE(train_id, run_date)`

### 3.10 `run_fare` 开行票价表

| 字段名 | 类型 | 约束 | 说明 |
| --- | --- | --- | --- |
| `id` | `BIGINT` | `PK`, `AUTO_INCREMENT` | 票价记录主键 |
| `train_run_id` | `BIGINT` | `NOT NULL`, `FK -> train_run.id` | 对应开行实例 |
| `from_stop_id` | `BIGINT` | `NOT NULL`, `FK -> train_stop.id` | 上车站 |
| `to_stop_id` | `BIGINT` | `NOT NULL`, `FK -> train_stop.id` | 下车站 |
| `seat_type_id` | `BIGINT` | `NOT NULL`, `FK -> seat_type.id` | 座席类型 |
| `price` | `DECIMAL(10,2)` | `NOT NULL` | 票价 |

补充约束：

- `UNIQUE(train_run_id, from_stop_id, to_stop_id, seat_type_id)`

### 3.11 `run_leg_inventory` 区间库存表

| 字段名 | 类型 | 约束 | 说明 |
| --- | --- | --- | --- |
| `id` | `BIGINT` | `PK`, `AUTO_INCREMENT` | 库存记录主键 |
| `train_run_id` | `BIGINT` | `NOT NULL`, `FK -> train_run.id` | 对应开行实例 |
| `start_stop_id` | `BIGINT` | `NOT NULL`, `FK -> train_stop.id` | 区间起点站 |
| `end_stop_id` | `BIGINT` | `NOT NULL`, `FK -> train_stop.id` | 区间终点站 |
| `seat_type_id` | `BIGINT` | `NOT NULL`, `FK -> seat_type.id` | 座席类型 |
| `total_count` | `INT` | `NOT NULL` | 总库存 |
| `available_count` | `INT` | `NOT NULL` | 可用库存 |
| `updated_at` | `DATETIME` | `NOT NULL` | 更新时间 |

补充约束与索引：

- 建议 `UNIQUE(train_run_id, start_stop_id, end_stop_id, seat_type_id)`
- `INDEX(train_run_id, start_stop_id, seat_type_id, available_count)`

### 3.12 `ticket_order` 订单表

| 字段名 | 类型 | 约束 | 说明 |
| --- | --- | --- | --- |
| `id` | `BIGINT` | `PK`, `AUTO_INCREMENT` | 订单主键 |
| `order_no` | `VARCHAR(40)` | `NOT NULL`, `UNIQUE` | 订单号 |
| `account_id` | `BIGINT` | `NOT NULL`, `FK -> account.id` | 下单账号 |
| `order_status` | `ENUM('CREATED','PENDING_PAYMENT','PAID','PARTIAL_REFUNDED','REFUNDED','CANCELLED','COMPLETED')` | `NOT NULL`, `DEFAULT 'CREATED'` | 订单状态 |
| `total_amount` | `DECIMAL(10,2)` | `NOT NULL` | 订单总金额 |
| `paid_amount` | `DECIMAL(10,2)` | `NOT NULL`, `DEFAULT 0.00` | 实付金额 |
| `contact_phone` | `VARCHAR(20)` | `NOT NULL` | 联系电话 |
| `pay_deadline` | `DATETIME` | `NULL` | 支付截止时间 |
| `created_at` | `DATETIME` | `NOT NULL` | 创建时间 |
| `updated_at` | `DATETIME` | `NOT NULL` | 更新时间 |

### 3.13 `ticket` 车票表

| 字段名 | 类型 | 约束 | 说明 |
| --- | --- | --- | --- |
| `id` | `BIGINT` | `PK`, `AUTO_INCREMENT` | 车票主键 |
| `ticket_no` | `VARCHAR(40)` | `NOT NULL`, `UNIQUE` | 车票号 |
| `order_id` | `BIGINT` | `NOT NULL`, `FK -> ticket_order.id` | 所属订单 |
| `passenger_id` | `BIGINT` | `NOT NULL`, `FK -> passenger.id` | 对应乘车人 |
| `train_run_id` | `BIGINT` | `NOT NULL`, `FK -> train_run.id` | 对应开行实例 |
| `from_stop_id` | `BIGINT` | `NOT NULL`, `FK -> train_stop.id` | 乘车起点站 |
| `to_stop_id` | `BIGINT` | `NOT NULL`, `FK -> train_stop.id` | 乘车终点站 |
| `seat_type_id` | `BIGINT` | `NOT NULL`, `FK -> seat_type.id` | 座席类型 |
| `seat_id` | `BIGINT` | `NULL`, `FK -> seat.id` | 分配的座位，可为空 |
| `carriage_no_snapshot` | `VARCHAR(10)` | `NULL` | 下单时快照车厢号 |
| `seat_no_snapshot` | `VARCHAR(10)` | `NULL` | 下单时快照座位号 |
| `preferred_position` | `VARCHAR(20)` | `NULL` | 座位偏好 |
| `allocation_group_no` | `VARCHAR(40)` | `NULL` | 同行分配组号 |
| `ticket_status` | `ENUM('LOCKED','ISSUED','WAITLISTED','REFUNDING','REFUNDED','CHANGED','CANCELLED')` | `NOT NULL`, `DEFAULT 'LOCKED'` | 车票状态 |
| `price` | `DECIMAL(10,2)` | `NOT NULL` | 票价 |
| `lock_expires_at` | `DATETIME` | `NULL` | 锁座失效时间 |
| `created_at` | `DATETIME` | `NOT NULL` | 创建时间 |

补充索引：

- `INDEX(train_run_id, seat_id, ticket_status)`
- `INDEX(order_id)`
- `INDEX(passenger_id)`

### 3.14 `payment_record` 支付记录表

| 字段名 | 类型 | 约束 | 说明 |
| --- | --- | --- | --- |
| `id` | `BIGINT` | `PK`, `AUTO_INCREMENT` | 支付记录主键 |
| `payment_no` | `VARCHAR(40)` | `NOT NULL`, `UNIQUE` | 支付流水号 |
| `order_id` | `BIGINT` | `NOT NULL`, `FK -> ticket_order.id` | 对应订单 |
| `pay_amount` | `DECIMAL(10,2)` | `NOT NULL` | 支付金额 |
| `pay_method` | `VARCHAR(20)` | `NOT NULL` | 支付方式 |
| `pay_status` | `ENUM('INIT','SUCCESS','FAILED','EXPIRED','REFUNDED')` | `NOT NULL`, `DEFAULT 'INIT'` | 支付状态 |
| `paid_at` | `DATETIME` | `NULL` | 实际支付时间 |
| `expire_at` | `DATETIME` | `NULL` | 支付失效时间 |
| `created_at` | `DATETIME` | `NOT NULL` | 创建时间 |

补充索引：

- `INDEX(order_id, pay_status)`

### 3.15 `refund_record` 退票记录表

| 字段名 | 类型 | 约束 | 说明 |
| --- | --- | --- | --- |
| `id` | `BIGINT` | `PK`, `AUTO_INCREMENT` | 退票记录主键 |
| `refund_no` | `VARCHAR(40)` | `NOT NULL`, `UNIQUE` | 退票流水号 |
| `ticket_id` | `BIGINT` | `NOT NULL`, `FK -> ticket.id` | 对应车票 |
| `refund_amount` | `DECIMAL(10,2)` | `NOT NULL` | 退款金额 |
| `refund_reason` | `VARCHAR(200)` | `NULL` | 退款原因 |
| `refund_status` | `VARCHAR(20)` | `NOT NULL` | 退款状态 |
| `requested_at` | `DATETIME` | `NOT NULL` | 申请时间 |
| `completed_at` | `DATETIME` | `NULL` | 完成时间 |

### 3.16 `change_record` 改签记录表

| 字段名 | 类型 | 约束 | 说明 |
| --- | --- | --- | --- |
| `id` | `BIGINT` | `PK`, `AUTO_INCREMENT` | 改签记录主键 |
| `change_no` | `VARCHAR(40)` | `NOT NULL`, `UNIQUE` | 改签流水号 |
| `old_ticket_id` | `BIGINT` | `NOT NULL`, `FK -> ticket.id` | 原车票 ID |
| `new_ticket_id` | `BIGINT` | `NOT NULL`, `FK -> ticket.id` | 新车票 ID |
| `fare_diff_amount` | `DECIMAL(10,2)` | `NOT NULL`, `DEFAULT 0.00` | 差价金额 |
| `change_status` | `VARCHAR(20)` | `NOT NULL` | 改签状态 |
| `created_at` | `DATETIME` | `NOT NULL` | 创建时间 |
| `completed_at` | `DATETIME` | `NULL` | 完成时间 |

### 3.17 `waitlist_order` 候补订单表

| 字段名 | 类型 | 约束 | 说明 |
| --- | --- | --- | --- |
| `id` | `BIGINT` | `PK`, `AUTO_INCREMENT` | 候补单主键 |
| `waitlist_no` | `VARCHAR(40)` | `NOT NULL`, `UNIQUE` | 候补单号 |
| `account_id` | `BIGINT` | `NOT NULL`, `FK -> account.id` | 所属账号 |
| `passenger_id` | `BIGINT` | `NOT NULL`, `FK -> passenger.id` | 候补乘客 |
| `train_run_id` | `BIGINT` | `NOT NULL`, `FK -> train_run.id` | 对应开行实例 |
| `from_stop_id` | `BIGINT` | `NOT NULL`, `FK -> train_stop.id` | 上车站 |
| `to_stop_id` | `BIGINT` | `NOT NULL`, `FK -> train_stop.id` | 下车站 |
| `seat_type_id` | `BIGINT` | `NOT NULL`, `FK -> seat_type.id` | 候补座席类型 |
| `preferred_position` | `VARCHAR(20)` | `NULL` | 座位偏好 |
| `waitlist_status` | `ENUM('ACTIVE','FULFILLED','CANCELLED','EXPIRED')` | `NOT NULL`, `DEFAULT 'ACTIVE'` | 候补状态 |
| `created_at` | `DATETIME` | `NOT NULL` | 创建时间 |
| `fulfilled_ticket_id` | `BIGINT` | `NULL`, `FK -> ticket.id` | 兑现后的车票 ID |

补充索引：

- `INDEX(train_run_id, waitlist_status, created_at)`

### 3.18 `notification_message` 通知消息表

| 字段名 | 类型 | 约束 | 说明 |
| --- | --- | --- | --- |
| `id` | `BIGINT` | `PK`, `AUTO_INCREMENT` | 通知主键 |
| `account_id` | `BIGINT` | `NOT NULL`, `FK -> account.id` | 接收账号 |
| `biz_type` | `VARCHAR(30)` | `NOT NULL` | 业务类型，如订单/候补/晚点 |
| `biz_id` | `BIGINT` | `NOT NULL` | 关联业务主键 |
| `channel` | `VARCHAR(20)` | `NOT NULL` | 通知渠道，如站内信/短信 |
| `content` | `TEXT` | `NOT NULL` | 通知内容 |
| `send_status` | `VARCHAR(20)` | `NOT NULL` | 发送状态 |
| `sent_at` | `DATETIME` | `NULL` | 发送时间 |
| `created_at` | `DATETIME` | `NOT NULL` | 创建时间 |

---

## 4. 关键状态字段建议值

| 字段 | 建议值 |
| --- | --- |
| `role` | `USER`, `ADMIN` |
| `run_status` | `NORMAL`, `DELAYED`, `CANCELLED`, `CLOSED` |
| `order_status` | `CREATED`, `PENDING_PAYMENT`, `PAID`, `PARTIAL_REFUNDED`, `REFUNDED`, `CANCELLED`, `COMPLETED` |
| `ticket_status` | `LOCKED`, `ISSUED`, `WAITLISTED`, `REFUNDING`, `REFUNDED`, `CHANGED`, `CANCELLED` |
| `pay_status` | `INIT`, `SUCCESS`, `FAILED`, `EXPIRED`, `REFUNDED` |
| `waitlist_status` | `ACTIVE`, `FULFILLED`, `CANCELLED`, `EXPIRED` |

---

## 5. 说明补充

- `run_leg_inventory` 用于区间余票查询，某区间可售余票取所跨相邻站段 `available_count` 的最小值。
- `ticket` 是座位分配事实表，同一 `train_run_id + seat_id` 下，有效乘车区间不得重叠。
- `ticket_order` 与 `ticket` 是一对多关系，一张订单可以包含多张票。
- `change_record` 通过 `old_ticket_id` 和 `new_ticket_id` 关联原票与新票。
- `waitlist_order` 在库存释放后可兑现为正式车票，并回填 `fulfilled_ticket_id`。
