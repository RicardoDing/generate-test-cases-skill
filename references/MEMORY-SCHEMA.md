# 记忆数据结构

定义 `.memory` 目录中各文件的数据结构，支撑跨会话自学习。

## .memory 目录结构

```
.memory/
├── project-context.json      # 项目上下文信息
├── terminology.json          # 领域术语库（自动学习 + 手动添加）
├── generation-history.json   # 生成历史（质量趋势分析）
├── user-preferences.json     # 用户偏好（自动记忆）
└── ambiguity-decisions.json  # 歧义决策记录（避免重复询问）
```

## 各文件 Schema

### project-context.json

项目基本信息，首次 `init` 时创建。

```json
{
  "project_name": "string",
  "initialized_at": "ISO datetime",
  "requirements_dir": "string (用户指定的需求文档目录，如 requirements/、docs/req/ 等)",
  "output_dir": "string (用户指定的输出目录，如 test-docs/、output/ 等)"
}
```

### terminology.json

领域术语和模块缩写，解析需求时自动提取 + 用户手动补充。

```json
{
  "domain_terms": {
    "GMV": "商品交易总额",
    "SKU": "库存单位"
  },
  "module_abbreviations": {
    "登录": "LOGIN",
    "订单": "ORDER"
  }
}
```

**自学习规则**：
- 解析需求时自动识别 `术语（解释）`、`缩写：全称` 等模式
- 用户修正术语时询问是否记住
- 跨会话复用，减少重复提取

### generation-history.json

生成历史，用于质量趋势分析和生成策略优化。

```json
{
  "generations": [
    {
      "timestamp": "YYYYmmddHHMMSS (14位时间戳，如 20260424153716)",
      "source_files": ["string (需求源文件路径数组)"],
      "output": "string (输出的 XMind 文件路径)",
      "case_count": 42,
      "coverage_rate": "100%",
      "priority_distribution": { "P0": 5, "P1": 16, "P2": 17, "P3": 4 },
      "tags": ["string (标签数组，如 '后端接口'、'C端'、'B端')"],
      "design_methods": {
        "ST": 10, "PV": 8, "BVA": 5, "EP": 3, "EG": 2,
        "RV": 4, "BS": 3, "RL": 2, "DB": 1, "REDIS": 0
      },
      "modules_covered": ["string (覆盖的功能模块名称数组)"]
    }
  ]
}
```

**字段说明**：

| 字段 | 必填 | 说明 |
|------|------|------|
| `timestamp` | ✅ | 14 位时间戳，与输出文件名中的时间戳一致 |
| `source_files` | ✅ | 本次生成使用的需求源文件路径列表 |
| `output` | ✅ | 生成的 XMind 文件完整路径 |
| `case_count` | ✅ | 生成的用例总数 |
| `coverage_rate` | ✅ | 需求覆盖率 |
| `priority_distribution` | ✅ | 各优先级用例数量 |
| `tags` | ✅ | 标签数组，标识测试类型（如"后端接口"、"C端"） |
| `design_methods` | ✅ | 各设计方法的用例数量分布，用于方法占比趋势分析 |
| `modules_covered` | ✅ | 本次覆盖的功能模块列表，用于模块覆盖趋势分析 |

**自学习规则**：
- 每次生成后自动写入，积累质量基线
- 下次生成同类需求时参考历史分布（如 P0 占比偏高则自动提示）
- 统计常见模块，优化场景覆盖模式
- `design_methods` 分布用于 Phase 2.8 方法占比异常检测（参见 TEST-DESIGN-METHODS.md 中的阈值表）
- 当记录数超过 500 条时，在 `read` 操作时提示用户考虑清理历史记录（可通过 `clear` 重置）

### user-preferences.json

用户交互偏好，自动记忆操作习惯。

```json
{
  "default_output_dir": "./test-docs",
  "show_samples_in_preview": true,
  "auto_confirm_parsing": false,
  "ambiguity_handling": "ask | skip | mark",
  "priority_distribution": {
    "p0_min": 10,
    "p0_max": 15,
    "warn_on_imbalance": true
  },
  "default_tag": "C端",
  "step_granularity": "fine | normal | coarse",
  "title_style": null,
  "updated_at": "ISO datetime"
}
```

