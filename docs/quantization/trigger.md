# quantization 仓：定位、调用方与 Board Trigger

**结论**：`quantization` 不是量化算法库，而是 Laser / Model Ocean 的**模型配方 + 编译交付仓**。`laser_board_service` **不执行**本仓 Python，只通过 GitLab Trigger API 拉起 pipeline，再用 Repository API 读配置。真正跑 `quantization.py` / `msir_run.sh` / `mlc_run.sh` 的是 GitLab Runner。

GitLab：`ad/edge/service_component/ai_compiler/quantization`（Board 配置项 `quantization_project_id`，旧 Flask 里为 `553`）。

---

## 1. 仓库做什么

本仓提供三样东西：

| 资产 | 路径 | 作用 |
|------|------|------|
| 任务配方 | `{task}/config.json`、`model_ocean/configs/**/config_*.json` | 模型路径、IO、量化策略、后端（CUTLASS / TRT / AllSpark / MLC） |
| 校准数据入口 | `{task}/preprocess.py:get_feed_data` | 把车端数据喂给 MSIR |
| CI 壳 | `.gitlab/*.yml` + `.cicd/*.sh` | export / save_ckpt / compile / mlc / compare / check |

**不算** scale、不做 FakeQuant、不写 Relax/QNN 算子。算法在：

- CNN/BEV：`nio/tvm` MSIR
- LLM：仓外 TorchAO / AutoAWQ / SmoothQuant + `mlc-llm`

统一 CLI：

```bash
python3 quantization.py -tt scenecls -m xxx.pt -data ./data -trt FP16
```

### 1.1 两条主路径

```text
CNN/BEV（MSIR PTQ）
  export(.model) → save_checkpoint（校准 + FakeQuant + realize）→ compile_checkpoint（.so）

LLM / 类 LLM（MLC）
  （仓外已量化权重）→ convert_weight → compile graph_lib.so
```

Board 用 `TASK_TYPE` + `TARGET` 选路径，不点名 job。

### 1.2 和周边仓的边界

| 仓 / 服务 | 角色 |
|-----------|------|
| **quantization（本仓）** | 配方 + 校准数据 + CI 壳 |
| **laser_board_service** | 触发、落库、收 artifacts；**不做**量化/编译 |
| **nio/tvm MSIR** | FakeQuant、gather/calibrate、realize、CUTLASS/TRT |
| **mlc-llm** | `convert_weight` + Relax 编译 |
| **msir_pipe** | `save_checkpoint` / `compile_checkpoint`、上传、回调 |
| **laser_model_export** | 另一条 Trace 仓，与本仓并行 |
| **data_model** | export 的替代仓（`export_by=data_model`） |

---

## 2. 调用方

几乎所有正式调用都是：**调 Board → Board 对本仓 `trigger/pipeline`**。

```text
人 / 算法脚本
  ├─ laser-board-web
  └─ msir_pipe.TaskScheduler
        │
        ▼
laser_board_service（或旧 laser_board）
        │  GitLab trigger  project=quantization
        ▼
quantization 仓 CI
  export / quant-ckpt / compile / mlc / compare / check
        │
        ├─ 调 nio/tvm、mlc-llm、msir_pipe
        └─ 回调 Board
```

| 调用方 | 怎么调 | 用来干什么 |
|--------|--------|------------|
| **laser_board_web** | `POST /model_ocean/task/trigger` | 看板上点 export / compile / 取消 / 重试 |
| **laser_board_service** | `TriggerTaskPipelines` → GitLab Trigger | 主编排：落库、组 variables、拉 pipeline |
| **laser_board（旧 Flask）** | 同一套 `QUANTIZATION_PROJECT_ID` | 遗留看板，逻辑等价 |
| **msir_pipe.TaskScheduler** | `export_by` / `compile_by` 默认 `quantization` | 业务脚本组任务再打 Board；自动指向 `model_ocean/configs/{pipe}/{model}/config_{version}.json` |
| **Compare 工具** | `POST /tools/compare/trigger` | 仍 trigger 本仓，用 `COMPARE_MODE` 命中 `.gitlab/check.yml` |
| **本仓 CI / 本地** | Runner 或 `python3 quantization.py` | 执行端，不是上游调用方 |

