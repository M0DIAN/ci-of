# CI Optimization Framework

Conservative, fail-closed CI optimization: deterministic risk-tier
classification of change ranges and exact-tree post-merge FULL reuse for
GitHub Actions. Runtime is Python 3.11+ stdlib-only, with a single
strict configuration file (`ciopt.toml`).

> **Provenance.** This framework was extracted and generalized from the
> production CI governance and optimization system developed for
> MarketVault. Source extraction baseline:
> [M0DIAN/market-vault](https://github.com/M0DIAN/market-vault) at
> `99840349bdab4f0dc56a420bc66a1556750d1878`. The extraction carries
> forward only the production-proven, fail-closed V1 contracts.

**Repository:** <https://github.com/M0DIAN/ci-of>

## Problem

CI often reruns expensive validation even when the risk profile does not
require it. A documentation-only change reruns the same hours of tests
as a change to core code, and every merge to `main` reruns the entire
FULL matrix that the merged pull request already validated.

## Framework strategy

```
CHANGE
  ↓
RISK CLASSIFIER
  ├─ docs_fast
  ├─ package_docs
  ├─ control_plane
  └─ full

For FULL PR:

RUN FULL
  ↓
CREATE ATTESTATION

After squash merge:

PROVE SAME TESTED TREE
  ↓
YES → reuse PR FULL evidence
NO  → run FULL again
```

The classifier maps a change range to one of four stable tiers:

| tier | meaning |
| --- | --- |
| `docs_fast` | every changed path is inside the configured docs scope (validated generic fast path) |
| `package_docs` | every changed path is docs or a package-doc file (e.g. README), and a package-doc file changed — core tests may skip, package validation MUST still run |
| `control_plane` | OPT-IN, disabled by default: every changed path is inside the exact configured control-plane eligible subset — a downstream-specific conservative validation surface |
| `full` | everything else — including empty diffs, unknown paths, and any failure to classify |

**Component impact is exposed separately from the tier and never
authorizes skip.** A registered `[components.*]` surface may report
`independent_only=true`, but the tier stays `full` unless an explicit
validated validation contract exists.

**The tiers are per-surface, not a single switch.** A tier authorizes a
skip only for the surface whose validation is unnecessary under that
tier's contract: `docs_fast` may skip both the core matrix and package
validation; `package_docs` may skip the core matrix but MUST still run
package validation (a package-doc change such as README is package
metadata); `control_plane` is never claimed by the generic template —
the framework does not auto-provide a validated control-plane subset.

## Security principle

> Cannot prove safe optimization
> ⇒ do more work.
>
> Never:
> cannot prove
> ⇒ skip work.

The framework fails closed everywhere. Unknown / unset / malformed
config, unknown paths, empty diffs, unresolvable refs, git or network
failures, missing or stale evidence — every one of them lands on
`full` / `reuse=false`, and the workflow runs normal FULL validation.

Post-merge FULL reuse (V1) is authorized **only** when every proof
holds: exact push shape, one-parent squash topology, exactly one
associated merged PR with exact base/head identity, a completed
successful `pull_request` run of the configured workflow on the exact
head SHA, a job set that terminates SUCCESS on exactly the configured
formal jobs (no missing/duplicate/extra), an attempt-bound attestation
matching every identifier, **tree equivalence** between the current
`main` tree and the tree the PR run tested, and no control-plane
mutation. Anything else ⇒ `POST_MERGE_REUSE=false` ⇒ fresh FULL.

**Proof failure is never a CI failure.** `reuse=false` just means the
workflow runs the validation itself.

## Installation

**v0.1.0 is not published to PyPI.** Install it from source.

For local framework development, install the checkout in editable mode
with the dev extras:

```console
python -m pip install -e ".[dev]"
```

Downstream repositories pin the framework source with a git install.
The v0.1.0 release of the framework lives in this repository:

```console
python -m pip install \
  "git+https://github.com/M0DIAN/ci-of.git@v0.1.0"
```

`ci-opt` is the CLI; `python -m ci_optimizer` works too. Copy
`ciopt.example.toml` to `ciopt.toml` in your repository root and adapt
it (see [docs/configuration.md](docs/configuration.md)).

```console
ci-opt classify --config ciopt.toml \
  --mode pull_request --base <BASE_SHA> --head <HEAD_SHA>
```

`classify` prints JSON by default; `--output env` emits production
key=value lines; `--output github-env` emits only valid GitHub Actions
environment assignments (`CI_TIER`, `CI_TIER_REASON`, `CI_COMPONENTS`,
`CI_CORE_CHANGED`, `CI_PACKAGE_CHANGED`, `CI_UNKNOWN_CHANGED`,
`CI_SHARED_CHANGED`, `CI_INDEPENDENT_ONLY`, `CI_FULL_MATRIX_REQUIRED`,
`CI_CHANGED_FILES`) — the renderer used by the workflow templates.

## Adoption path

The framework is designed to be adopted gradually. **Strongly
discouraged: enabling all optimizations at once.**

1. **Phase 1 — classification only / observe.** Add the classifier to
   your workflow and log `CI_TIER` for every run. Nothing is skipped yet.
2. **Phase 2 — enable safe tiers per surface.** Once the observed
   classifications match your expectations, guard each heavy step with
   its own tier: `docs_fast` may skip any heavy surface; `package_docs`
   may skip the core test matrix but must still run package validation.
   `control_plane` stays disabled (empty `control_plane_eligible`):
   activate it only after implementing and reviewing a dedicated
   conservative control-plane validation surface, populating the
   eligible list, gating the FULL surfaces, and shadow-observing first.
3. **Phase 3 — enable post-merge FULL reuse.** Only after phases 1–2 are
   stable, add the attestation creation step and the post-merge reuse
   proof.

See [docs/adoption.md](docs/adoption.md) for the full guide.

## Documentation

- [docs/architecture.md](docs/architecture.md) — classifier boundary,
  config, evidence producer/consumer, GitHub API, workflow guards,
  exact-tree model
- [docs/configuration.md](docs/configuration.md) — full config schema
  reference
- [docs/security-model.md](docs/security-model.md) — trust boundary and
  attack/failure cases
- [docs/evidence-model.md](docs/evidence-model.md) — why tree equality,
  not commit equality, is the core proof
- [docs/adoption.md](docs/adoption.md) — staged adoption guide
- [docs/non-goals.md](docs/non-goals.md) — what v0.1.0 explicitly does
  not support
- [templates/github-actions/ci.yml](templates/github-actions/ci.yml) —
  downstream workflow template
- [examples/python-repository/](examples/python-repository/) — example
  config for a Python repository

## License

MIT — see [LICENSE](LICENSE). Copyright (c) 2026 M0DIAN.

---

# CI Optimization Framework 中文说明

保守、失败关闭（fail-closed）的 CI 优化框架：对代码变更范围进行确定性的风险分级，并在 GitHub Actions 中基于精确 Git Tree 证明复用合并后的 FULL CI 验证结果。

运行时要求 Python 3.11+，核心运行时仅依赖 Python 标准库，并使用单一严格配置文件 `ciopt.toml`。

> **项目来源。** 本框架从 MarketVault 项目中已经投入生产使用的 CI 治理与优化系统中抽取并泛化而来。
>
> MarketVault 源基线：
> [M0DIAN/market-vault](https://github.com/M0DIAN/market-vault)
>
> `99840349bdab4f0dc56a420bc66a1556750d1878`
>
> 当前独立框架仅保留已经经过生产验证、采用失败关闭原则的 V1 合约。

**项目仓库：** <https://github.com/M0DIAN/ci-of>

## 要解决的问题

传统 CI 经常在实际风险并不需要时重复执行昂贵的验证任务。

例如：

- 仅修改文档，却重新执行与核心代码修改相同的大量测试；
- Pull Request 已经完成完整 FULL 验证，但 squash merge 到 `main` 后又再次运行完全相同的完整矩阵。

CI Optimization Framework 的目标不是“尽可能少跑 CI”，而是：

> **只有在能够严格证明可以安全跳过时，才减少重复工作。**

无法证明时，始终执行更多验证，而不是减少验证。

## 框架策略

    变更
      ↓
    风险分类器
      ├─ docs_fast
      ├─ package_docs
      ├─ control_plane
      └─ full

    对于 FULL PR：

    执行 FULL
      ↓
    生成 Attestation

    Squash Merge 之后：

    证明当前 Tree 与已测试 Tree 完全相同
      ↓
    是 → 复用 PR FULL 证据
    否 → 再次执行 FULL

分类器会把一个变更范围映射到四个稳定风险等级之一：

| 等级 | 含义 |
| --- | --- |
| `docs_fast` | 所有变更路径均位于配置的文档范围内；这是经过验证的通用快速路径 |
| `package_docs` | 所有变更均属于文档或 package-doc 文件（例如 README），并且至少有一个 package-doc 文件发生变化；核心测试可以跳过，但 package validation **必须继续执行** |
| `control_plane` | **显式启用（OPT-IN）且默认关闭**；所有变更路径均位于精确配置的 control-plane eligible 子集中，需要下游项目自己提供经过审查的保守验证 surface |
| `full` | 其他所有情况，包括空 diff、未知路径以及任何分类失败 |

**组件影响分析与风险等级彼此独立，组件影响结果本身永远不能授权跳过验证。**

注册在 `[components.*]` 下的 surface 可以报告：

`independent_only=true`

但除非存在显式并且经过验证的验证合约，否则 tier 仍然保持：

`full`

**这些 tier 是按验证 surface 生效的，而不是一个全局开关。**

具体来说：

- `docs_fast` 可以允许跳过核心测试矩阵和 package validation；
- `package_docs` 可以跳过核心测试矩阵，但 **必须继续执行 package validation**；
- `control_plane` 不由通用模板自动宣称安全，框架本身不会自动提供一个经过验证的 control-plane 子集。

## 安全原则

> 无法证明优化是安全的
>
> ⇒ 执行更多工作。
>
> 永远不能：
>
> 无法证明
>
> ⇒ 跳过工作。

整个框架采用失败关闭设计。

以下任何情况：

- 配置未知、缺失或格式错误；
- 出现未知路径；
- diff 为空；
- Git ref 无法解析；
- Git 操作失败；
- 网络失败；
- 证据缺失；
- 证据过期；
- 证明链不完整；

都会落入：

    full
    reuse=false

随后 workflow 会执行正常的 FULL 验证。

Post-merge FULL reuse（V1）只有在**所有证明条件同时成立**时才允许启用，包括：

- push 形态精确符合要求；
- 使用单 parent 的 squash topology；
- 能唯一确定一个与 base/head identity 精确匹配的已合并 PR；
- 配置的 workflow 在精确 PR head SHA 上存在已完成且成功的 `pull_request` run；
- 正式 job 集合与配置要求完全一致；
- 不允许缺失、重复或额外 formal jobs；
- attempt-bound attestation 与所有相关 identifier 精确匹配；
- 当前 `main` tree 与 PR run 实际测试的 tree 完全相同；
- control-plane 没有发生不允许的修改。

任何一个条件不成立：

    POST_MERGE_REUSE=false

随后执行新的 FULL 验证。

**Reuse proof 失败本身并不代表 CI 失败。**

`reuse=false`

只表示 workflow 必须自己重新执行验证。

## 安装

**v0.1.0 当前未发布到 PyPI。**

请从源码安装。

本地开发框架时，可以在 checkout 中使用 editable mode 和开发依赖：

    python -m pip install -e ".[dev]"

下游仓库应固定到明确的框架 Git tag。

v0.1.0：

    python -m pip install \
      "git+https://github.com/M0DIAN/ci-of.git@v0.1.0"

CLI 命令为：

`ci-opt`

也可以使用：

`python -m ci_optimizer`

将：

`ciopt.example.toml`

复制为仓库根目录下的：

`ciopt.toml`

然后根据自己的项目进行调整。

完整配置说明：

[docs/configuration.md](docs/configuration.md)

分类示例：

    ci-opt classify --config ciopt.toml \
      --mode pull_request --base <BASE_SHA> --head <HEAD_SHA>

`classify` 默认输出 JSON。

使用：

`--output env`

会输出生产使用的 `key=value` 格式。

使用：

`--output github-env`

则只输出合法的 GitHub Actions 环境变量：

- `CI_TIER`
- `CI_TIER_REASON`
- `CI_COMPONENTS`
- `CI_CORE_CHANGED`
- `CI_PACKAGE_CHANGED`
- `CI_UNKNOWN_CHANGED`
- `CI_SHARED_CHANGED`
- `CI_INDEPENDENT_ONLY`
- `CI_FULL_MATRIX_REQUIRED`
- `CI_CHANGED_FILES`

这也是 workflow 模板实际使用的 renderer。

## 推荐接入方式

框架设计为逐步接入。

**强烈不建议一次性启用所有优化。**

### Phase 1 — 只分类，不跳过

首先只把 classifier 接入 workflow。

记录每一次运行的：

`CI_TIER`

此阶段不跳过任何任务。

先观察分类结果是否符合项目实际情况。

### Phase 2 — 按 surface 启用安全 tier

确认分类结果稳定后，再为每一个 heavy surface 单独设置 guard。

例如：

- `docs_fast` 可以允许跳过 heavy surface；
- `package_docs` 可以跳过核心测试矩阵；
- `package_docs` **不能跳过 package validation**。

`control_plane` 默认保持关闭：

    control_plane_eligible = []

只有在下游项目已经：

1. 实现专用、保守的 control-plane validation surface；
2. 完成人工审查；
3. 配置 eligible list；
4. 对 FULL surface 设置相应 gating；
5. 先进行 shadow observation；

之后才应启用。

### Phase 3 — 启用 post-merge FULL reuse

只有 Phase 1 和 Phase 2 已经稳定后，才接入：

- attestation producer；
- post-merge reuse proof。

完整接入指南：

[docs/adoption.md](docs/adoption.md)

## 文档

- [docs/architecture.md](docs/architecture.md) — 分类器边界、配置、证据 producer/consumer、GitHub API、workflow guards、精确 Tree 模型
- [docs/configuration.md](docs/configuration.md) — 完整配置 schema
- [docs/security-model.md](docs/security-model.md) — 信任边界及攻击/失败场景
- [docs/evidence-model.md](docs/evidence-model.md) — 为什么核心证明使用 Tree equality，而不是 commit equality
- [docs/adoption.md](docs/adoption.md) — 分阶段接入指南
- [docs/non-goals.md](docs/non-goals.md) — v0.1.0 明确不支持的内容
- [templates/github-actions/ci.yml](templates/github-actions/ci.yml) — 下游项目 workflow 模板
- [examples/python-repository/](examples/python-repository/) — Python 仓库示例配置

## 许可证

MIT — 参见 [LICENSE](LICENSE)。

Copyright (c) 2026 M0DIAN.
