---
name: download_x_video
description: 目标是下载X网站的的视频文件，支持单个视频文件下载和批量下载。
version: 3.0.0
---

## 设计原则（来自 6 轮实战、18 次下载的问题复盘）

1. **能不用浏览器就不用浏览器**：纯 curl 方案一次性消灭五大历史问题——执行慢、下载弹窗、页面残留误判、agent 预算耗尽、视口/点击异常。
2. **文件落盘一律 curl**：任何情况下不触发浏览器原生下载通道（物理上杜绝"选择保存位置"弹窗和 net::ERR_ABORTED）。
3. **失败快跳**：单条最多重试 2 次，卡死即跳过继续下一条，绝不死磕；失败时先做一次诊断再决策。
4. **先验证后汇总**：所有文件必须 ffprobe 验证后才能计入成功。

## 降级链

```
A. twitsave GET 接口（纯 curl，首选）
B. x-downloader.com 浏览器解析（仅拿直链）→ curl 下载
C. twitsave.com 网页版浏览器解析 → curl 下载
```

## 步骤1：提取并校验链接

- 提取所有 `https://x.com/{用户名}/status/{数字ID}` 格式的链接，去掉追踪参数（`?s=20`、`?s=46&t=...`）。
- 记录基线：`ls -lt ~/Downloads/*.mp4 | head`。

## 步骤2：路径 A —— twitsave GET 接口（首选，一条脚本完成批量）

对每条链接执行「解析 → 下载 → 验证」循环（写入 `~/Downloads` 需 `dangerouslyDisableSandbox: true`）：

```bash
LINKS="{用户名1}:{statusID1} {用户名2}:{statusID2} ..."   # 空格分隔
for entry in $LINKS; do
  user="${entry%%:*}"; sid="${entry##*:}"
  # 解析：GET 接口返回结果页 HTML，第一个 download 链接是最高分辨率
  link=$(curl -s -L --max-time 30 "https://twitsave.com/info?url=https%3A%2F%2Fx.com%2F${user}%2Fstatus%2F${sid}" \
    | sed 's/&amp;/\&/g' \
    | grep -oE 'href="https://twitsave.com/download\?file=[^"]*"' | head -1 | sed 's/href="//;s/"$//')
  if [ -n "$link" ]; then
    # 下载：静默，无任何弹窗
    if curl -s -L --fail --max-time 180 -o "/Users/yanjun/Downloads/${user}_${sid}.mp4" "$link"; then
      dur=$(ffprobe -v error -show_entries format=duration -of default=noprint_wrappers=1:nokey=1 "/Users/yanjun/Downloads/${user}_${sid}.mp4" 2>/dev/null)
      [ -n "$dur" ] && echo "OK ${user}/${sid} 时长=${dur}s" || echo "FAIL ${user}/${sid} 文件无效"
    else echo "FAIL ${user}/${sid} 下载失败"; fi
  else echo "FAIL ${user}/${sid} 未解析到直链"; fi
done
```

要点：
- `sed 's/&amp;/\&/g'` 必须执行，防止 HTML 实体转义破坏 URL。
- 结果页含多个分辨率（高→低排列），`head -1` 即最高质量。
- `link` 为空 = 视频不存在/私密账号/接口故障；先原样重试 1 次，仍失败则将该条转入路径 B。
- 文件命名 `{作者}_{statusID}.mp4`（此接口不提供原始标题）。

## 步骤3：路径 B —— x-downloader.com 浏览器解析（备用）

browser_use 子代理操作 `https://x-downloader.com/auto`，**只解析直链，不触发文件下载**：
1. 填链接 → 点击 Download Video → `browser_wait_for({ textGone: "Please wait", timeout: 90000 })`（按钮半透明 opacity 0.3 或 href="#" 不算完成；超时跳回首页则重试，最多 2 次）
2. 用 `browser_evaluate` 只读提取直链（不要点击 Download MP4）：
   ```js
   JSON.stringify(Array.from(document.querySelectorAll('a')).map(a=>({t:(a.textContent||'').trim().slice(0,50),h:a.href})).filter(x=>/x-downloader\.com\/v\/|\.mp4/i.test(x.h)).slice(0,5))
   ```
3. **核对文件名归属当前链接**（防页面残留旧结果——曾导致重复下载错文件）
4. 拿到直链后仍用 curl 下载
5. 已知故障模式（2026-10-18 实测）：按钮报"视口外元素"且 DOM `el.click()` 后页面回空首页 = 站点在此环境不可用，**立即转路径 C，不要死磕**
6. 若 DOM 无直链必须点击 Download MP4 时：立即 `browser_network_requests` 抓 `api.x-downloader.com/v/....mp4` 直链；若弹"选择保存位置"框，调用 `browser_waiting_for_user_interaction` 请用户点取消（我们只要直链）

## 步骤4：路径 C —— twitsave 网页版浏览器解析（次备用）

browser_use 子代理操作 `https://twitsave.com`：
1. browser_type 填入链接（若输入框视口异常，用 `browser_evaluate` 设 value + dispatch input 事件）
2. 提交：`browser_evaluate` 执行 `el.click()` 或 `form.submit()`（实体点击可能被视口拦截）
3. `browser_wait_for` 等 15-30 秒结果页出现，`browser_evaluate` 只读提取所有 download 链接，取最高分辨率
4. 每条解析前确认输入框清空、无上一条残留；拿到直链后仍用 curl 下载

## 步骤5：汇总

- **文件保存路径**：告知用户目录（默认 `~/Downloads`），用可点击链接展示
- 表格列出：链接 ↔ 文件名 ↔ 时长（分辨率），统计总数/成功数/失败数
- 失败链接单独列出：失败原因 + 已走的路径 + 已尝试的补救，方便用户后续处理
- 提醒：文件名与页面标题可能不同（中文/表情名终端显示为下划线属正常），以 ffprobe 时长与短码后缀综合核对