`nio/tvm`、`mlc-llm` 是**被本仓调用**，不是调用本仓。

---

## 3. Board 怎么 trigger 本仓

Board **不 import、不 exec** 本仓代码。它只做：

1. 用 Trigger API 拉起 pipeline
2. 用 Repository API 读 `model_ocean/` 配置
3. 收回调、轮询 pipeline 状态，串 export → compile

### 3.1 入口

```text
POST /model_ocean/task/trigger
  router.go
    → task_trigger_handler.go::TriggerTaskPipeline
      → task_trigger_service.go::TriggerTaskPipelines
```

请求体核心字段（`TaskTriggerRequest`）：

| 字段 | 含义 |
|------|------|
| `name` / `version` / `user` / `alias` | 任务名、版本、触发人 |
| `task_info` | cosine / torch / task_tag 等 |
| `stage_config[]` | 每项一个阶段：`stage=export\|compile`，含 `branch`、`target`、`type`、`export_by`、模型路径等 |

`stage_config` 为空则直接报错。`export_by` 空则默认 `quantization`。

### 3.2 选仓：project_id + token

配置（`GitLabConfig`）：

| 配置项 | 用途 |
|--------|------|
| `quantization_project_id` | 本仓 GitLab 项目 |
| `quantization_export_trigger_token` | export pipeline |
| `quantization_compile_trigger_token` | compile pipeline |
| `quantization_read_token` | 读分支 / tree / pipeline 状态 |
| `data_model_project_id` + export token | 仅当 `export_by=data_model` |

`resolveTaskTriggerProjectAndToken`：

| 阶段 | 条件 | 打到哪 |
|------|------|--------|
| **compile** | 一律 | **quantization**（compile token） |
| **export** | `export_by=quantization`（默认） | **quantization**（export token） |
| **export** | `export_by=data_model` | data_model 仓 |

### 3.3 真正的 HTTP 调用

`TriggerTaskPipeline` → `GitLabClient.TriggerRepoPipeline`：

```http
POST {base_url}/api/v4/projects/{quantization_project_id}/trigger/pipeline
Content-Type: multipart/form-data

token=<export|compile trigger token>
ref=<stage_config.branch>          # 例如 master_2.0
variables[TASK_TYPE]=...
variables[TARGET]=...
variables[EXPORT_CONFIG]=...       # export：JSON 字符串
variables[REQUEST]=...             # compile：JSON 字符串
```

Board **不指定 job 名**。本仓 `.gitlab/*.yml` 用 `only: variables` 自己匹配。

实现：`laser_board_service/src/utils/gitlab_pipeline.go` 的 `TriggerRepoPipeline`。

---

## 4. 组给本仓的 variables

`expandTriggerRequests` 把 `stage_config[]` 拆成 export / compile 两次 trigger，各自一套变量。

### 4.1 export → `export_a100` 等

`buildExportStageVariables`：

| 变量 | 含义 | 本仓怎么用 |
|------|------|------------|
| `TASK_TYPE` | 如 `common` | `.gitlab/x86.yml` 的 `only`；`1_parse_args.sh` 映射 `-tt` |
| `TARGET` | 如 `a100` | 匹配 `export_a100`（`$TARGET =~ /a100/`） |
| `MODEL_PATH` / `IO_INFO_PATH` | 源模型与 IO | `quantization.py -m / -io-info` |
| `EXTRA_ARGS` | 常含 `-tc ./model_ocean/configs/.../config_xx.json` | 指定配方 |
| `EXPORT_CONFIG` | name / callback / hpc whl | 回调 Board、上传 `.model` |
| `EXPORT_ID` | `default` | 非空则只 export，不走全量 quant |
| `COSINE_VERSION` / `GITLAB_USER_LOGIN` | 环境、通知 | CI / 飞书 |

