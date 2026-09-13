# 细分行业报告性能优化说明

## 问题分析

原始实现中，`run_customized_review()` 函数会调用 `get_market_overview()`，该方法会获取：
1. 主要指数行情
2. 涨跌统计
3. 板块涨跌榜
4. 概念涨跌榜
5. 申万三级行业

但我们只需要第5项（申万三级行业），前4项都是不必要的开销。

## 优化措施

### 1. 快速数据获取（已实现）

**修改文件**：`src/core/market_review.py`

优化前：
```python
# 获取所有市场数据
overview = market_analyzer.get_market_overview()
```

优化后：
```python
# 只获取申万三级行业数据（快速模式）
overview = MarketOverview(date=datetime.now().strftime('%Y-%m-%d'))
market_analyzer._get_sw3_sector_rankings(overview)
```

**预期提升**：从 60-120 秒降低到 10-30 秒

### 2. 数据缓存机制（已实现）

**修改文件**：`data_provider/akshare_fetcher.py`

新增缓存：
```python
_sw3_sector_cache = {
    'data': None,
    'timestamp': 0,
    'ttl': 1500  # 25分钟缓存有效期
}
```

**缓存策略**：
- 缓存有效期：25分钟（略短于30分钟的推送间隔）
- 缓存命中：直接返回，耗时 < 0.1 秒
- 缓存未命中：调用API，成功后更新缓存
- API失败：返回过期缓存（如果有）

**预期提升**：
- 缓存命中时：< 1 秒
- 缓存未命中时：10-30 秒
- API失败时：返回旧数据，避免完全失败

### 3. 超时控制（已实现）

**修改文件**：`.github/workflows/01-customized-sector.yml`

```yaml
timeout-minutes: 5  # 从10分钟降低到5分钟
```

## 性能对比

| 场景 | 优化前 | 优化后 |
|------|--------|--------|
| 首次运行 | 60-120 秒 | 10-30 秒 |
| 缓存命中 | 60-120 秒 | < 1 秒 |
| API失败 | 返回失败 | 返回旧缓存 |

## 测试方法

### 本地测试
```bash
# 测试数据获取速度
python test_customized_speed.py
```

### GitHub Actions 测试
1. 手动触发 `customized-only` 模式
2. 查看 Actions 日志中的执行时间
3. 连续触发两次，第二次应该明显更快（缓存命中）

## 进一步优化建议

### 1. 使用更快的数据源
如果东财API较慢，可以考虑：
- 使用本地缓存的行业分类数据
- 使用更快的第三方API

### 2. 预计算
在交易日开盘前预计算行业分类，交易期间只获取涨跌幅数据。

### 3. 并行获取
如果需要多个数据源，可以并行调用。

## 监控指标

在 Actions 日志中关注：
```
[缓存命中] 申万三级行业 - 缓存年龄 XXXs/1500s
[缓存未命中] 触发申万三级行业数据获取
[Akshare] 获取细分行业板块成功: top=XX, bottom=XX
⏱️ 执行耗时: XX 秒
```

## 预期效果

- **目标**：每次执行 < 30 秒
- **理想**：缓存命中时 < 5 秒
- **容错**：API失败时返回旧数据，不中断服务
