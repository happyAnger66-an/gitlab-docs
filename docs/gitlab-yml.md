# quantization 仓 GitLab CI：`.gitlab/` 与流水线原理

**结论**：`.gitlab/` 不是独立系统，而是本仓 **GitLab CI 的拆分配置（路由表）**。GitLab 只认仓库根 `.gitlab-ci.yml`；它 `include` 全部 YAML 后，按 Board 传入的 `TASK_TYPE` / `TARGET` / `COMPARE_MODE` 决定跑哪条 job。量化/编译算法不在 YAML 里，YAML 只负责**何时跑、跑在哪台机器**，真正执行在 `.cicd/*.sh`。

相关：[`quantization/trigger.md`](./quantization/trigger.md)（Board 如何 trigger 本仓）。

---

## 1. GitLab CI 怎么加载这些文件

GitLab 对每个 pipeline 只读根文件 `.gitlab-ci.yml`。本仓根文件几乎只有 `include`：

```yaml
# quantization/.gitlab-ci.yml
include:
  - local: "/.gitlab/comm.yml"
  - local: "/.gitlab/mr.yml"
  - local: "/.gitlab/x86.yml"
  - local: "/.gitlab/orin.yml"
  - local: "/.gitlab/nts.yml"
  - local: "/.gitlab/mlc.yml"
  - local: "/.cicd/.benchmark-ci.yml"
  - local: "/.cicd/.kernel-workload.yml"
  - local: "/.cicd/.allspark.yml"
  - local: "/.cicd/msir/.msir-ci.yml"
  - local: "/.gitlab/check.yml"
```

拼装后的效果：

```text
Board Trigger
  variables[TASK_TYPE / TARGET / COMPARE_MODE / ...]
        │
        ▼
.gitlab-ci.yml  include 全部 yml  →  一条逻辑 pipeline
        │
        ├─ .gitlab/comm.yml      全局镜像、stages、默认变量、x86 before_script
        ├─ .gitlab/x86.yml       A100 export
        ├─ .gitlab/nts.yml       NTS export
        ├─ .gitlab/orin.yml      Orin 模板（本身几乎没有业务 job）
        ├─ .gitlab/mlc.yml       LLM compile
        ├─ .gitlab/check.yml     对点 / kernel check
        ├─ .gitlab/mr.yml        MR 拦截与冒烟
        └─ .cicd/...             MSIR compile / benchmark（不在 .gitlab/ 里）
```

Board 调 `POST /api/v4/projects/{id}/trigger/pipeline` 时**不指定 job 名**，只传 `ref` + `variables[...]`。GitLab 加载上述全部定义，每个 job 用 `only: variables` 或 `rules` 自己决定是否进入这次 pipeline。

---

## 2. 原理：variables 路由，而不是点名 job

这是本仓 CI 和 Board 的契约。

```text
Board 组 variables
  TASK_TYPE=common | msir_release | mlc_llm | ...
  TARGET=a100 | ntsctl,orin | ...
  COMPARE_MODE=x86_x86 | ...     # 仅 Compare
        │
        ▼
GitLab 对每个 job 求值 only/rules
  命中 → 进 pipeline，按 tags 分配 runner
  未命中 → 本 job 不出现
        │
        ▼
job.script  source .cicd/*.sh
  跑 quantization.py / mlc_run.sh / msir_run.sh
```

因此：

- 改「谁能被 Board 打中」→ 改 `.gitlab/*.yml`（或 `.cicd/msir/.msir-ci.yml`）的 `only`
- 改「打中之后干什么」→ 改 `.cicd/*.sh` 或 `quantization.py`
- `comm.yml` 的 `stages` 只决定**已命中 job 的先后顺序**，不决定谁被选中

常见匹配字段：

