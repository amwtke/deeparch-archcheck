# archcheck

> 软件架构体检 skill。基于 Robert C. Martin《架构整洁之道》和 Alistair Cockburn 端口适配器架构。

## 是什么

一个 Claude Code skill,接收"项目代码"作为输入,输出一份**双格式架构体检报告**(Markdown + HTML)。

## 目录结构

```
archcheck/
├── SKILL.md                                   主入口,执行流程
├── README.md                                  本文件
├── references/
│   ├── dependency-direction.md                维度 1 规则库
│   ├── boundary-integrity.md                  维度 2 规则库
│   ├── entity-vs-usecase.md                   维度 3 规则库
│   ├── testability.md                         维度 4 规则库
│   ├── code-smells.md                         维度 5 规则库
│   ├── scoring-rubric.md                      综合评分细则
│   └── language-patterns/
│       ├── java-spring.md                     Java/Spring (主深)
│       ├── go.md                              Go 扩展点
│       ├── python.md                          Python 扩展点
│       └── typescript.md                      TypeScript 扩展点
└── assets/
    └── report-template.html                   HTML 报告模板(深色科技风)
```

## 安装

```bash
# 复制到你的 Claude Code skills 目录
cp -r archcheck ~/.claude/skills/

# 或者
mv archcheck ~/.claude/skills/
```

## 使用

```
/archcheck <项目路径>
```

或者直接传项目压缩包/目录给 Claude,说 "帮我做架构体检"。

## 五大评估维度

| 维度 | 关注点 | 权重 |
|------|--------|------|
| 1. 依赖方向 | 业务核心是否被框架污染 | 30% |
| 2. 边界完整性 | Web/持久化/外部系统/模块边界 | 25% |
| 3. Entity vs UseCase | 充血还是贫血,业务规则归属 | 15% |
| 4. 可测试性 | 业务核心能否独立测试 | 20% |
| 5. 代码坏味道 | SOLID 违反、巨型类、命名失真 | 10% |

## 评分体系

A (整洁标杆) → B (接近整洁) → C (传统三层) → D (有问题) → F (腐烂)

支持根据**项目阶段、团队规模、业务复杂度**做评分修正。

## 输出

- `archcheck-report.md` - 纯文本报告,适合 PR 评论和版本控制
- `archcheck-report.html` - 深色科技风 HTML 报告,含 SVG 雷达图

## 设计哲学

- **代码引用必须有行号** - 不允许"凭感觉"扣分
- **不只批评,给改造代码** - 每个扣分项给出"改之前 → 改之后"
- **支持工程妥协** - `@Transactional` 等合理妥协会标注而非扣分
- **三层剖析** - 现象层 / 机制层 / 原理层(继承自 deeparch 风格)

## 版本

v1.0 - 初版
