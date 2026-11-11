# Elysium LP 重开监测 (Elysium LP Monitor)

纯只读看板网页，供委托方在手机/电脑浏览器开一个窗口常驻观看 Elysium 链（chainId 99901）上的自动 LP 哨兵监控数据。

- **在线访问**: [https://jayxsodo.github.io/elysium-watch/](https://jayxsodo.github.io/elysium-watch/)
- **单文件入口**: `index.html`（内联原生 CSS / JS，零外部 CDN 依赖，纯原生 JSON-RPC + BigInt 运算）
- **安全规格**: 纯只读，绝不连接钱包、不签名、不发送任何交易，无任何私钥/助记词。

---

## 一、数据口径与运算逻辑

1. **KNTQ 现价 (HYPE 计价)**:
   - 调用 Elysium 链上 StateView 合约 `0x27eD73aE0652fdCf71d6793412A7525f32bE9dfE`。
   - `data = "0xc815641c" + poolId`。
   - 主池 poolId 为 `c96bfd21d36f8e7252f0d80fcb5f3bcacdf47c8b927b20711104713ebeb383ac`（KNTQ/HYPE 1% 费率主池）。
   - 解析返回结果 word0 得到 `sqrtPriceX96`。
   - 计算公式：$q = \frac{2^{192}}{\text{sqrtP}^2}$（每 KNTQ 多少 HYPE）。大数使用 `BigInt` 精确运算。

2. **Elysium 溢价**:
   - 调用 HyperEVM RPC `https://rpc.hyperliquid.xyz/evm`。
   - 调用池合约 `0xC8e5EE73b80bb3f6c9b5348771De8217aBD412E7`，`data = "0xe76c01e4"`。
   - 解析 word0 为 `sqrtPrice`。
   - 计算公式：$\text{hyp} = \frac{\text{sqrt}^2}{2^{192}}$。
   - 溢价% 计算公式：$\text{溢价}\% = \left(\frac{q}{\text{hyp}} - 1\right) \times 100$。

3. **成交额与笔数 (近 30 分钟与近 5 分钟)**:
   - 调用 `eth_getLogs`，地址为 PoolManager `0xE2e105d3F7209A9DA11f83cCf3E7398a753823F1`。
   - topics 为 `[SwapTopic, "0x" + poolId]`。
   - `SwapTopic = 0x40e9cecb9f5f1f1c5b9c97dec2917b7ee92e57ba5563708daca94dd84ad7112f`（即 `keccak256("Swap(bytes32,address,int128,int128,uint160,uint128,int24,uint24)")`）。
   - 每条日志 data 的 word0 为 `amount0`（int128 符号扩展至 256 位，还原补码负数）。
   - 取 $|amount0|$ 求和除以 $10^{18}$ 得到 HYPE 成交量，乘以 HYPE 美元价换算为美元成交额；日志数量即为笔数。
   - 区块窗口：由出块速率（约 6.7 blocks/s）与目标时间戳逼近定位 30 分钟前与 5 分钟前起点块；单次跨度超过 4000 块时分段合并；后续刷新增量同步并去重。

4. **HYPE 美元价**:
   - 首选：POST `https://api.hyperliquid.xyz/info`，参数 `{"type":"allMids"}`，读取 `mids.HYPE`。
   - 备选降级：Elysium 本地 USDC/HYPE 池（poolId = `b3032695ffb0a66e6342876dd2386835471466cb2bb70f341947d98a69e265cc`，currency0=HYPE, currency1=USDC），换算得出。

5. **LP 钱包与代币余额**:
   - LP 钱包地址：`0xA2eC4c3874640d18f97A1A6B2c725a371A426f9a`。
   - 原生 HYPE 余额：`eth_getBalance`。
   - KNTQ 余额：调用 KNTQ 合约 `0xe45a355d9e9cAd2aD85F2d03E3eE3102A036acF0` 的 `0x70a08231` (balanceOf)。若余额 > 1，页面红标提示「疑似已开仓，请核对」。

---

## 二、构建时实测项与 CORS 验证结果

| 读取项 / 节点 | 目标方法 / 接口 | 实测状态 | CORS 表现 | 备注与实测值 |
| :--- | :--- | :--- | :--- | :--- |
| **Elysium RPC 1** (`https://rpc-live.boxa.club`) | JSON-RPC (eth_blockNumber, eth_call) | **受限** | ❌ 拦截 (HTTP 403) | 携带页面 Origin 时返回 `This RPC is not available to this site`，页面自动容错切换至备选节点 |
| **Elysium RPC 2** (`https://rpc.elysium.kinetiq.xyz`) | JSON-RPC 全量读取 | **成功** | ✅ 允许 (`*`) | 响应正常，延迟稳定，作为主工作节点 |
| **HyperEVM RPC** (`https://rpc.hyperliquid.xyz/evm`) | `eth_call` (slot0) | **成功** | ✅ 允许 (`*`) | 成功读出 hyp sqrtPrice，计算得溢价 +2.37% |
| **Hyperliquid Info API** (`https://api.hyperliquid.xyz/info`) | POST `allMids` | **成功** | ✅ 允许 (`*`) | 成功读出 HYPE 实时现价（实测 $71.93） |
| **StateView KNTQ 现价** | `eth_call` 0xc815641c... | **成功** | ✅ 正常 | 实测 KNTQ 现价 0.004263 HYPE（≈ $0.3066） |
| **钱包 HYPE / KNTQ 余额** | `eth_getBalance` / `balanceOf` | **成功** | ✅ 正常 | 实测 HYPE: 7.4043，KNTQ: 0.0000 |
| **Swap 成交日志与窗口统计** | `eth_getLogs` (PoolManager) | **成功** | ✅ 正常 | 30m 成交约 $1,106 (17笔)；5m 约 $151 (3笔)；同量级契合验收基线 |

---

## 三、已知限制与降级策略

1. **首选 RPC 节点限制**: `rpc-live.boxa.club` 对浏览器跨域请求有站点白名单限制，页面内建了无缝容错切换，自动优先使用 `rpc.elysium.kinetiq.xyz`。
2. **溢价速度（第 4 条触发线）采样限制**: 溢价速度依赖当前浏览器页面的本地采样历史（存储于 `localStorage`，键名 `elysium_watch_prem_hist`，保留近 2 小时）。用户首次打开页面时，历史记录不足 4 分钟前会如实显示「采样中」，历史累积超过 4 分钟后自动计算真实 5 分钟涨幅。
3. **后台休眠控制**: 当浏览器标签页切入后台（`document.hidden` 为 true）时，定时刷新自动暂停以避免不必要的 RPC 消耗；标签页切回前台时立即重新触发一轮刷新。
