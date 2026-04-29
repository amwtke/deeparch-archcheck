# deeparch-archcheck

> 一个 Claude Code skill,给软件项目做**架构层面体检**,输出 Markdown + HTML 双格式报告。

理论根基:Robert C. Martin《架构整洁之道》、Alistair Cockburn 端口适配器架构、Eric Evans DDD。

主深 **Java/Spring**,同时支持 **Go / Python / TypeScript**。

---

## 安装

把 `archcheck/` 目录放到 Claude Code 的 skills 目录下:

```bash
git clone git@github.com:amwtke/deeparch-archcheck.git
cp -r deeparch-archcheck/archcheck ~/.claude/skills/
```

或者直接软链接(本地修改即时生效):

```bash
ln -s "$(pwd)/deeparch-archcheck/archcheck" ~/.claude/skills/archcheck
```

安装后在 Claude Code 里输入 `/` 应该能看到 `archcheck` 出现在 skill 列表里。

---

## 使用

### 方式一:斜杠命令

```
/archcheck <项目路径>
```

例如:

```
/archcheck ~/code/my-spring-app
/archcheck ./backend
```

### 方式二:自然语言

把项目目录或压缩包给 Claude,然后说:

- "帮我做架构体检"
- "分析一下这个项目的架构"
- "看看这个项目符不符合整洁架构"
- "评估一下依赖方向有没有问题"

skill 都会被触发。

---

## 输出

执行结束后,会在当前目录生成两份报告:

| 文件 | 用途 |
|------|------|
| `archcheck-report.md` | 纯文本,适合放进 PR 评论或版本控制 |
| `archcheck-report.html` | 深色科技风可视化报告(SVG 雷达图、可折叠扣分项),双击直接看 |

**HTML 报告会被排在前面,可以直接在浏览器里打开。**

---

## 五大体检维度

| 维度 | 关注点 | 权重 |
|------|--------|------|
| 1. 依赖方向 | 业务核心是否被框架污染、接口归属、能否独立编译 | 30% |
| 2. 边界完整性 | Web / 持久化 / 外部系统 / 模块边界 | 25% |
| 3. Entity vs UseCase | 充血还是贫血、业务规则归属、UseCase 是否纯编排 | 15% |
| 4. 可测试性 | 业务核心能否脱离框架测试、mock 密度 | 20% |
| 5. 代码坏味道 | SOLID 违反、巨型类、命名失真 | 10% |

---

## 评分体系

| 等级 | 含义 |
|------|------|
| **A** | 整洁架构标杆,业务核心可独立编译/测试,罕见 |
| **B** | 接近整洁,架构师有意识但局部漏洞,优秀团队水平 |
| **C** | 传统三层 Spring 应用,能用但变化时成本高 |
| **D** | 改一处崩三处,新人上手慢 |
| **F** | 已经成为业务发展瓶颈,需紧急止血 |

支持根据**项目阶段、团队规模、业务复杂度**做评分修正。

---

## 报告长什么样

每个扣分项都是这样的结构:

```
✗ 【高】Service 层直接依赖 JPA Entity
  - 文件:src/main/java/.../OrderService.java:42
  - 违反原则:DIP(依赖倒置),业务核心不应依赖具体持久化框架
  - 三层剖析:
    - 现象层:OrderService 引用 javax.persistence.EntityManager
    - 机制层:业务编排层穿透到了 ORM 层,改 ORM 就要改业务
    - 原理层:依赖方向应朝内,而非朝外指向框架
  - 改造建议:
    [改之前 → 改之后 完整代码对照]
```

最后给出:
- **Top 5 优先改造点**(按 ROI 排序)
- **3 阶段重构路线图**(止血 / 修复 / 优化)

---

## 设计哲学

- **代码引用必须有行号** —— 不允许"凭感觉"扣分
- **不只批评,给改造代码** —— 每个扣分项都附"改之前 → 改之后"
- **支持工程妥协** —— 如 `@Transactional` 放 Service 层,标注而非扣分
- **三层剖析** —— 现象层 / 机制层 / 原理层

---

## 项目结构

```
archcheck/
├── SKILL.md                            主入口,执行流程
├── README.md
├── references/
│   ├── dependency-direction.md         维度 1 规则库
│   ├── boundary-integrity.md           维度 2 规则库
│   ├── entity-vs-usecase.md            维度 3 规则库
│   ├── testability.md                  维度 4 规则库
│   ├── code-smells.md                  维度 5 规则库
│   ├── scoring-rubric.md               综合评分细则
│   └── language-patterns/
│       ├── java-spring.md              Java/Spring(主深)
│       ├── go.md
│       ├── python.md
│       └── typescript.md
└── assets/
    └── report-template.html            HTML 报告模板
```

---

## License

MIT
