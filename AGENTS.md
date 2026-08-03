# PROJECT KNOWLEDGE BASE

**Generated:** 2026-08-03
**Stack:** Python 3.10 + requests + GitHub Actions

## OVERVIEW
夸克网盘自动签到脚本。GitHub Actions 定时执行，调用夸克 API 签到 + 领取奖励，支持多账号 + WxPusher 微信通知。

## STRUCTURE
```
./
├── checkIn_Quark.py     # 主签到脚本（入口）
├── wxpusher.py          # WxPusher 推送通知模块
├── requirements.txt     # 仅 requests 依赖
├── .github/workflows/   # CI：quark_signin.yml
└── README.md            # 用户使用文档
```

## WHERE TO LOOK
| 任务 | 位置 | 备注 |
|------|------|------|
| 签到逻辑 | `checkIn_Quark.py` → `Quark.do_sign()` | 查信息 → 签到 → 返结果 |
| API 调用 | `checkIn_Quark.py` → `get_growth_info()` / `get_growth_sign()` | 夸克网盘 API |
| Cookie 解析 | `checkIn_Quark.py` → `get_env()` | `\n` 或 `&&` 分割 |
| 通知推送 | `wxpusher.py` → `wxpusher()` | WxPusher HTTP API |
| CI 定时 | `.github/workflows/quark_signin.yml` | cron: 8:01 & 19:01(UTC+8) |
| 保持仓库活跃 | `.github/workflows/quark_signin.yml` → 空提交 step | 防 Actions 停用 |

## CODE MAP
| Symbol | Type | Location | Role |
|--------|------|----------|------|
| `Quark` | class | checkIn_Quark.py:33 | 签到核心封装 |
| `Quark.do_sign()` | method | checkIn_Quark.py:114 | 执行签到流程 |
| `Quark.get_growth_info()` | method | checkIn_Quark.py:57 | 获取签到状态 |
| `Quark.get_growth_sign()` | method | checkIn_Quark.py:77 | 执行签到请求 |
| `get_env()` | function | checkIn_Quark.py:17 | 解析 COOKIE_QUARK 环境变量 |
| `main()` | function | checkIn_Quark.py:152 | 入口：遍历账号签到 |
| `wxpusher()` | function | wxpusher.py:5 | 发送 WxPusher 通知 |

## CONVENTIONS
- 环境变量驱动配置（无配置文件）
- Cookie 格式：`user=张三; kps=xxx; sign=xxx; vcode=xxx;`
- 多账号用 `\n` 或 `&&` 分隔
- 签到结果用 emoji 日志（✅/❌）
- CI 中随机延迟模拟人工（0-60s morning, 0-30s afternoon）

## ANTI-PATTERNS (THIS PROJECT)
- `global cookie_quark` 无必要使用（仅 main 内赋值）
- `except Exception as err:` 吞没异常无 traceback（line 209）
- `print()` 调试残留注释（多处 `#print(response)`）
- `wxpusher.py` 有独立 `main()` 但从 `checkIn_Quark.py` 导入调用 → 混合模式
- line 7 死代码：模块级 `cookie_list` 赋值被 `get_env()` 局部变量覆盖，`.split('\n|&&')` 与 line 21 `re.split` 语义不同
- `wxpusher.py:21` 用 HTTP 明文调用 WxPusher API（非 HTTPS），消息明文传输
- `quark_signin.yml` if 条件 `github.event.schedule == '0 1 * * *'` 与 cron 定义 `1 0 * * *` 不匹配 → 随机延迟永不触发

## COMMANDS
```bash
# 本地运行
pip install requests
export COOKIE_QUARK="user=xxx; kps=xxx; sign=xxx; vcode=xxx;"
python checkIn_Quark.py

# 可选通知
export WXPUSHER_APP_TOKEN="xxx"
export WXPUSHER_UID="xxx"
```

## NOTES
- Cookie 中 `kps/sign/vcode` 有效期约 2 个月，过期需重新抓包
- CI cron 为 UTC 时间，+8 = 北京时间
- `get_growth_info()` 失败时抛异常（单账号设计），多账号场景需注意
- 无测试覆盖
