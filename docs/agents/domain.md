# 领域文档

工程技能在探索代码库时，应按本文件说明使用仓库的领域文档。

## 探索前先读取

- 根目录下的 **`CONTEXT.md`**；或者
- 如果根目录存在 **`CONTEXT-MAP.md`**，按其指向读取与当前主题相关的各上下文 `CONTEXT.md`
- **`docs/adr/`** 中涉及当前工作区域的 ADR；在多上下文仓库中，还要检查 `src/<context>/docs/adr/` 下的上下文级决策

如果这些文件不存在，静默继续。其缺失无需报告，也无需预先建议创建。`/domain-modeling` 技能（可通过 `/grill-with-docs` 和 `/improve-codebase-architecture` 使用）会在术语或决策得到确认时按需创建它们。

## ADR 命名约定

- 文件名采用 `YYYY-MM-DD-<名称>.md` 格式，例如 `2026-03-20-event-sourced-orders.md`
- 日期使用 ADR 首次创建日期，并按 ISO 8601 的 `YYYY-MM-DD` 格式补齐位数
- 名称应简短、稳定且能概括决策；英文单词使用小写 kebab-case

## 文件结构

单上下文仓库（适用于大多数仓库）：

```
/
├── CONTEXT.md
├── docs/adr/
│   ├── 2026-03-20-event-sourced-orders.md
│   └── 2026-03-21-postgres-for-write-model.md
└── src/
```

多上下文仓库（根目录存在 `CONTEXT-MAP.md`）：

```
/
├── CONTEXT-MAP.md
├── docs/adr/                          ← 系统级决策
└── src/
    ├── ordering/
    │   ├── CONTEXT.md
    │   └── docs/adr/                  ← 上下文级决策
    └── billing/
        ├── CONTEXT.md
        └── docs/adr/
```

## 使用术语表中的词汇

当输出内容命名领域概念时（例如问题标题、重构建议、假设或测试名称），使用 `CONTEXT.md` 中定义的术语，保持与术语表一致。

如果术语表尚未包含所需概念，应重新检查该表述是否符合项目语言；若确有缺口，则记录下来供 `/domain-modeling` 处理。

## 明确指出 ADR 冲突

如果输出内容与现有 ADR 冲突，应明确说明，而不是静默覆盖：

> _这与 ADR `2026-03-20-event-sourced-orders.md` 冲突，但值得重新讨论，因为……_
