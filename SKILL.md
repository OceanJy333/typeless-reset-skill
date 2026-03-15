---
name: reset-typeless
description: 重置 Typeless.app 试用期、登出账号、备份/迁移数据。当用户说"重置 typeless"、"typeless 试用"、"清除 typeless"、"备份 typeless"、"迁移 typeless"时调用此 skill。
---

# Typeless 管理工具

管理 Typeless.app 的试用期重置、账号登出、数据备份与跨账号迁移。

参考项目：[mercy719/typeless-migrator](https://github.com/mercy719/typeless-migrator)

## 关键路径

| 路径 | 用途 |
|------|------|
| `~/Library/Application Support/now.typeless.desktop/device.cache` | 设备 UUID（服务端识别设备的唯一标识，自生成，非硬件绑定） |
| `~/Library/Application Support/Typeless/user-data.json` | 加密的认证 token（electron-store + AES-256-CBC 双重 PBKDF2） |
| `~/Library/Application Support/Typeless/app-storage.json` | 明文用户数据、配额、设置缓存 |
| `~/Library/Application Support/Typeless/typeless.db` | SQLite 数据库（history 表，每行有 user_id 字段） |
| `~/Library/Application Support/Typeless/Recordings/` | 录音 .ogg 文件 |
| `~/Library/Application Support/Typeless/app-settings.json` | 快捷键、麦克风等设置 |
| `~/Library/Application Support/Typeless/Cookies` | 登录会话 |
| `~/Library/Application Support/Typeless/Local Storage/` | 前端会话存储 |

## user-data.json 加密细节

```
arch = "arm64" (Apple Silicon) 或 "x64" (Intel)
platform_hash = SHA256("darwin-{arch}").hex()
encryption_key = PBKDF2-SHA256(platform_hash + "Typeless", "typeless-user-service", 10000, 32)

文件格式 = [16字节 IV] + ':' + [AES-256-CBC 密文]
iv_salt = IV.toString('utf-8')  // 无效字节变 U+FFFD
password = PBKDF2-SHA512(encryption_key, iv_salt, 10000, 32)
解密后 JSON 包含: email, access_token, refresh_token, user_id, login_time
```

解密需要 Python: `pip install pycryptodome gmssl requests`

## API 签名协议

Typeless API (`https://api.typeless.com`) 需要自定义安全头：

```
HMAC_KEY = "4466060e8649a69f2478dd40fedabc9c70cdfbc47089c6d19759de03"
AES_PASSWORD = "151907eb389d85aef303509b7ef7a1a0ab8fb0ceb8e46ac4fe5586d5"

签名流程:
1. sign_str = "{timestamp}:{version}:{api_path}:{user_id}"
2. sha1 = HMAC-SHA1(key="{timestamp}:{HMAC_KEY}", data=sign_str)
3. sm3_sign = SM3("{timestamp}:{sha1}:{HMAC_KEY}")
4. X-Authorization = CryptoJS.AES.encrypt(payload_json, AES_PASSWORD)
   payload 包含: X-Env, X-Client-Domain, t, p(sm3_sign), d(device_id)
```

词典数据只存在云端，需要通过 API 备份：
- 列表: `GET /user/dictionary/list?size=200&offset=0`
- 添加: `POST /user/dictionary/add` body: `{term, lang, category}`

## 操作模式

根据用户意图选择对应模式：

---

### 模式 A：重置试用期 + 登出

当用户说"重置"、"清除"、"试用期"时执行。

#### 前置检查
1. 确认 Typeless 没有在运行：`pgrep -fl Typeless`
   - 如果在运行，提示用户先退出（Cmd+Q 或菜单栏退出），**不要强杀进程**

#### 步骤

1. **删除设备指纹**
   ```bash
   rm ~/Library/Application\ Support/now.typeless.desktop/device.cache
   ```

2. **删除加密用户数据**
   ```bash
   rm ~/Library/Application\ Support/Typeless/user-data.json
   ```

3. **清除登录会话**
   ```bash
   rm ~/Library/Application\ Support/Typeless/Cookies ~/Library/Application\ Support/Typeless/Cookies-journal 2>/dev/null
   ```

4. **清除前端会话存储**
   ```bash
   rm -rf ~/Library/Application\ Support/Typeless/Local\ Storage/
   ```

5. **重置 app-storage.json**
   读取文件，只保留 `__internal__` 字段，删除 `userData`、`quotaUsage`、`accessibilityConfig`、`currentRoute` 等：
   ```json
   {
     "__internal__": {
       "migrations": {
         "version": "<保留原版本号>"
       }
     }
   }
   ```

#### 完成确认
告知用户：设备指纹已清除（新 UUID），账号已登出，可用新账号获得新试用期。

---

### 模式 B：备份数据

当用户说"备份 typeless"时执行。

#### 步骤

1. **读取当前账号信息**
   ```bash
   # 从 app-storage.json 读 email
   python3 -c "import json; d=json.load(open('$HOME/Library/Application Support/Typeless/app-storage.json')); print(d['userData']['email'])"
   # 从 typeless.db 读 user_id
   sqlite3 ~/Library/Application\ Support/Typeless/typeless.db "SELECT user_id FROM history WHERE user_id IS NOT NULL LIMIT 1;"
   ```

2. **创建备份目录**
   ```bash
   DEST=~/Backups/typeless/$(date +"%Y-%m-%d_%H-%M-%S")
   mkdir -p "$DEST"
   ```

3. **备份数据库**（即使 Typeless 在运行也安全）
   ```bash
   sqlite3 ~/Library/Application\ Support/Typeless/typeless.db ".backup '$DEST/typeless.db'"
   ```

4. **备份录音文件**
   ```bash
   cp -r ~/Library/Application\ Support/Typeless/Recordings "$DEST/Recordings"
   ```

5. **备份配置文件**
   ```bash
   for f in app-settings.json app-storage.json app-onboarding.json user-data.json; do
     cp ~/Library/Application\ Support/Typeless/$f "$DEST/$f" 2>/dev/null
   done
   ```

6. **写入备份元数据**（backup-meta.json 包含 email、user_id、record_count）

7. **备份词典**（可选，需要 Python 依赖）
   使用 `crypto_utils.py` 解密 token → 调用 `/user/dictionary/list` API 分页获取所有词条 → 保存到 `dictionary_backup.json`

---

### 模式 C：跨账号迁移

当用户说"迁移 typeless"时执行。完整流程：

1. 在旧账号登录状态下执行**模式 B**备份
2. 可选：备份词典（需要 Python）
3. 执行**模式 A**重置
4. 用户登录新账号
5. 迁移本地数据：
   ```bash
   # 复制数据库并替换 user_id
   cp "$BACKUP_DIR/typeless.db" ~/Library/Application\ Support/Typeless/typeless.db
   sqlite3 ~/Library/Application\ Support/Typeless/typeless.db \
     "UPDATE history SET user_id = '$NEW_USER_ID' WHERE user_id = '$OLD_USER_ID';"
   ```
6. 复制录音文件（跳过已存在的）
7. 恢复 app-settings.json
8. 可选：导入词典到新账号（调用 `/user/dictionary/add` API）

---

## 注意事项

- 仅支持 macOS
- 加密密钥基于 Typeless v1.0.4，版本更新可能失效
- 词典操作需要安装 Python 依赖：`pip install pycryptodome gmssl requests`
- 如果用了代理（如 Clash），设置 `HTTPS_PROXY` 环境变量
- 迁移数据库前务必先备份