`type=common` 且没带 `-tc` 时，Board 自动拼：

```text
./model_ocean/configs/{name 转路径}/config_{version}.json
```

`EXPORT_CONFIG.callback` = `{board_public_url}/model_ocean/task/export`。

本仓匹配（`.gitlab/x86.yml`）：

```yaml
export_a100:
  stage: export
  variables:
    JUST_EXPORT: "TRUE"
  only:
    variables:
      - $TARGET =~ /^.*a100.*/ && $TASK_TYPE != "msir_release" && ...
```

`EXPORT_ID` 非空 → `2_run_quantization.sh` → `python3 quantization.py -tt ... -m ... ${EXTRA_ARGS}`。

### 4.2 compile → `msir_*` / `mlc_compile_*`

`buildCompileStageVariables`：

| 变量 | 含义 |
|------|------|
| `TASK_TYPE` | `mapTaskType` 后的值 |
| `TARGET` | `ntsctl,orin` 等 |
| `REQUEST` | JSON：model、callback、`task_kwargs` |
| `STORE` | `release_system` |
| `CKPT_TARGET` | 决定 `save_checkpoint` 打哪 |

`mapTaskType`（仅 compile）：

| `stage_config.type` | 本仓 `TASK_TYPE` |
|---------------------|------------------|
| `release` | `msir_release` |
| `release_new` | `msir_release_new` |
| `mlc_llm` | `mlc_llm` |
| 其它 / 空 | `msir_deploy` |

本仓对应 job：

| 条件 | Job |
|------|-----|
| `TASK_TYPE=msir_release` + `CKPT_TARGET=ntsctl` | `msir_quant_ckpt_nts` |
| `TASK_TYPE=msir_release` + `TARGET` 含 orin | `msir_compile_ckpt_orin` |
| `TASK_TYPE=mlc_llm` + `TARGET` 含 ntsctl | `mlc_compile_a100` |
| `TASK_TYPE=mlc_llm` + allspark / orin / a40 | `mlc_compile_*` |

`REQUEST.callback` = `{board_public_url}/model_ocean/task/compile`。

compile 前 Board 还会读本仓 `model_ocean/ocean_info.json` 的 `name_strategy`，写入 `REQUEST.task_kwargs`（失败静默降级）。

---

## 5. 阶段链：export 成功后再 trigger compile

`TriggerTaskPipelines` 若同时启用 export 和 compile，**先只 trigger export**，compile 记成 `waiting`（`shouldDeferCompileTrigger`）。

```text
trigger 请求（export + compile）
  → 落库 quant_task_record
  → 只 POST export pipeline
  → compile pipe_info.status = waiting

本仓 export 跑完
  → POST {board}/model_ocean/task/export
       artifacts.export.model = release_system:xxx.model:<task_id>

Board poll
  → 写 artifacts
  → maybeTriggerCompileAfterExport
       把 .model 填进 REQUEST.model
  → 再 POST 一次 compile pipeline（仍打本仓）
```

一次 Ocean 任务对本仓通常是 **两次 GitLab pipeline**。export 失败则 compile 标 `skipped`，不触发。

---

## 6. 反向：本仓怎么交回 Board

Board 塞进 variables 的 callback：

```text
TaskCallbackURL(stage) = {public_url}/model_ocean/task/{export|compile}
```

本仓 `export_for_msir_pipe` / `mlc_post.py` / `msir_pipe` POST 到这些 URL。Board 路由：

| 路径 | 作用 |
|------|------|
| `POST /model_ocean/task/export` | 收 export 回调（与 poll 同 handler） |
| `POST /model_ocean/task/compile` | 收 compile 回调 |
| `POST /model_ocean/task/poll_status` | 通用 poll |

