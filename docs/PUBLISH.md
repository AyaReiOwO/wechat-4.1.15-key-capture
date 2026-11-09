# 发布清单

## 一、仓的元信息（直接填）

| 项 | 值 |
|---|---|
| 仓库名 | `wechat-4.1.15-key-capture` |
| 描述 | `WeChat 4.1.15 (Windows) DB key extraction — Frida hook on the codec config function, HMAC-verified. 微信 4.1.15 数据库密钥提取` |
| Topics | `wechat` `weixin` `sqlcipher` `frida` `reverse-engineering` `forensics` `wcdb` `windows` |
| License | MIT |
| 首屏要点 | 4.1.15 的 RVA、28,425 次命中 / 959 候选 / 1 把 passphrase、负面结果「key 绑安装实例」 |

## 二、发布顺序

1. 建仓（public）→ 推 README + 三个脚本 + `docs/`
2. 回帖到**正在求助的 issue**（下面三段可直接粘）
3. 看雪发一篇分析（标题：《微信 4.1.15 Windows 数据库密钥提取：从 MMV1 到 passphrase》）
4. Linux.do / V2EX 发短版，只放结论 + 仓链接
5. 一周后补一条：把 MOLY（同版本另一台）的复现结果加进版本矩阵

## 三、可直接粘的回帖

### 1) TANGandXUE/wcdb-key-tool · issue #4（Windows 4.1.12.55 扫不到 key）

> Windows 侧换条路可行：不扫 `Config.Cipher`，直接在 codec 配置函数入口挂钩。
>
> - 4.1.15.56 的地址：`Weixin.dll + 0x35D62E0`（"MMV1" 字符串的 lea 引用 + `.pdata` 反查入口，`scan_lea.py` 现算）
> - Frida **spawn**（attach 会错过），命中时 dump `rcx/rdx/r8/r9` 指向的 64 字节
> - 候选里既有 passphrase 也有各库 enc_key；判据：`PBKDF2-SHA512(passphrase, 库 salt, 256000) == enc_key`，再用 SQLCipher4 首页 HMAC 复核
> - 实测 28,425 次命中 / 959 候选 / 反推出 1 把 passphrase，能开该账号全部库
>
> 代码和复现步骤：https://github.com/AyaReiOwO/wechat-4.1.15-key-capture
> （4.1.12.55 只需把 RVA 换成 `scan_lea.py` 现算的那个）

### 2) sunfmin/wxread · issue #81（macOS 4.1.15 找不到 key）

> Windows 侧同版本已经跑通，思路可以平移：**不要找 `x'<64hex><32hex>'` 字面量**，改为定位 codec 配置函数（`MMV1` 字符串 → lea 引用 → 函数入口），在启动早期挂钩后 dump 参数内存，再用 `PBKDF2(passphrase, salt, 256000) == enc_key` + 首页 HMAC 判定。
>
> macOS 上对应的是 dyld 里找同一特征串的引用，用 LLDB 断在入口（wcdb-key-tool 的 macOS 路线就是这个形状）。
> Windows 版实现与实测数据：https://github.com/AyaReiOwO/wechat-4.1.15-key-capture

### 3) pandorafuture/wx-cli · issue #21（版本白名单拦截）

> 白名单拦的是「按版本查已知偏移」这条路；换成**每次现算**就不需要白名单：`MMV1` 特征串 → RIP-relative lea → `.pdata` 函数入口，三步就能拿到当前版本的地址。
>
> 现算 + Frida spawn 挂钩的完整实现在 Windows 4.1.15.56 上验过：https://github.com/AyaReiOwO/wechat-4.1.15-key-capture

## 四、想要星星，靠这三件事

1. **一个能跑的最小复现**（已具备：三条命令 + 版本矩阵）
2. **一个别人没有的负面结果**（key 绑安装实例、KDF 参数轮试表）——这类信息最容易被引用
3. **出现在别人正在搜的 issue 里**（上面三段），比任何推广都有效

## 五、推送命令（QIN 上执行）

```bash
cd ~/GitHub/wechat-4.1.15-key-capture
git init -b main
git add -A
git commit -m "feat: WeChat 4.1.15 (Windows) key capture via codec-config hook"
# 有 gh 就用它建仓并推：
gh repo create wechat-4.1.15-key-capture --public --source=. --push \
  --description "WeChat 4.1.15 (Windows) DB key extraction — Frida hook + HMAC verification"
# 没 gh 就手动建空仓后：
git remote add origin git@github.com:<you>/wechat-4.1.15-key-capture.git
git push -u origin main
```

推完在仓库 Settings → Topics 里补上面那 8 个标签（GitHub 搜索靠它们）。