| 变量 | 谁设 | 用来选 |
|------|------|--------|
| `TASK_TYPE` | Board `mapTaskType` / export 的 `type` | export vs msir_* vs mlc_llm |
| `TARGET` | Board `stage_config.target` | a100 / ntsctl / orin / allspark / a40 |
| `CKPT_TARGET` | Board compile | `save_checkpoint` 打 NTS 还是其它 |
| `COMPARE_MODE` | Board Compare | `.gitlab/check.yml` |
| `EXPORT_ID` | Board export 固定 `default` | x86 模板走「只 export」短路径 |
| `CI_PIPELINE_SOURCE` | GitLab 自己 | MR job（`mr.yml`） |

---

## 3. `.gitlab/` 各文件职责

| 文件 | 角色 | 谁会命中 |
|------|------|----------|
| **`comm.yml`** | 全局底座：默认镜像、`stages`、compiler/msir_pipe/mlc 版本、x86 `before_script` | 每次 pipeline 都生效 |
| **`x86.yml`** | A100 上 CNN **export** | `TARGET` 含 `a100`，且不是 msir_* / mlc |
| **`nts.yml`** | 训练机上 CNN **export**（占 NTS 再跑） | `TARGET` 含 `ntsctl`，排除 msir_* / mlc_llm |
| **`orin.yml`** | Orin 模板：清目录、抢 SOC、ssh 到板子 | 被 mlc / check / msir-ci `extends` |
| **`mlc.yml`** | LLM compile（a100 / a40 / orin / allspark） | `TASK_TYPE == mlc_llm` |
| **`check.yml`** | 两次任务对点、Cutlass/Ansor check | `COMPARE_MODE=...` |
| **`mr.yml`** | MR 标题检查 + bev/plannn 冒烟 | `merge_requests`，与 Board 无关 |

### 3.1 `comm.yml`：CI 总开关

- 默认 image：`harbor-ad2.nioint.com/aicompiler/laser-docker:...`
- 阶段顺序：

```text
gen_config → benchmark → ci_test → export → checkpoint
  → build_model → compile_ckpt → unify_check → regression
  → benchmark-1..4 → kernel_workload
```

- 默认包版本：`release_system_version`、`msir_pipe_version`、`mlc_llm_version`、AllSpark 相关
- 锚点 `.x86_env_prepare`：装 msir_pipe、拉 Laser whl、准备 x86 环境
- 顶层 `before_script` 引用该锚点；**job 自己写了 `before_script` 则不会走这套**（如 mlc / nts / orin）

没单独写 `before_script` 的 x86 job 都会走这套准备。

### 3.2 `x86.yml`：Board export（A100）

模板 `.x86_ppl_template` 按变量分叉：

```text
有 EXPORT_ID（Board 设为 default）
  → 只跑 .cicd/2_run_quantization.sh     # JUST_EXPORT，出 .model
否则
  → 生成版本 → 量化 → post → benchmark → 推模型信息
```

唯一业务 job：`export_a100`

- `stage: export`，`tags: gitlab-runner-idc-ai-compiler-group-a100`
- `JUST_EXPORT: "TRUE"`
- `only`：`$TARGET =~ /a100/`，且 `TASK_TYPE` 不是 `msir_release` / `msir_deploy` / `msir_regression` / `msir_compatible` / `msir_update_guard` / `tuning_ansor`

这就是 Board `stage=export, target=a100` 打中的那条。失败时 `after_script` 发飞书。

### 3.3 `nts.yml`：Board export（NTS）

K8s runner（`gitlab-runner-eks-ai-compiler-group`）只做壳：

1. `0_prepare_ntsctl_env.sh` 占训练机
2. 组 `QUANT_ARGS="-tt ... -m ... ${EXTRA_ARGS}"`
3. `2_nts_trigger_run.sh` 把真正的 `quantization.py` 丢到 GPU 实例
4. `after_script` 释放 `ntsctl` 实例

业务 job：`export_nts`

- `only`：`$TARGET =~ /ntsctl/`，排除 `msir_*` 和 `mlc_llm`
- 对应 Board `target=ntsctl` 的 export

`.nts_prepare_template` 还会被 `.cicd/msir/.msir-ci.yml` 的 NTS compile job `extends`。