`PollTaskPipelinesStatus` 再用 **read token** 查本仓 pipeline 状态（`GetPipelineStatus`），合并 artifacts。Board 不跑量化，只编排和记账。

---

## 7. 除了 trigger，Board 还会读本仓

| 场景 | API | 读什么 |
|------|-----|--------|
| compile 兼容 | `GetRepositoryFileRaw` | `model_ocean/ocean_info.json` → `name_strategy` |
| 看板选模型/版本 | `repository_tree` | `model_ocean/configs/**/config_*.json` |
| 分支列表 | GitLab branches | 本仓分支 |
| 状态刷新 | pipelines / jobs API | 轮询 `pipe_id` |

`POST /model_ocean/task/update_repo_info` → `SyncOceanBranchConfigsFromGitLab`：扫 `model_ocean/configs`，解析 `config_*.json` → `(task_name, version)` 落库。

---

## 8. 本仓 CI 如何接住 variables

```text
.gitlab/comm.yml     镜像、阶段、默认 compiler / msir_pipe / mlc 版本
.gitlab/x86.yml      export_a100 等（JUST_EXPORT）
.gitlab/nts.yml      NTS 上 export / quant
.gitlab/orin.yml     Orin 真机
.gitlab/mlc.yml      mlc_compile_{a100,a40,orin,allspark}
.cicd/msir/.msir-ci.yml   save_checkpoint / compile_checkpoint / deploy
.gitlab/check.yml    COMPARE_MODE 对点
```

匹配原则：`$TASK_TYPE` + `$TARGET`（及 `CKPT_TARGET` / `COMPARE_MODE` / `guard_level`）。

CNN export 落地：

```text
1_parse_args.sh → CI_QUANT_TT / CI_MODEL_PATH / CI_DATA_PATH
2_run_quantization.sh → python3 quantization.py -tt ${CI_QUANT_TT} -m ... ${EXTRA_ARGS}
quantization.py → export_for_msir_pipe() → 上传 .model → callback
```

LLM compile 落地：

```text
.cicd/mlc_run.sh
  → clone mlc-llm
  → 按 task_tag 找 model_ocean/configs/.../config_*.json
  → mlc-llm/compile/mlc_compile.py
  → mlc_post.py 上传并回调
```

---

## 9. 关键代码索引

**Board**

- `router/router.go` — `/model_ocean/task/trigger|export|compile|poll_status`
- `src/handlers/quant_task/task_trigger_handler.go` — HTTP 入口
- `src/services/quant_task/task_trigger_service.go` — 拆阶段、组 variables、选仓、trigger
- `src/services/quant_task/task_stage_chain.go` — export 成功后再 trigger compile
- `src/services/quant_task/task_poll_service.go` — 回调、artifacts、状态
- `src/services/quant_task/branch_sync_service.go` — 扫 `model_ocean/configs`
- `src/utils/gitlab_pipeline.go` — `TriggerRepoPipeline`
- `src/structs/config.go` / `quant_task_record.go` — GitLab 配置与请求体
- `config/config.go` — `TaskCallbackURL`

**本仓**

- `quantization.py` — `pipeline_msir` / `export_for_msir_pipe`
- `.gitlab/x86.yml`、`mlc.yml`、`check.yml`
- `.cicd/1_parse_args.sh`、`2_run_quantization.sh`、`mlc_run.sh`、`mlc_post.py`
- `model_ocean/configs/`、`model_ocean/ocean_info.json`

**客户端**

- `msir_pipe/client/task_scheduler.py` — `export_by` / `compile_by` 默认本仓

---

## 10. 命名说明

`quantization` 只覆盖 CNN PTQ 一段。调用方要的是：配方、export、compile、对点、多后端交付。Board / `msir_pipe` 字段已是 `export_by` / `compile_by`，产品名是 **Model Ocean**。

更贴切的仓名：**`laser-model-ocean`**（模型配方与编译交付仓）。不建议继续用带 `quant` 的名字，否则 LLM / 纯 FP16 compile 会被误解。
)