**字段说明**：
| 字段 | 说明 |
|------|------|
| `default_output_dir` | XMind 输出目录 |
| `show_samples_in_preview` | 预览时展示样例 |
| `auto_confirm_parsing` | 自动确认解析结果 |
| `ambiguity_handling` | 歧义处理策略：`ask` 询问 / `skip` 跳过 / `mark` 标记 |
| `priority_distribution` | P0 占比偏好 + 失衡警告开关 |
| `default_tag` | 上次选择的标签值，下次可作为默认建议 |
| `step_granularity` | 用例步骤粒度：`fine` 细化 / `normal` 默认 / `coarse` 粗略 |
| `title_style` | 用户偏好的用例标题风格（如“输入X，结果Y”），null 表示未设置 |

### ambiguity-decisions.json

歧义决策记录，避免相同歧义反复询问用户。

```json
{
  "decisions": [
    {
      "date": "2026-02-25",
      "type": "BOUNDARY_UNCLEAR",
      "context": "密码复杂度",
      "original_text": "密码应有足够的复杂度",
      "user_decision": "长度8-20位，必须包含大小写字母和数字",
      "auto_accepted": false,
      "applied_to": ["REQ_001"]
    }
  ]
}
```

**字段说明**：

| 字段 | 必填 | 说明 |
|------|------|------|
| `date` | ✅ | 决策日期 |
| `type` | ✅ | 歧义类型代码（BOUNDARY_UNCLEAR / RULE_CONFLICT / MISSING_ERROR / VAGUE_CRITERIA / INCOMPLETE_FLOW） |
| `context` | ✅ | 歧义上下文关键词 |
| `original_text` | ✅ | 需求原文 |
| `user_decision` | ✅ | 用户决策或 AI 自动采用的理解 |
| `auto_accepted` | ✅ | `true` = AI 自动采用（未经用户逐条确认），`false` = 用户明确确认。Phase 2.8 质量报告中汇总 `auto_accepted: true` 的数量，提示用户回溯 |
| `applied_to` | ✅ | 关联的需求 ID 列表 |

**自学习规则**：
- 每次歧义决策自动存入
- 后续遇到相似歧义时自动复用历史决策
- `auto_accepted: true` 的决策在后续复用时仍标记为自动采用，直到用户明确修正

---

## 记忆更新规则

| 时机 | 更新操作 | 脚本调用 |
|------|---------|---------|
| 首次使用 | 创建所有文件 | `memory_manager.py --action init` |
| 解析需求 | 提取新术语 | `memory_manager.py --action update --type terminology` |
| 处理歧义 | 记录决策 | `memory_manager.py --action add-ambiguity` |
| 生成完成 | 写入历史 | `memory_manager.py --action add-record --data '{...}'` |
| 用户反馈 | 更新偏好 | `memory_manager.py --action set-pref` |
| 用户要求 | 清除所有 | `memory_manager.py --action clear` |

## 使用示例

```bash
# 初始化记忆
python3 memory_manager.py --action init --project "${SKILL_ROOT}"

# 读取记忆
python3 memory_manager.py --action read --project "${SKILL_ROOT}"

# 更新术语
python3 memory_manager.py --action update --project "${SKILL_ROOT}" --type terminology \
  --data '{"domain_terms": {"SKU": "库存单位"}}'

# 添加生成记录
python3 memory_manager.py --action add-record --project "${SKILL_ROOT}" \
  --data '{"type":"test_case","source":"requirements/用户登录模块_PRD.md","output":"test-docs/testcases_20260225143022.xmind","case_count":42,"coverage_rate":"100%"}'

# 记录歧义决策
python3 memory_manager.py --action add-ambiguity --project "${SKILL_ROOT}" \
  --keyword "密码复杂度" --data '{"type":"BOUNDARY_UNCLEAR","user_decision":"8-20位含大小写字母和数字"}'

# 查找历史歧义
python3 memory_manager.py --action find-ambiguity --project "${SKILL_ROOT}" --keyword "密码"

# 设置用户偏好
python3 memory_manager.py --action set-pref --project "${SKILL_ROOT}" \
  --data '{"default_tag": "C端"}'

# 清除记忆
python3 memory_manager.py --action clear --project "${SKILL_ROOT}"
```
