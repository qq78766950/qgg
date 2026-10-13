# heiliao.com 去广告规则 · 使用说明（Quantumult X）

配套文件：`heiliao-qx.conf`　　版本：v2（2026-10-13）

---

## 一、这份规则能去掉什么

| 广告形态 | 由谁渲染 | 处理方式 | 效果 |
|---|---|---|---|
| 弹窗浮层（`.pgx-slot[data-slot=pop_notice]`）| `/static/pgx/slot-ads.js` | 该 JS 返回空 | 弹窗再也弹不出来 |
| 横幅 Banner（`data-slot=b1/b2`，950:2021 竖版大图）| 同上 | 同上 | 广告位清空 + 占位框隐藏，不留空白 |
| 首页网格广告（`pop_grid`）、详情页上下标签（`c1/c2`）| 同上 | 同上 | 全部消失 |
| 浮窗广告、"web弹窗应用中心" | 同上 | 同上 | 全部消失 |
| **列表内嵌广告卡**（首页 15 个 / 第2页 2 个 / 后续页随机）| 服务端直出空壳 + `slot-ads.js` 回填 | 掐 JS + CSS 收壳 | 空壳与裂图一起消失（← v2 修复）|
| 底部弹出弹窗（`.bottom-ads`）| `/html/popup-feed.json` | 该 JSON 返回空 | 永不出现 |
| 第三方统计/追踪（GTM、Yandex、addtoany、dc-report.cc）| — | 域名 REJECT | 一并封掉 |
| 广告落地页（64 个棋牌/菠菜/交友域名 + 2 个 IP）| — | 域名/IP REJECT | 误点也跳不过去 |

**正文图片和视频完全不受影响**。正文缩略图走 `z-image-loader-url`（`/upload_01/`、`/upload/`），广告图单独走 `/hc237/`，实测零重叠。

---

## 二、v2 修复了什么：列表里「每隔一行一张图无法显示」

### 现象
首页、第 2 页的列表里，**每隔一个格位就有一张图裂开**（无法显示）。

### 不是误杀，是「广告壳没被隐藏」

抓取真实页面后确认，那不是一个"被拦截的正文图"，而是一个**服务端直出的广告卡空壳**：

```html
<div class="video-item">
  <a class="cursor-pointer pgx-card" data-card="54779" data-page_key="index" data-page_name="首页">
    <div class="video-item-img">
      <div class="ishot">热</div>
      <div class="mask"></div>
      <div class="placeholder-img">
        <img class="fill-img" alt="黑料网免费吃瓜">   ← 既没有 src，也没有 z-image-loader-url
      </div>
    </div>
  </a>
</div>
```

- 这个 `<a>` **没有 `href`**，封面 `<img>` **一个图片地址都没有**；
- 它的 `href` 和封面图，原本由 `slot-ads.js` 的 `hydrateCard()` 在运行时回填；
- 我们把 `slot-ads.js` 掐成空体后，没人回填 → 这个 `<img>` 永远无图可显示 → 裂图。

实测各页广告壳数量（服务端固定投放，不是随机）：

| 页面 | 列表格位 | 其中广告壳 | 序列 |
|---|---|---|---|
| 首页 | 30 | **15** | `CT AD CT AD CT AD …`（严格交替，就是你看到的"每隔一行"）|
| `/page/2/` | 30 | 2 | `CT AD CT AD CT CT CT …` |
| `/page/3/` 及以后 | 30 | 0 | 全 `CT` |

### 为什么可以确定正文图片没被误杀

1. `pic.hdhwqx.cn` **从未出现在** `[filter_local]` 或 `[mitm]` 里 → 图片请求根本没被拦截；
2. 抽样 20 张正文缩略图逐个实测：全部 `HTTP 200`，真实字节（如 273,440 字节）；
3. 裂掉的那个 `<img>` 本来就没有任何 URL，**没有任何东西被"拦截"**；
4. 新加的 CSS 选择器实测命中数 = 广告壳数，正文卡片 0 误伤：

| 页面 | 命中的广告壳 | 完好保留的正文卡片 |
|---|---|---|
| 首页 | 15 | 15 |
| `/page/2/` | 2 | 28 |
| `/page/3/`、`/page/5/` | 0 | 30 |

### 修复方式
在注入的 CSS 里补上针对性选择器（见 `heiliao-qx.conf` 最后一条 rewrite）：

```css
/* ① 空广告位 / 广告空壳 / 无源图片，一起收掉 */
div.pgx-slot,
.bottom-ads,
a.pgx-card[data-card]:not([href]),
img.fill-img:not([z-image-loader-url]):not([src]) { display: none !important }

/* ② 连网格格位一起收掉，避免留下空白行（:has 需 iOS 15.4+，不支持时仅本条失效）*/
.video-item:has(>a.pgx-card[data-card]:not([href])) { display: none !important }
```

