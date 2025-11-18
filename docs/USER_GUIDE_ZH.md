# PocketBase Python SDK 新手完整指南

本文件以中文整理 PocketBase Python SDK 的常用操作，幫助從官方 JavaScript SDK 轉到 Python 的新手快速上手。內容涵蓋安裝、認證、CRUD、篩選、檔案上傳、即時訂閱、與 JavaScript SDK 的差異等。

## 與官方 JavaScript SDK 的對照

Python 版刻意維持與官方 JavaScript SDK 相同的服務名稱與方法（例如 `collection(id_or_name)`、`auth_with_password`、`get_list` 等），但有以下差異：

- **命名風格**：預設會把 API 回傳的 `camelCase` 轉成 `snake_case` 欄位，例如 `created`、`collection_id`。若想保留原始鍵名，可在建立客戶端時設定 `auto_snake_case=False`。
- **HTTP 客戶端**：Python 版使用 `httpx.Client`，但對使用者而言介面仍是同步方法。
- **檔案上傳**：改以 `FileUpload` 包裝 Python 檔案物件即可，多檔案時可以丟入 tuple/list。

> 心法：只要知道 JavaScript SDK 的用法，Python 版幾乎是「同名方法 + Python 語法糖」。

## 安裝與初始化

```bash
python3 -m pip install pocketbase
```

建立客戶端：

```python
from pocketbase import PocketBase

client = PocketBase("http://127.0.0.1:8090", auto_snake_case=True)
```

重要參數：

- `base_url`：PocketBase 伺服器位址。
- `auth_store`：可自訂 Token 儲存策略，預設為記憶體儲存。
- `timeout`：單位秒，預設 120。
- `auto_snake_case`：是否將 API 回應鍵名轉成 snake_case。

## 使用者與管理員認證

### 註冊/登入（集合帳號）

```python
# 以 email/密碼登入 auth collection（預設 users）
user_auth = client.collection("users").auth_with_password(
    "user@example.com", "0123456789"
)
print(user_auth.token)
print(user_auth.record)  # 已驗證的 Record 物件
```

其他常用認證操作：

- OAuth2 登入：`auth_with_oauth2(provider, code, code_verifier, redirect_url, create_data=None)`
- 更新 token：`auth_refresh()`
- 信箱驗證與重設：`request_verification(email)`, `confirm_verification(token)`, `request_password_reset(email)`, `confirm_password_reset(token, password, password_confirm)`

### 管理員登入

```python
admin_auth = client.admins.auth_with_password("admin@example.com", "pwd")
print(admin_auth.token)
print(admin_auth.admin)  # Admin 物件
```

## 讀取資料（查詢與篩選）

PocketBase 的所有資料列都透過 `collection("your_collection")` 取得 `RecordService`，常用查詢方法：

```python
service = client.collection("posts")

# 分頁查詢：page=1、per_page=20，並套用篩選與排序
result = service.get_list(
    page=1,
    per_page=20,
    query_params={
        "filter": 'status = true && created > "2024-01-01"',
        "sort": "-created",  # 以 created 由新到舊
        "expand": "author,comments",  # 展開關聯
    },
)

print(result.items)       # 取得 Record 物件列表
print(result.total_items) # 總筆數
print(result.total_pages)
```

常見查詢技巧：

- **單筆查詢**：`service.get_one(record_id)`
- **依條件取第一筆**：`service.get_first_list_item('title ~ "Python"')`
- **取完整清單（自動分頁迭代）**：`service.get_full_list(batch=200, query_params={...})`
- **展開關聯**：`query_params` 帶入 `expand="author,comments"`，展開結果會出現在 `record.expand`。
- **排序**：`sort="-created,title"`（負號代表 DESC）。
- **文字搜尋**：在 `filter` 中使用 PocketBase 的查詢語法，如 `category = "tech" && created >= "2024-01-01"`。

## 新增、更新、刪除

```python
from pocketbase.client import FileUpload

service = client.collection("posts")

# 建立資料（含檔案上傳）
created = service.create({
    "title": "Hello PocketBase",
    "status": True,
    "image": FileUpload(("cover.png", open("cover.png", "rb"))),
})

# 更新資料（只要送出的欄位會被覆蓋）
updated = service.update(
    created.id,
    {"title": "Updated title"}
)

# 刪除資料
service.delete(created.id)
```

小提醒：若目前的 `auth_store` 保存的就是被更新或刪除的紀錄，SDK 會自動同步或清除 token 及模型，維持登入狀態一致。

## 即時更新（Realtime）

```python
def on_message(msg):
    print("收到變更", msg)

# 訂閱整個集合
sub_id = service.subscribe(on_message)

# 取消訂閱整個集合或特定紀錄
service.unsubscribe()            # 取消集合訂閱
service.unsubscribe(created.id)  # 只取消某筆
```

