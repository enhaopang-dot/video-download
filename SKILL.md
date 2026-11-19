---
name: video-download
description: 各平台视频 / 直播回放 / 图片的下载统一入口。当用户说「把这个视频下载下来」「下载这个链接的内容」「去水印」「解析这个链接」「视频号回放怎么下」「抖音视频保存」或贴出抖音、快手、小红书、B站、微博、视频号等平台链接并要求保存时使用。含已验证的下载链路、反爬拦截判据、下载后的转正与压缩实测参数。关键词：下载视频、去水印、解析链接、视频号、直播回放、抖音提取、m3u8、ffmpeg 压缩。
---

# 各平台视频 / 直播回放下载

> 来源：2026-11-15 微信视频号直播回放实测（`2026 AI-FDE 城市共建计划·杭州站`，3 小时 23 分 / 3.79 GiB），全链路跑通。
> 深度资料：抖音 CDP 抓流链路见 memory `douyin-cdp-extraction-chain.md`；视频号助手后台拉流见 skill `channels-live-analysis`。

## 零、先判平台，再选链路

不要一上来就试工具。先看链接域名，按下面路由走：

| 平台 | 链接特征 | 走哪条 |
| --- | --- | --- |
| 微信视频号 | `weixin.qq.com/sph/xxx`、`channels.weixin.qq.com` | **第三节：第三方解析站（人工）** |
| 抖音 | `v.douyin.com`、`www.douyin.com/video/{id}` | memory `douyin-cdp-extraction-chain.md` 的 CDP 抓流 |
| 小红书 / 快手 / B站 / 微博 / YouTube / TikTok 等 | 对应域名 | 第三节：第三方解析站，通用 |
| 自己运营的视频号账号 | — | skill `channels-live-analysis`，走 `timeshift.m3u8` 原画 |

**铁律**：能人工就人工。第三方解析站普遍有反自动化检测，AI 硬闯会把 IP 打进冷却期，反而让用户自己都下不了。

---

## 一、结论：视频号内容只有两条路能走通

2026-11-15 实测，四条路径的结果：

| 路径 | 结果 | 原因 |
| --- | --- | --- |
| 无头浏览器直开 `sph` 短链 | ❌ | 落到 `channels.weixin.qq.com/finder-preview/pages/sph?id=xxx`，正文要求微信内播放 |
| 拦网络请求 + 扫 HTML 找流地址 | ❌ | 只暴露 `finder.video.qq.com/251/20304/stodownload?...&picformat=200`，那是**封面图** |
| 视频号助手后台拉 `timeshift.m3u8` | ⚠️ 仅运营者 | 需目标账号运营者权限，非运营者拿不到 |
| **第三方解析站人工粘贴** | ✅ **可用** | 见第三节 |

**能走通的两条**：① 第三方解析站人工操作；② 自己运营的账号走视频号助手。

---

## 二、视频号为什么难下（技术原因，别浪费时间重试）

- 短链 `https://weixin.qq.com/sph/{id}` 会 302 到 `finder-preview` 页，页面能读到标题和发布日期，但没有播放器。
- 页面里唯一的媒体地址带 `picformat=200` / `wxampicformat=503` 参数 —— 这是**封面图规格**，不是视频流。
- 真正的流在 `timeshift.m3u8`（视频号回放，HEVC 1080×1920 竖屏，实测约 9.6 Mbps ≈ 4.3 GB/小时），只对账号运营者开放。

---

## 三、第三方解析站（已验证可用）

### 3.1 已验证站点

**dy.kukutool.com** —— 2026-11-15 实测成功下载视频号直播回放。

```
1. 浏览器打开 https://dy.kukutool.com/
2. 把视频号链接粘进输入框（placeholder 是「粘贴带链接的文本」）
3. 点「开始解析」
4. 出结果后点下载，得到 stodownload.mp4
```

产出的文件名固定是 `stodownload.mp4`，可以在浏览器下载目录或桌面找到。

### 3.2 反爬拦截：必须人工，别用脚本

自动访问会被拦在 `/api/auth-970b03`：

```
429  ticket_ip_restricted|auto_script|<你的公网IP>
当前请求暂时无法处理，请稍后再试。通常 20 分钟后自动修复。
```

**拦截是按 IP + 自动化特征判定的，冷却约 20 分钟。** 实测加反检测补丁（抹 `navigator.webdriver`、伪造 plugins/languages、有头浏览器、逐字符输入）**仍然被拦**。

所以：

- ❌ 不要写脚本去灌这个站
- ❌ 不要在冷却期内反复重试（会让用户自己也被挡）
- ✅ **让用户在自己浏览器里手动操作一次**，这是最快也最稳的

### 3.3 站点能力边界（别照官方文案信）

它的平台清单 `/api/maintenance-mode` 只有 16 个分组：`douyin / kuaishou / xiaohongshu / bilibili / weibo / xigua / haokan / zhihu / tieba / tiktok / instagram / facebook / youtube / x / threads / other`，**列表里没有 weixin / channels / sph**，但实测视频号能下。

**教训：平台枚举不等于能力边界。** 判断某个站能不能下，以实际跑通为准，不要从它的清单反推。

### 3.4 同类站点

