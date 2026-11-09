# 实测记录 · 2026-11-09

一台 Windows 机器，微信 4.1.15.56（x64），账号已登录，`documents\xwechat_files\<wxid>_<hash>\db_storage` 下 28 个库。

## 1. 先确认「老办法」确实死了

| 路线 | 结果 |
|---|---|
| 内存扫描 `x'<64hex><32hex>'` | 无命中（4.1.11+ 已不再缓存这个形态） |
| 运行时扫 `Config.Cipher` 对象 | `nodes=245, candidates=50`，HMAC 全不过（对象偏移随版本挪了） |
| `sqlite3_key` 外壳 / `sqlite3_key_v2` / `PRAGMA key` | 断不到有效值 |
| 3.9.x 客户端重登 | 服务端强制升级，登不上 |

## 2. 定位函数

```
Weixin.dll  204,305,968 字节
SHA256      E0428A7B691853DE496532523C2C09AA095F90941C37C9F4CFB3F137F3C66556

scan_lea.py --pattern MMV1 --pattern Config.Cipher
  off=0x76463d4 rva=0x7646FD4  ascii
  off=0x9676ff8 rva=0x96789F8  ascii   ← 被 lea 引用
  off=0x91f1079 rva=0x91F2A79  ascii
  lea 0x35D6319 -> 0x96789F8   func=0x35D62E0
  候选函数 0x35D62E0  48 83 ec 48 48 8b 05 55 c5 d4 07 48 31 e0 48 89
```

`0x35D62E0` 落在 `R E` 段内，序言是 `sub rsp, 0x48` + 栈保护 cookie，函数入口无疑。

## 3. 挂钩与抓取

```
frida.spawn("Weixin.exe") → base=0x7ffd8fd10000
Interceptor.attach(base + 0x35D62E0)
点「进入微信」后连续命中
```

- 命中 **28,425** 次（每开一个库触发一次；`-wal` 回放也会触发）
- 每次 dump `rcx/rdx/r8/r9` 各 64 字节，写入 JSONL
- 去重后得到 **959** 个 32 字节候选（同一 dump 的后半段会把候选放大十几倍，判定时只取前 32 字节）

## 4. 判定

```
候选 959 个，待验证库 1 个
  ✅ enc_key 直命：contact.db ← 1ee81f99…（1 个候选）
🎯 passphrase = 29830854…（后略）
   能开 1/1 个库
```

- 第一步：候选本身当 enc_key，用 SQLCipher4 首页 HMAC 验 —— 命中 1 个
- 第二步：`PBKDF2-SHA512(候选, 该库 salt, 256000) == 命中那把 enc_key` —— 反推出 passphrase
- 第三步：换库复核（`session.db` 等），同一把 passphrase 派生后全部通过

耗时：挂钩 3 分钟（含人工点登录）、判定 9 秒（16 核并行 PBKDF2）。

## 5. 一个重要的负面结果

拿同一把 passphrase 去解**另一台同版本机器**留下的备份（83 个分片），四个库全不过；把 KDF 换成 SHA1 / SHA256、迭代数换成 64000 / 256000 / 1024 / 4000 也都不对。

结论：**4.x 的密钥绑安装实例**。重装、换机、退出重登都可能重新生成，所以解旧备份必须在写出那份数据的安装上抓，跨机器不通用。

## 6. 复现清单

1. `pip install frida`
2. `python scan_lea.py --dll <Weixin.dll> --pattern MMV1`
3. `python frida_hook.py --rva <上一步的函数入口> --out hits.jsonl`
4. 微信里登录（或点「进入微信」）
5. `python validate.py --hits hits.jsonl --db-dir <db_storage>`
6. 拿到 passphrase 后按 SQLCipher 参数逐库派生、解密