判据说明：
- 正文卡片是 `<a class="cursor-pointer" href="/archives/…">` —— 有 `href`、**没有** `pgx-card`；
  广告壳是 `<a class="cursor-pointer pgx-card" data-card="…">` —— **没有** `href`。
  所以 `:not([href])` 能把两者干净分开。
- 正文缩略图 100% 带 `z-image-loader-url`，广告壳的 `<img>` 一个都没有。

---

## 三、安装（3 步）

1. **合并配置**：把 `heiliao-qx.conf` 里的 `[filter_local]`、`[rewrite_local]`、`[mitm]` 三段，分别并入你 Quantumult X 配置里的同名段落。
   - `[mitm]` 用了 `%APPEND%`，**不会覆盖**你原有的 hostname 列表。
   - 如果你原本没有 `[filter_local]` 段，把 `;final, direct` 的注释去掉并按你的策略改。

2. **开启重写**：Quantumult X → 设置 → 打开「重写（Rewrite）」。
   建议再打开「MitM over HTTP/2」（部分页面是 HTTP/2）。

3. **安装并信任证书**：Quantumult X → MitM → 生成证书 → 安装描述文件 →
   设置 → 通用 → VPN与设备管理 → 安装 → 通用 → 关于本机 → 证书信任设置 → **打开完全信任**。

> 更新规则后，建议下拉刷新页面（或清一次 Safari 缓存）再看效果，避免命中旧缓存。

---

## 四、原理（为什么不封域名）

这个站的广告全部由**同源资源 + 服务端直出空壳**驱动，广告投放域名轮换再勤也没用：

```
/static/pgx/slot-ads.js          ~90KB，广告素材池直接硬编码在里面
      ├─ 职责1：boot()        → 给所有 .pgx-slot 广告位填内容（Banner/弹窗/网格/浮窗）
      ├─ 职责2：hydrateCard() → 给列表里的 .pgx-card 空壳回填落地页 + 封面
      ├─ 素材图：pic.hdhwqx.cn/hc237/...     （与正文图目录零重叠）
      └─ 落地页：64 个随机域名（03901655.com / 914.zfmxjc.cn / 583.0579259.cc ...）

/html/popup-feed.json             底部弹窗配置，由 bottom_popup_poll.js 拉取
      └─ 渲染 <div class="bottom-ads bottom-banner">
```

所以策略是**掐资源 + 收空壳**：

- `slot-ads.js` → `reject-200`（返回空内容）：广告位不被填充、列表广告卡不被激活；
- `popup-feed.json` → `reject-dict`（返回 `{}`）：弹窗调度器判定未启用直接退出；
- HTML 响应注入一段 CSS：把清空后的空广告位、和服务端直出的广告卡空壳一起隐藏。

这几个特征（资源路径 + 容器类名）是站点的固定实现，比封域名稳定得多。

---

## 五、维护与排障

| 情况 | 处理 |
|---|---|
| 站点换了新「线路」域名 | 把新域名追加到 `[mitm] hostname`；若是 `.cc` 随机子域，加根域通配（如 `*.xxxx.cc`）即可 |
| 规则完全没生效 | 多半是证书没信任或「重写」没开；在 QX 的「网络活动 / 日志」里看 `slot-ads.js` 是否被改写 |
| 还有大片空白 | CSS 注入（最后一条 rewrite）没命中——检查 `[mitm]` 里有没有你正在访问的那个域名 |
| 又出现"裂图" | 说明广告壳换了类名，长按裂图 → 检查元素，把新类名补进最后一条 rewrite 的 CSS |
| 广告又回来了 | 站点改了 JS 路径。抓一次新页面看 `<script src=...>`，把新路径替换进 `[rewrite_local]` 第一条 |
| 怀疑误伤了正文图 | 在 QX 日志里搜 `pic.hdhwqx.cn`：如果它被你**其他**广告规则集拦截，在 `[filter_local]` **最顶部**加一行 `host, pic.hdhwqx.cn, direct` 放行即可 |
| 误封了线路 | `③ 广告落地页域名`整段可直接删除，不影响正文浏览 |

---

## 六、边界与提醒

- 广告落地域名会不定期轮换，`③` 那一段属于「过期即失效」的补充保险，核心效果不依赖它。
- 4 个 CloudFront 域名同时出现在广告池和线路列表里，已默认注释，避免误伤备用线路。
- 该站点本身是灰产内容站，页面上还挂着大量第三方追踪、互动聊天 SDK 与外链推广。
  规则只做去广告和追踪阻断，不改变站点内容；访问这类站点请自行注意隐私与风险。