### 3.4 `orin.yml`：模板，不是业务入口

只定义两个隐藏 job（点号开头，不会单独跑）：

| 模板 | 作用 |
|------|------|
| `.clean_repo_hook` | `pre_get_sources_script`：checkout 前清 Orin runner 工作目录，必要时 ssh 到 s1–s4 杀占目录进程 |
| `.orin_template` | `tags: ccc-runner${STABLE_RELEASE}`；`0_get_available_soc.sh` 抢 SOC |

真正的 Orin 业务 job：

- LLM：`.gitlab/mlc.yml` 的 `mlc_compile_orin`
- Compare：`.gitlab/check.yml` 的 `model_compare_orin_*`
- CNN compile：`.cicd/msir/.msir-ci.yml` 的 `msir_compile_ckpt_orin` 等

它们都 `extends: .orin_template`，再 ssh 到板子跑 `.cicd` 脚本。

### 3.5 `mlc.yml`：Board compile（`TASK_TYPE=mlc_llm`）

| Job | Runner | 匹配 |
|-----|--------|------|
| `mlc_compile_a100` | a100 | `$TARGET =~ /ntsctl/` |
| `mlc_compile_a40` | a40 | `$TARGET =~ /a40/` |
| `mlc_compile_orin` | Orin 模板 | `$TARGET =~ /orin/`，ssh 跑 `mlc_run.sh` |
| `mlc_compile_allspark` | a100 | `$TARGET =~ /allspark/` |

对应 Board `mapTaskType`：`type=mlc_llm` 原样下发。脚本统一 `.cicd/mlc_run.sh`（clone mlc-llm → compile → `mlc_post.py` 回调）。

注意：`mlc_compile_a100` 的匹配是 `TARGET` 含 **`ntsctl`**（Board 把 a100 编译目标写成 ntsctl），不是字面 `a100`。

### 3.6 `check.yml`：Compare / kernel 检查

命中变量是 **`COMPARE_MODE`**，不是 `TASK_TYPE`。Board `POST /tools/compare/trigger` 走这里。

| `COMPARE_MODE` | Job | 做什么 |
|----------------|-----|--------|
| `x86_x86` | `model_compare_x86_x86` | `scripts/msir_job_comp.py` 两次 x86 任务对点 |
| `orin_orin` | `model_compare_orin_orin` | 板子上两次 Orin 对点 |
| `orin_x86` | `model_compare_collect_orin` → `model_compare_x86`（`needs`） | Orin 采集 + x86 对比 |
| `check_cutlass_x86` | `model_check_cutlass_x86` | `cutlass_kernel_check.py` |
| `check_ansor_x86` | `model_check_ansor_x86` | `ansor_kernel_check.py` |
| `p1_x86` | `model_compare_collect_p1` → `model_compare_p1_x86` | AllSpark P1 对点 |

### 3.7 `mr.yml`：和 Ocean 任务无关

只在 **Merge Request** 上跑（`only: merge_requests` / `rules: merge_request_event`）。Board trigger 的 pipeline **不会**走这些。

| Job | 作用 |
|-----|------|
| `check_mr_title` | 标题须形如 `m-123456 [feature] 描述`，并评论飞书需求链接 |
| `bev_s1_quant` / `bev_s3_quant` | 对应目录有改动时冒烟 `quantization.py` |
| `plann2_quant` | 验 MLC / plannn2 路径 |

---

## 4. `.gitlab/` 和 `.cicd/` 怎么分工

| 目录 | 职责 |
|------|------|
| **`.gitlab/*.yml`** | **声明** job：何时跑、跑在哪台 runner、匹配哪组 variables |
| **`.cicd/*.sh`** | **执行**：装环境、解析参数、调 `quantization.py` / `mlc_run.sh` / `ntsctl` |
| **`.cicd/msir/.msir-ci.yml`** | CNN **compile / save_checkpoint / deploy** 的 job 声明（同样被根文件 include） |