同类型站点很多（130+ 平台去水印站），一个不行换一个。判断标准只有一条：**人工粘链接能出下载按钮**。

---

## 四、下载后必做三件事

### 4.1 ffprobe 核实参数

```bash
ffprobe -v error -show_entries format=duration,size,bit_rate \
  -show_entries stream=codec_name,width,height,avg_frame_rate,bit_rate \
  -of default=nw=1 <文件>
```

要确认三件事：**时长对不对**（是不是完整一场）、**分辨率与帧率**（有没有被降质）、**有没有音轨**（抖音 Web 常把音视频拆成两条流，只下视频会哑片）。

### 4.2 检查画面有没有被旋转

第三方站转码时常出这个问题：像素本身转了但**没有写 rotation 元数据**，导致播放器不知道该转回来。

```bash
ffprobe -v error -select_streams v:0 -show_entries stream_side_data=rotation <文件>
```

输出为空 = 没有旋转元数据。此时用**蓝屏质心法**判定正确朝向（适用于会场/课件类素材）：

```python
from PIL import Image
im = Image.open('frame.png').convert('RGB')
for rot in (None, Image.ROTATE_90, Image.ROTATE_180, Image.ROTATE_270):
    t = im if rot is None else im.transpose(rot)
    s = t.resize((96, int(96*t.size[1]/t.size[0]))); px = s.load()
    w, h = s.size; xs = []; ys = []
    for y in range(h):
        for x in range(w):
            r, g, b = px[x, y]
            if b > 90 and b > r+35 and b >= g+10:   # LED 屏的饱和蓝
                xs.append(x); ys.append(y)
    if xs:
        print(rot, 'x=%.2f y=%.2f' % (sum(xs)/len(xs)/w, sum(ys)/len(ys)/h))
```

**怎么判**：会场实景里，大屏应当**水平居中、位于画面上三分之一**。选质心最接近 `x≈0.51 / y≈0.35` 的那个旋转。实测该帧 `transpose=2`（逆时针 90°）命中。

转正常用量：`-vf transpose=2`（逆时针）/ `transpose=1`（顺时针）。

### 4.3 无损转正走不通

`-c copy -metadata:s:v:0 rotate=-90` 重新封装 **不生效**，新版本 ffmpeg 已不为 mp4 写入 rotate 标签，输出文件仍无旋转 side data。**要转正只能重编码。**

---

## 五、压缩参数（2026-11-15 实测数据）

前提认知：**已是 HEVC 的源，做"无损压缩"体积只会变大**。所谓"不降清晰度"实际是"视觉无感"的有损重编码。

### 5.1 命名与画质实测（912×1920 25fps 2.54 Mbps 源，5 分钟双样本）

| 参数 | 5 分钟体积 | SSIM | PSNR | 相对源 |
| --- | --- | --- | --- | --- |
| 源 | 95 MB | 1.000 | ∞ | — |
| libx265 `-crf 23 -preset fast` | 25–31 MB | 0.988–0.990 | 38.3 dB | **−70%** |
| libx265 `-crf 26 -preset fast` | 16–20 MB | 0.988 | 38.1 dB | **−80%** |
| hevc_nvenc `-cq 24 -preset p6` | 80 MB | 0.991 | 38.4 dB | −21% |
| hevc_nvenc `-cq 28 -preset p6` | 41 MB | 0.989 | 38.3 dB | −57% |

**结论**：x265 CRF23/26 的 SSIM 只差 0.0016，会议类素材肉眼分不出来；NVENC 快但同等画质下体积大 2 倍。

### 5.2 推荐命令（转正 + 压缩一步完成）

```bash
ffmpeg -i in.mp4 \
  -vf transpose=2 \
  -c:v libx265 -preset fast -crf 23 \
  -tag:v hvc1 \
  -c:a copy \
  -movflags +faststart \
  out.mp4
```

- `-tag:v hvc1`：不加的话 Apple 系播放器可能不认。
- `-movflags +faststart`：moov 前置，便于边下边看与快速 seek。
- `-c:a copy`：音频不重编码，省时间且无损。
- **耗时参考**：3 小时 23 分、912×1920、`preset fast` 约 60–90 分钟。赶时间换 `hevc_nvenc -cq 24`，约 8 分钟但体积只降两成。

---

## 六、坑位清单

- **反爬是 IP 级冷却**，自动重试会拖累用户自己的手动操作。
- **平台清单不代表能力**，以实跑为准。
- **音视频分离**（抖音 Web 尤其）：CDN 直链里画面和声音是两条，只下一条会得到哑片或黑屏。
- **`picformat=200` 一律是图不是视频**，看到这个参数就别再解析了。
- **无损旋转写不进 mp4**，别在 `-c copy` 上浪费时间。
- **`r_frame_rate=90000/1` 是假象**，实际帧率看 `avg_frame_rate` 和 `nb_frames / duration`。
- **下载站逐帧转正有方向概率错**，务必用 4.2 的方法验证后再批量处理。

---

## 七、完整实战记录

2026-11-15 视频号回放下载全过程的证据、参数、尝试记录：
`<你的项目路径> AI培训与比赛/20261115-AI-FDE城市共建计划-杭州站/04-视频回放/回放链接与下载状态.md`
