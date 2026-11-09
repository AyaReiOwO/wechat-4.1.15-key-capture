# 微信 4.1.15（Windows）数据库密钥提取

> 4.1.10 起微信不再在进程里保留「拿起来就能用」的数据库密钥；到 **4.1.15**，公开渠道能查到的多是「扫不到 key」的求助。本仓给出一条在 **Windows 4.1.15.56 实测跑通**的路径：不扫内存，在 codec 配置函数入口挂一次钩子，把密钥材料捞出来再判定。

- 不是内存扫描（`x'<64hex><32hex>'` 那套在 4.1.15 已经没了）
- 不是改客户端文件（只在运行时下断点，读一次寄存器）
- 提供判据：抓到的候选必须通过 SQLCipher4 首页 HMAC 才算数，不做猜测

## 原理

```
特征串 "MMV1"（4.1.15.56 里有 3 处 ascii）
   ↓ 找 RIP-relative lea 引用
lea 0x35D6319  →  rva 0x96789F8
   ↓ .pdata 反查函数边界
函数入口 RVA 0x35D62E0   （序言 48 83 ec 48 48 8b 05 … 48 31 e0 …）
   ↓ Frida spawn 早期挂钩（attach 会错过登录期计算）
命中时读 rcx/rdx/r8/r9 指向的内存
   ↓ 候选集里既有 passphrase 也有各库 enc_key
HMAC 校验 + PBKDF2 反推  →  passphrase
```

- 加密参数（微信 4.1.x）：SQLCipher 4，AES-256-CBC + HMAC-SHA512，page 4096，reserve 80，IV@4016，HMAC@4032
- 派生：`PBKDF2-HMAC-SHA512(passphrase, 该库文件头 16 字节 salt, 256000, 32)` → 每库独立的 enc_key
- 校验：`mac_key = PBKDF2-HMAC-SHA512(enc_key, salt^0x3a, 2, 32)`，`HMAC-SHA512(mac_key, page1[16:4032] + uint32le(1)) == page1[4032:4096]`

## 版本矩阵

| 微信版本 | 平台 | codec 配置函数 | 状态 |
|---|---|---|---|
| 4.1.12.26 | Windows | `0x3486140` | 社区已有（[stargazer-2026/wechat-4.1.12-decrypt](https://github.com/stargazer-2026/wechat-4.1.12-decrypt)） |
| **4.1.15.56** | **Windows** | **`0x35D62E0`** | **本仓实测跑通**（Weixin.dll 204,305,968 字节 / SHA256 `E0428A7B…C66556`） |
| 其它 4.1.x | Windows | `scan_lea.py` 现算 | 未测；地址会变，方法不变 |
| macOS / Linux 4.1+ | — | 见 wcdb-key-tool（LLDB / GDB） | 本仓不覆盖 |

## 快速开始

```bash
pip install frida

# 1) 现算函数地址（换版本也能用）
python scan_lea.py --dll "C:\Program Files\Tencent\Weixin\4.1.15.56\Weixin.dll" \
                   --pattern MMV1 --json rva.json

# 2) spawn 微信并挂钩子（会重启微信，需要扫码或点「进入微信」）
python frida_hook.py --rva 0x35D62E0 --out hits.jsonl --seconds 600

# 3) 判定：先用 HMAC 找 enc_key，再用 PBKDF2 反推 passphrase
python validate.py --hits hits.jsonl \
                   --db-dir "C:\Users\<you>\Documents\xwechat_files\<wxid>_<hash>\db_storage"

# 4) 只验证一把 passphrase 能开多少库
python validate.py --passphrase <64位hex> --db-dir <db_storage>
```

`frida_hook.py` 只读：dump 四个寄存器参数指向的内存，不写目标进程、不碰网络。

## 实测数据（2026-11-09，Windows 4.1.15.56）

| 项 | 值 |
|---|---|
| 挂钩命中 | 28,425 次（每次开一个库触发一次） |
| 去重后候选（32 字节） | 959 个 |
| 直接命中 enc_key | 1 个 |
| PBKDF2 反推出的 passphrase | 1 个，能开该账号的库（示例 `contact.db` 首页 HMAC 通过） |
| 全流程耗时 | 挂钩 3 分钟 / 判定 9 秒（16 核并行 PBKDF2） |

结论：**key 绑安装实例**。换一台机器（哪怕同版本同账号）的备份，passphrase 不通用——所以要解旧备份，必须在写出那份数据的安装上抓。

## 已知限制

- 只覆盖 Windows 4.1.15.56；升级后必须重跑 `scan_lea.py` 取新 RVA
- 必须 `spawn`，`attach` 到已登录进程会错过登录期的计算
- 部分版本存在 hook 检测，挂载后可能弹错误报告；spawn 早期挂载是目前最稳的姿势
- 解不了「已重装/已换机」的旧库（key 已随安装实例变掉）
- 3.x 的库不适用（SQLCipher 3，参数不同）

## 合规

只用于提取**本人设备、本人账号**的数据库密钥；全程不接触微信服务器，不修改客户端文件。

## Credits

- [stargazer-2026/wechat-4.1.12-decrypt](https://github.com/stargazer-2026/wechat-4.1.12-decrypt)：4.1.12 上「MMV1 → codec 函数 → Frida spawn」的原始思路
- [TANGandXUE/wcdb-key-tool](https://github.com/TANGandXUE/wcdb-key-tool)：SQLCipher 参数、PBKDF2 与 HMAC 判定、Linux/macOS 的 GDB/LLDB 路线
- `scan_lea.py` 出自同一套自用工具链（PE 静态定位：字符串 → lea → `.pdata`）

## License

MIT