`.gitlab` 里几乎都是 `source .cicd/xxx.sh`。Board 的契约在 YAML 的 `only: variables`；真正干活在 `.cicd`。

CNN 的 compile **不在** `.gitlab/`。Board `TASK_TYPE=msir_release` 打的是 `.cicd/msir/.msir-ci.yml`（如 `msir_quant_ckpt_nts`、`msir_compile_ckpt_orin`），不是 `x86.yml` 的 `export_a100`。

其它被 include、但不在 `.gitlab/` 的：

| 文件 | 大致用途 |
|------|----------|
| `.cicd/.benchmark-ci.yml` | 测速 / benchmark 阶段 |
| `.cicd/.kernel-workload.yml` | kernel workload |
| `.cicd/.allspark.yml` | AllSpark 相关 job |

---

## 5. 和 Board variables 的对照

```text
Board stage=export, TARGET=a100
  →  .gitlab/x86.yml              export_a100

Board stage=export, TARGET=ntsctl
  →  .gitlab/nts.yml              export_nts

Board stage=compile, TASK_TYPE=mlc_llm
  →  .gitlab/mlc.yml              mlc_compile_*

Board stage=compile, TASK_TYPE=msir_release / msir_deploy
  →  .cicd/msir/.msir-ci.yml      msir_quant_ckpt_* / msir_compile_ckpt_*

Board /tools/compare, COMPARE_MODE=...
  →  .gitlab/check.yml

GitLab MR
  →  .gitlab/mr.yml

每次 pipeline 的底座
  →  .gitlab/comm.yml
```

`export_a100` 的 `only` 故意排除 `msir_*`，避免 compile pipeline 误跑 export job。`export_nts` 同样排除 `mlc_llm`，避免 LLM compile 的 `TARGET=ntsctl` 误跑 CNN export。

---

## 6. 一次典型 pipeline 长什么样

### export（A100）

```text
include 全部 yml
  → comm.yml 注入 image / stages / before_script
  → 仅 export_a100 的 only 为真
  → runner: a100
  → 1_parse_args.sh → 2_run_quantization.sh
  → quantization.py（JUST_EXPORT）→ 上传 .model → 回调 Board
```

### compile（MSIR release，NTS + Orin）

```text
Board 第二次 trigger（export 成功之后）
  TASK_TYPE=msir_release  TARGET=ntsctl,orin
  → .gitlab 里的 export/mlc/check 全不命中
  → .cicd/msir/.msir-ci.yml 命中
       msir_quant_ckpt_nts（CKPT_TARGET=ntsctl）
       msir_compile_ckpt_orin（TARGET 含 orin，extends .orin_template）
```

### LLM compile

```text
TASK_TYPE=mlc_llm  TARGET=ntsctl,orin
  → mlc_compile_a100 + mlc_compile_orin
  → .cicd/mlc_run.sh
```

---

## 7. 改配置时怎么找文件

| 你想改 | 去哪 |
|--------|------|
| 默认 docker / compiler / msir_pipe 版本 | `.gitlab/comm.yml` |
| Board export 打不中 / 误打中 | `.gitlab/x86.yml` 或 `nts.yml` 的 `only` |
| LLM 编译机型 | `.gitlab/mlc.yml` |
| Orin 抢机、清目录 | `.gitlab/orin.yml` |
| Compare 对点 | `.gitlab/check.yml` |
| CNN 量化后编译、落 ckpt | `.cicd/msir/.msir-ci.yml` |
| export 实际命令 | `.cicd/1_parse_args.sh`、`2_run_quantization.sh` |
| MLC 实际命令 | `.cicd/mlc_run.sh` |

---

## 8. 一句话

`.gitlab-ci.yml` 是目录；`.gitlab/*.yml` 是按**硬件 / 场景**拆开的 job 路由；`.cicd/` 是被路由点中之后的脚本。Board 只往 GitLab 灌 variables，本仓用 `only: variables` 把这些变量翻译成具体 runner 上的一条命令。
)