回呼 `msg` 內容與 JavaScript 版相同，包含 `action`（create/update/delete）與資料快照。

## 檔案網址與下載

```python
record = service.get_one(created.id)
file_url = service.get_file_url(record, record.image)

# 或透過 client.files 直接產生網址
file_url = client.get_file_url(record, record.image)
```

`query_params` 可附加縮圖、token 等參數：`get_file_url(record, filename, {"thumb": "100x100"})`。

## 實戰範例：醫院藥局排班手冊（含 Python/官方 JavaScript 寫法）

以下以「醫院藥局排班系統」為例，示範如何在 PocketBase 中完成常見的排班操作，並同時提供 Python 及官方 JavaScript SDK 的寫法供對照。假設有以下集合：

- `pharmacists`：藥師基本資料（`name`、`department`、`license_no`、`is_active` 等）。
- `departments`：部門列表（門診/住院/調劑等）。
- `shifts`：排班主表，欄位包含 `pharmacist`（Relation 到 `pharmacists`）、`department`、`date`、`shift_type`（早/中/晚/大夜）、`status`（draft/published/leave）、`note`。
- `leave_requests`：請假/代班申請（`pharmacist`、`start_date`、`end_date`、`reason`、`status`）。

> PocketBase 的欄位可依實際需要調整，以下範例聚焦操作流程與 API 用法。

### 初始化客戶端

```python
from pocketbase import PocketBase

client = PocketBase("http://127.0.0.1:8090")
```

```javascript
import PocketBase from 'pocketbase'

const client = new PocketBase('http://127.0.0.1:8090')
```

### 1. 取出指定日期區間的排班

查詢 2024-07-01 到 2024-07-07 的藥師排班，並展開藥師與部門名稱。

```python
schedule = client.collection("shifts").get_list(
    1,
    200,
    query_params={
        "filter": 'date >= "2024-07-01" && date <= "2024-07-07"',
        "expand": "pharmacist,department",
        "sort": "date,shift_type",
    },
)

for shift in schedule.items:
    pharmacist = shift.expand.get("pharmacist")
    dept = shift.expand.get("department")
    print(shift.date, shift.shift_type, pharmacist.name, dept.name)
```

```javascript
const schedule = await client.collection('shifts').getList(1, 200, {
  filter: 'date >= "2024-07-01" && date <= "2024-07-07"',
  expand: 'pharmacist,department',
  sort: 'date,shift_type',
})

schedule.items.forEach((shift) => {
  const pharmacist = shift.expand?.pharmacist
  const dept = shift.expand?.department
  console.log(shift.date, shift.shift_type, pharmacist?.name, dept?.name)
})
```

### 2. 建立新排班（單筆或批量）

新增 2024-07-02 早班的排班資料；若要一次新增多筆，可使用 `create_bulk`（Python）或 `create` 搭配陣列（JS）。

```python
service = client.collection("shifts")

created = service.create({
    "pharmacist": "pharmacist_id",
    "department": "dept_id",
    "date": "2024-07-02",
    "shift_type": "morning",
    "status": "draft",
})

# 批量新增（避免逐筆往返）
service.create_bulk([
    {
        "pharmacist": "ph1",
        "department": "dept_opd",
        "date": "2024-07-03",
        "shift_type": "evening",
        "status": "draft",
    },
    {
        "pharmacist": "ph2",
        "department": "dept_ipd",
        "date": "2024-07-03",
        "shift_type": "night",
        "status": "draft",
    },
])
```

```javascript
const service = client.collection('shifts')

const created = await service.create({
  pharmacist: 'pharmacist_id',
  department: 'dept_id',
  date: '2024-07-02',
  shift_type: 'morning',
  status: 'draft',
})

// JS SDK 沒有 create_bulk，但可以用 Promise.all 一次送出多筆
await Promise.all([
  service.create({
    pharmacist: 'ph1',
    department: 'dept_opd',
    date: '2024-07-03',
    shift_type: 'evening',
    status: 'draft',
  }),
  service.create({
    pharmacist: 'ph2',
    department: 'dept_ipd',
    date: '2024-07-03',
    shift_type: 'night',
    status: 'draft',
  }),
])
```

### 3. 修改排班或換班

將某筆排班的 `pharmacist` 改為另一位藥師，並加上備註。

```python
updated = client.collection("shifts").update(
    "shift_id",
    {
        "pharmacist": "new_pharmacist_id",
        "note": "門診請求互換",
    },
)
```

```javascript
const updated = await client.collection('shifts').update('shift_id', {
  pharmacist: 'new_pharmacist_id',
  note: '門診請求互換',
})
```

### 4. 請假與代班

在請假單集合寫入請假紀錄，並更新排班狀態為 `leave`，同時指派代班人員。

