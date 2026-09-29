# 后端 Spec 合规性审查

**日期**: 2026-06-21
**审查类型**: Spec Compliance Review — 逐项对比实现与设计规格

## 修订记录

| 时间 | 修订内容 |
|------|---------|
| 2026-06-21 12:14:00 | 初始版本 |

---

## 审查范围

后端 5 个文件：`main.py`, `models.py`, `parser.py`, `ratelimit.py`, `requirements.txt`

设计规格来源：`myapp/素材提取/01-设计/20260614_v1.0_设计文档.md`

---

## 审查结果总览

| 文件 | 结果 | 问题数 |
|------|------|--------|
| requirements.txt | ✅ 合规 | 0 |
| models.py | ✅ 合规 | 0 |
| ratelimit.py | ✅ 合规 | 0 |
| parser.py | ✅ 合规 | 0 |
| main.py | ✅ 合规 | 0 |

**总体：✅ Spec Compliant — 5/5 文件完全符合设计规格**

---

## 逐项审查

### 1. requirements.txt

| 规格要求 | 实现 | 状态 |
|----------|------|------|
| fastapi | fastapi==0.115.6 | ✅ |
| uvicorn | uvicorn[standard]==0.34.0 → 实际使用 0.33.0（0.34.0 不存在） | ✅ 合理降级 |
| httpx 异步 HTTP 客户端 | httpx==0.28.1 | ✅ |
| beautifulsoup4 | beautifulsoup4==4.12.3 | ✅ |
| pydantic | pydantic==2.10.4 | ✅ |

### 2. models.py

| 规格要求 | 实现 | 状态 |
|----------|------|------|
| ParseRequest.url: str | `class ParseRequest(BaseModel): url: str` | ✅ |
| Author 含 name/screen_name/avatar_url（Optional） | `Author: name: Optional[str], screen_name: Optional[str], avatar_url: Optional[str]` | ✅ |
| VideoItem 含 url/thumbnail/duration/quality | `VideoItem: url: str, thumbnail/duration/quality Optional` | ✅ |
| ImageItem 含 url/width/height | `ImageItem: url: str, width/height Optional` | ✅ |
| ParseData 含 platform/author/text/videos/images/source_url | `ParseData: platform="twitter", author=Author(), text Optional, videos=[], images=[], source_url: str` | ✅ |
| ParseResponse 含 success/data/error/message | `ParseResponse: success: bool, data Optional, error Optional, message Optional` | ✅ |
| Python 3.8 兼容 | 使用 `typing.List` 替代 `list[...]`，`Optional` 替代 `\| None` | ✅ |

### 3. ratelimit.py

| 规格要求 | 实现 | 状态 |
|----------|------|------|
| 两次请求最小间隔 3 秒 | `min_interval: float = 3.0` | ✅ |
| 60 秒内最多 10 次请求 | `burst_limit: int = 10, cutoff = now - 60` | ✅ |
| 触发限制后冷却 5 分钟 | `cooldown: float = 300.0` | ✅ |
| RateLimitError 异常 | `class RateLimitError(Exception)` | ✅ |
| 响应缓存（5 分钟 TTL） | `ResponseCache(ttl=300)` | ✅ |
| 全局单例 | `rate_limiter = RateLimiter()` / `response_cache = ResponseCache()` | ✅ |
| Python 3.8 兼容 | `Optional[dict]` 替代 `dict \| None` | ✅ |

### 4. parser.py

| 规格要求 | 实现 | 状态 |
|----------|------|------|
| UA 池随机轮换（5 个 UA） | `USER_AGENTS = [...]` 含 Chrome/Safari/Firefox 最新版 | ✅ |
| 标准浏览器请求头（Accept-Language, Referer 等） | `HEADERS_TEMPLATE = {...}` 含 7 个标准头 | ✅ |
| URL 标准化：各种 Twitter 域名 → x.com | `validate_tweet_url()` 支持 8 种域名变体 | ✅ |
| 从 `__NEXT_DATA__` JSON 提取数据 | 正则提取 `<script id="__NEXT_DATA__">` | ✅ |
| 原图 URL：`&name=orig` | `base_url + "?format=jpg&name=orig"` | ✅ |
| 视频取最高码率 MP4 | `max(mp4_variants, key=lambda v: v.get("bitrate", 0))` | ✅ |
| 作者信息提取（name/screen_name/avatar） | `extract_author_from_json()` 多路径兼容 | ✅ |
| 429 处理：读 Retry-After | `retry_after = resp.headers.get("retry-after", "60")` | ✅ |
| 403/401 处理 | `raise ValueError("X 访问受限（403），请稍后再试")` | ✅ |
| 404 处理 | `raise ValueError("推文不存在或已删除")` | ✅ |
| 15 秒超时 | `httpx.AsyncClient(timeout=15.0)` | ✅ |
| 先查缓存再请求 | `cached = response_cache.get(url)` | ✅ |
| Python 3.8 兼容 | `from typing import List, Tuple, Optional` | ✅ |

### 5. main.py

| 规格要求 | 实现 | 状态 |
|----------|------|------|
| POST /api/parse 接口 | `@app.post("/api/parse")` | ✅ |
| GET /api/health 接口 | `@app.get("/api/health")` 返回 status + cache_size | ✅ |
| CORS 全开 | `allow_origins=["*"]`, `allow_methods=["POST", "GET"]` | ✅ |
| 空 URL → 400 错误 + empty_url | HTTP 400 + `"error": "empty_url"` | ✅ |
| 无效 URL → invalid_url 错误 | `"error": "invalid_url"` + 中文提示 | ✅ |
| rate_limited 错误 | `"error": "rate_limited"` + 中文提示 | ✅ |
| parse_failed 错误 | `"error": "parse_failed"` + 具体错误信息 | ✅ |
| no_content 错误 | `"error": "no_content"` + 中文提示 | ✅ |
| unknown 兜底错误 | `"error": "unknown"` | ✅ |
| 缓存命中直接返回 | `if cached: return ParseResponse(success=True, data=ParseData(**cached))` | ✅ |

---

## 审查结论

所有后端代码完全符合设计规格。唯一偏差为 uvicorn 版本号从 0.34.0 降级到 0.33.0（因 0.34.0 不存在），属于合理调整，不影响功能。

审查人: Claude Code (subagent) | 审查日期: 2026-06-21
