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