```python
leave = client.collection("leave_requests").create({
    "pharmacist": "ph1",
    "start_date": "2024-07-05",
    "end_date": "2024-07-05",
    "reason": "家庭事假",
    "status": "approved",
})

client.collection("shifts").update(
    "shift_id",
    {
        "status": "leave",
        "pharmacist": "cover_pharmacist_id",
        "note": "代班：張藥師",
    },
)
```

```javascript
const leave = await client.collection('leave_requests').create({
  pharmacist: 'ph1',
  start_date: '2024-07-05',
  end_date: '2024-07-05',
  reason: '家庭事假',
  status: 'approved',
})

await client.collection('shifts').update('shift_id', {
  status: 'leave',
  pharmacist: 'cover_pharmacist_id',
  note: '代班：張藥師',
})
```

### 5. 查詢排班缺口與部門覆蓋率

利用 `filter` 檢查某日期、部門的班別是否已排滿（例如早中晚都需要一名）。

```python
needed_shifts = {"morning", "evening", "night"}
existing = client.collection("shifts").get_full_list(
    query_params={
        "filter": 'date = "2024-07-06" && department = "dept_opd"',
    },
)

assigned_types = {s.shift_type for s in existing}
missing = needed_shifts - assigned_types
print("缺少班別：", missing)
```

```javascript
const existing = await client.collection('shifts').getFullList({
  filter: 'date = "2024-07-06" && department = "dept_opd"',
})

const assignedTypes = new Set(existing.map((s) => s.shift_type))
const needed = new Set(['morning', 'evening', 'night'])
const missing = [...needed].filter((t) => !assignedTypes.has(t))
console.log('缺少班別：', missing)
```

### 6. 發佈或鎖定排班

將草稿排班統一更新為已發佈（`published`），避免被誤改：

```python
drafts = client.collection("shifts").get_full_list(
    query_params={"filter": 'status = "draft" && date >= "2024-07-01"'},
)

for draft in drafts:
    client.collection("shifts").update(draft.id, {"status": "published"})
```

```javascript
const drafts = await client.collection('shifts').getFullList({
  filter: 'status = "draft" && date >= "2024-07-01"',
})

await Promise.all(
  drafts.map((draft) =>
    client.collection('shifts').update(draft.id, { status: 'published' }),
  ),
)
```

### 7. 即時監控排班異動

訂閱 `shifts` 集合，當有新增/修改/刪除時即時通知排班管理員。

```python
def on_shift_change(msg):
    print("排班異動：", msg["action"], msg.get("record"))

sub_id = client.collection("shifts").subscribe(on_shift_change)

# ...需要時取消訂閱
client.collection("shifts").unsubscribe(sub_id)
```

```javascript
const unsubscribe = await client.collection('shifts').subscribe('*', (msg) => {
  console.log('排班異動：', msg.action, msg.record)
})

// 需要時呼叫 unsubscribe()
```

### 8. 匯出排班（例如 CSV）

可由 `shifts` 集合查詢後自行轉成 CSV，以下以 Python 為例：

```python
import csv

items = client.collection("shifts").get_full_list(
    query_params={
        "filter": 'date >= "2024-07-01" && date <= "2024-07-31"',
        "expand": "pharmacist,department",
        "sort": "date,shift_type",
    },
)

with open("pharmacy_schedule_2024_07.csv", "w", newline="", encoding="utf-8") as f:
    writer = csv.writer(f)
    writer.writerow(["日期", "班別", "藥師", "部門", "備註"])
    for s in items:
        p = s.expand.get("pharmacist")
        d = s.expand.get("department")
        writer.writerow([s.date, s.shift_type, p.name, d.name, s.note or ""])
```

> JS 端可使用陣列轉換成 CSV 字串或導入 `papaparse`/`json2csv` 等工具；PocketBase 只負責資料查詢與存取。

## 其他服務

- **Collections**：`client.collections` 取得/建立/更新集合定義。
- **Admins**：`client.admins` 管理員 CRUD 與登入。
- **Backups、Logs、Settings、Health**：對應後端 API，方法名稱與 JavaScript 版一致。

## 錯誤處理

任何 4xx/5xx 回應都會丟出 `ClientResponseError`，可取得 `status`、`data` 等資訊：

```python
from pocketbase.errors import ClientResponseError

try:
    client.collection("posts").get_one("bad-id")
except ClientResponseError as err:
    print(err.status)
    print(err.data)
```

## 維持與官方文件同步的技巧

- 官方 API 範例幾乎可以照抄，唯獨要改成 Python 語法與 snake_case 欄位。
- 若需要原始鍵名以方便與前端共用，可在建立客戶端時設 `auto_snake_case=False`，`Record` 會依原樣保留欄位名稱。
- 多檔案上傳時，用 `FileUpload(("image1.png", open(...)), ("image2.png", open(...)))`。

透過以上步驟，新手即可快速掌握 PocketBase Python SDK 的主要功能，並與官方 JavaScript SDK 保持同樣的開發流程。
