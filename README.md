# 🛠 Uptime Kuma 到 UptimeRobot 迁移脚本

## 📌 项目背景

目前本人有一套基于 **Uptime Kuma** 搭建的网站监控系统，监控了 16 个站点，现希望将所有监控迁移到 **UptimeRobot** 平台。

数据来源为 Uptime Kuma 导出的 `JSON` 配置文件，目标是使用 Python 脚本自动将其转为 UptimeRobot API 所支持的格式，并上传。

**如果我的代码对你有帮助，欢迎fork和star!!!**

---

## 📁 需求说明

- ✅ 读取 `Uptime Kuma` 的 JSON 配置文件；
- ✅ 将每个 HTTP 监控转换为 UptimeRobot API 所需字段；
- ✅ 处理重复添加问题（already_exists）；
- ✅ 处理速率限制（429）并重试；
- ✅ 提供跳过、成功、失败总数统计；
- ✅ 自动跳过 inactive（未启用）监控；
- ✅ 兼容收费版中 `interval < 300s` 限制问题（不传 interval）；

---

## ❗遇到的问题和解决方案

| 问题描述 | 错误信息 | 解决方式 |
|----------|----------|-----------|
| Windows 路径引发 `unicodeescape` 错误 | `'unicodeescape' codec can't decode` | 使用 `r"路径"` 格式或将 `\` 替换为 `\\` |
| 添加失败：监控已存在 | `already_exists` | 在添加前通过 `getMonitors` 接口获取已有监控名，提前跳过 |
| 添加失败：付费功能限制 | `"interval" you need to upgrade` | 不传 `interval` 参数（使用默认值 300 秒） |
| 添加失败：限流 | `HTTP 429` | 检测到限流后，根据响应头中的 `Retry-After` 自动等待再重试 |

---
## 📤 环境准备
安装依赖：
```bash
pip install requests
```
获取 API Key：

登录 UptimeRobot 控制台

进入 Profile → My Settings → API Keys → Main API Key

修改路径 & API Key：

替换 KUMA_JSON_FILE 为实际导出的 JSON 路径；

将 "你的UptimeRobot_API_Key" 替换为你的实际 API key。

## ✅ 最终版本 Python 脚本（推荐）

```python
import json
import requests
import time
import os

# 配置
KUMA_JSON_FILE = r"C:\Users\Uptime_Kuma_Backup.json"  # Uptime Kuma 导出 JSON 文件路径
UPTIMEROBOT_API_KEY = os.getenv("UPTIMEROBOT_API_KEY") or "你的UptimeRobot_API_Key"

type_mapping = {
    "http": 1,
    "keyword": 2,
    "ping": 3,
    "port": 4,
}

def load_kuma_config(path):
    with open(path, 'r', encoding='utf-8') as f:
        return json.load(f)

def get_existing_monitors():
    url = "https://api.uptimerobot.com/v2/getMonitors"
    payload = {"api_key": UPTIMEROBOT_API_KEY, "format": "json"}
    try:
        res = requests.post(url, data=payload)
        res.raise_for_status()
        data = res.json()
        if data.get("stat") == "ok":
            return {m["friendly_name"] for m in data.get("monitors", [])}
        else:
            print(f"获取已有监控失败: {data}")
            return set()
    except Exception as e:
        print(f"获取已有监控异常: {e}")
        return set()

def convert_monitor(kuma_monitor):
    if not kuma_monitor.get("active", True):
        return None
    kuma_type = kuma_monitor.get("type", "http").lower()
    uptype = type_mapping.get(kuma_type)
    if not uptype:
        return None
    return {
        "friendly_name": kuma_monitor.get("name", "Unnamed Monitor"),
        "url": kuma_monitor.get("url"),
        "type": uptype,
        # 不传 interval，避免 UptimeRobot 免费账户受限
    }

def create_monitor(monitor):
    if not monitor:
        return {"stat": "skip"}

    payload = {
        "api_key": UPTIMEROBOT_API_KEY,
        **monitor
    }

    while True:
        try:
            response = requests.post("https://api.uptimerobot.com/v2/newMonitor", data=payload)
            if response.status_code == 429:
                retry_after = int(response.headers.get("Retry-After", "30"))
                print(f"限流，等待 {retry_after} 秒后重试...")
                time.sleep(retry_after)
                continue
            if response.status_code != 200:
                return {"stat": "fail", "error": f"HTTP {response.status_code}: {response.text}"}
            try:
                return response.json()
            except json.JSONDecodeError:
                return {"stat": "fail", "error": "Invalid JSON returned", "raw": response.text}
        except Exception as e:
            return {"stat": "fail", "error": str(e)}

def main():
    kuma_data = load_kuma_config(KUMA_JSON_FILE)
    monitors = kuma_data.get("monitorList", [])
    existing_names = get_existing_monitors()

    print(f"共发现 {len(monitors)} 条 Kuma 监控记录")
    print(f"UptimeRobot 已有监控 {len(existing_names)} 条")

    success, failed, skipped = 0, 0, 0
    for m in monitors:
        name = m.get("name", "Unnamed Monitor")
        if name in existing_names:
            print(f"[跳过] {name} 已存在")
            skipped += 1
            continue
        converted = convert_monitor(m)
        result = create_monitor(converted)
        if result.get("stat") == "ok":
            print(f"[成功] {name}")
            success += 1
        else:
            print(f"[失败] {name} - 原因: {result}")
            failed += 1

    print(f"\n完成：成功 {success} 条，失败 {failed} 条，跳过 {skipped} 条")

if __name__ == "__main__":
    main()
```

## 📁 运行脚本
```py
共发现 16 条 Kuma 监控记录
UptimeRobot 已有监控 9 条
[跳过] Blog 已存在
[跳过] Address 已存在
[跳过] Alist 已存在
[跳过] Nav 已存在
[跳过] phpAdmin 已存在
[跳过] FileManger 已存在
[跳过] Address 已存在
[跳过] img-bed 已存在
[跳过] short-links 已存在
[跳过] short-links 已存在
[成功] img-bed-Telegraph
[成功] chatapi
[失败] Uptime Kuma - 原因: {'stat': 'fail', 'error': {'type': 'already_exists', 'message': 'monitor already exists.'}}
[成功] sink
[成功] TV
[成功] Github

完成：成功 5 条，失败 1 条，跳过 10 条
```
