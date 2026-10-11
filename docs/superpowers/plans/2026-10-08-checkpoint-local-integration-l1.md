# Checkpoint Local Integration L1 Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Проверить полный checkpoint lifecycle и восстановление после гибели процессов Nova/Cinder на постоянной двухтомной программной модели без доступа к СХД.

**Architecture:** Самостоятельный `checkpoint_lab` запускает процессы модели, Cinder и Nova, соединяя настоящий Nova executor с настоящим Cinder HTTP через штатный `CheckpointClient`. Состояние сервисов и модели сохраняется в раздельных файловых SQLite DB; внешний supervisor вводит отказы и независимо проверяет данные, effect counts, journals, guards и quotas. Синтетические runtime/storage/auth boundaries явно обозначаются в evidence.

**Tech Stack:** Python 3.11, Linux, stdlib unittest/sqlite3/subprocess/HTTP для управляющей части; отдельные Nova/Cinder environments из закреплённых public requirements/constraints, SQLAlchemy/Alembic, OVO и сериализованный fake oslo RPC внутри каждой сервисной группы.

**Spec:** [Согласованная спецификация L1](../specs/2026-10-08-checkpoint-local-integration-l1-design.md).

Статус: **согласован пользователем; исполнение Native начато**. Для исполнения
рекомендуется Native: последовательная работа в этой сессии с одним независимым
review комплекта после финальных проверок. Задачи тесно связаны интерфейсами;
метод исполнения Native подтверждён пользователем.

## Global Constraints

- «Будущий самостоятельный комплект: `work/checkpoint-local-integration/`.» Его Git repo отдельный; корень workspace и существующие checkout не переинициализировать.
- Nova commit `ce3c1c474b8fd04b6223cf8cf3af7079345d14b2`, tree `4d24555083fea5d547fe2e8ebf6ed39937c0733c`; Cinder commit `d065a55cbe5cb8cd59489752846a41145d72bd27`, tree `53888d9cea7fa1b12d35a0b9afd894eeee270843`.
- «Исторические bundles, Nova/Cinder production modules, миграции, policy, API/RPC версии и dependency pins остаются базой.» Код L1 размещается вне этих checkouts.
- «Команды сценария проходят через Nova API.» Recovery вызывает настоящий `RecoveryManager`; Cinder reconciliation — настоящий C1.1 manager. Нельзя заполнять journal/receipt результатами из supervisor.
- «В первом L1 API/conductor/compute работают внутри одного процесса Nova harness, а API/scheduler/volume workers — внутри одного процесса Cinder harness.» Имена целей отказа: `nova_group`, `cinder_group`; отдельный restart compute и RabbitMQ не заявляются.
- «Для данного L1 достаточно одной действующей cell»; cell0 при необходимости и API DB отдельны. Двухклеточная регрессия прежнего N2 не заменяется новой моделью.
- «Режим resume открывает те же файлы, проверяет manifest и версии; не пересоздаёт schema, не очищает строки, не генерирует новые resource UUID». Нельзя наследовать destructive `TestCase.setUp()`.
- «Отсутствие записи само по себе не возвращает `failed_no_effect`.» Нет blind retry, takeover или освобождения защиты по timeout.
- «Профили runtime/store/staging и сведения Neutron синтетические»; production allowlist не создаётся, положительных hardware profiles по L1 нет.
- Python 3.11 и disposable Linux. «Все endpoints — loopback и случайные свободные порты» внутри этой среды; реальные credentials/endpoints не принимаются.
- «Exit codes: `0` — все выбранные cases и обязательные проверки завершены; `1` — failed, timed out, crashed либо incomplete case; `2` — неверный input или preflight failure.» Subset success имеет `suite_complete=false`.
- Итог `local_simulated_integration`, `production_qualified=false`, `hardware_qualified=false`. Старые test counts не объявляются новым результатом.

## Review Focus

1. Два одновременных `resume` для одного каталога: второй runner отвергнут без второго эффекта; Task 2, `test_second_owner_cannot_resume`.
2. PID повторно использован ОС или ready-файл остался от старого запуска: supervisor не принимает чужой процесс за ребёнка и не посылает ему сигнал; Task 2, `test_stale_ready_and_foreign_pid_are_rejected`.
3. Bootstrap оборван между миграциями и seed: resume отвергает неполное состояние и не досоздаёт его молча; Tasks 3/4, `test_incomplete_bootstrap_cannot_resume`.
4. Прокси не увидел upstream response, но сценарий объявлен «ответ потерян после приёма»: case incomplete; Task 5, `test_drop_without_upstream_proof_is_incomplete`.
5. Параллельные worker/greenlet вызовы перепутали step/attachment IDs: эффекты остаются в собственном scope; Tasks 3/5, `test_effect_scope_is_local_to_worker`.

---

## 1. Рабочая область, доказательства и команды

Сначала выполнить read-only проверку существующих HEAD/status. При исполнении
использовать `superpowers:using-git-worktrees`: новый standalone repo не требует
ветки внутри Nova или Cinder. Создать его только если путь отсутствует; если
он появился, сначала проверить владельца и содержимое. Ветка нового repo:
`feature/checkpoint-local-integration-l1`. Не трогать пользовательские изменения.

Команды задач выполняются из корня этого будущего repo. `python3.11` означает
interpreter управляющего комплекта в disposable Linux. Nova и Cinder получают
собственные interpreters через `--nova-python` и `--cinder-python`. Тестовый CLI
— `python3.11 -m checkpoint_lab`; product interpreters не импортируются в него.

Протокол каждой задачи: `.superpowers/sdd/l1/task-N/` с фактическими commands,
RED/GREEN logs, commit SHA и решениями. Документы этой серии копируются в
`docs/design.md` и `docs/plan.md` нового repo как утверждённые входы, с hashes.
Runtime evidence и environments исключаются из Git. RED из-за сломанной
установки dependencies сначала исправляется как environment failure и не
считается подтверждением product/test требования.

## 2. Структура файлов и общие контракты

Все пути ниже относительно `work/checkpoint-local-integration/`.

| Файлы | Ответственность |
|---|---|
| `pyproject.toml`, `.gitignore`, `checkpoint_lab/__init__.py` | Лёгкий standalone package; supervisor использует stdlib |
| `checkpoint_lab/contracts.py`, `manifest.py` | Закрытые входы, source pins, immutable identities, serialization |
| `checkpoint_lab/model/{state,server,client}.py` | Постоянная модель, loopback API и клиент без retry |
| `checkpoint_lab/processes.py`, `preflight.py` | Владение процессами, launch identity, deadlines, проверка среды |
| `checkpoint_lab/cinder/{bootstrap,driver,auth,entry}.py` | Постоянный Cinder и test-only storage/auth adapters |
| `checkpoint_lab/nova/{bootstrap,runtime,adapters,entry}.py` | Постоянная Nova, model runtime и wiring настоящих N2 компонентов |
| `checkpoint_lab/faults.py`, `proxy.py` | Одноразовые barriers и наблюдаемая потеря HTTP ответа |
| `checkpoint_lab/observe.py`, `cases.py`, `oracle.py` | Чтение состояния, закрытая матрица сценариев и независимые ожидания |
| `checkpoint_lab/evidence.py`, `runner.py`, `__main__.py` | Полнота outcomes, CLI, итог и hashes |
| `environments/{nova,cinder}/`, `tools/prepare_environments.py` | Копии public requirements/constraints с provenance, установка только в выделенный prefix |
| `tests/unit/`, `tests/integration/`, `tests/fixtures.py` | Контрактные, компонентные, процессные и сквозные проверки |
| `README.md`, `VALIDATION.md`, `tools/package.py` | Подготовка среды, запуск, фактические границы результата и локальный пакет |

Task 1 определяет общие типы в `contracts.py`:

```text
Json = dict[str, Any]
SourcePin(commit: str, tree: str)  # frozen dataclass
RunManifest(run_id: str, case_id: str, root: Path,
            nova: SourcePin, cinder: SourcePin, ids: Json)  # schema_version=1
EffectKey(source: str, step_id: str, action: str, resource_id: str)
LabError(Exception)  # validation/setup error, не успешный effect
DuplicateDispatch(LabError)
```

`ids` содержит ровно deployment/cell/project/actor/service UUIDs, instance UUID,
host=`compute-l1`, node=`node-l1`, port UUID/MAC, два volume UUID и их backend/BDM/
initial attachment IDs. Volume devices — `/dev/vda`, `/dev/vdb`; initial backend
hosts — `cinder-l1@backend-a`, `cinder-l1@backend-b`. UUID создаются до запуска
сервисов и сохраняются; `resume` их только читает. Пути БД выводятся из `root`:
`nova-api.sqlite`, `nova-cell.sqlite`, `nova-cell0.sqlite`, `cinder.sqlite`,
`model.sqlite`; последний cell0 создаётся при bootstrap Nova, не используется
для выбранной ВМ. JSON запрет unknown fields, duplicate keys и неканонических
UUID; hash — SHA256 UTF-8 sorted-key JSON с фиксированными separators.

Сервисный ready envelope: `run_id`, `case_id`, `role`, `launch_id`, `pid`,
`control_endpoint`, `api_endpoint` (null у модели), `manifest_sha256`.
Supervisor выдаёт новый launch UUID на каждый `Popen`, проверяет совпадение
своего handle и ready envelope. Product IDs при этом не меняются.

Команды управления закрыты: `ready`, `observe`, `recover`, `expire_user`,
`revoke_service`. Изменение marker данных доступно только модельной команде
`guest_write` при RUNNING. Ни одного control endpoint для изменения journal,
guard, quota, receipt либо принудительного success.

### Task 1: Постоянная модель двух дисков и RAM

**Files:** Create `pyproject.toml`, `.gitignore`, package init files,
`checkpoint_lab/contracts.py`, `manifest.py`, `model/{state,server,client}.py`;
Test `tests/unit/test_manifest.py`, `test_model.py`, `tests/fixtures.py`.

**Interfaces:** Produces `create_manifest(root: Path, case_id: str) -> RunManifest`,
`read_manifest(path: Path) -> RunManifest`; `ModelState.open(path: Path, *, create: bool) -> ModelState`;
`ModelState.seed(manifest: RunManifest) -> None`, `execute(key: EffectKey, payload: Json) -> Json`,
`query(key: EffectKey) -> Json`, `observe() -> Json`, `guest_write(ram: bytes, disks: dict[str, bytes]) -> None`.
Auth API: `auth_state() -> Json`,
`set_auth_state(*, user_expired: bool | None = None, service_revoked: bool | None = None) -> None`.
`ModelClient(endpoint: str)` предоставляет те же execute/query/observe/guest_write
и auth методы по versioned test-only JSON протоколу, не выполняя network retry.

- [ ] **Step 1: Подготовить отдельный repo и failing tests модели.**
  Тестовый helper `seeded_model(tmp)` создаёт два UUID-тома из manifest и возвращает
  `(model, manifest)`; он принадлежит `tests/fixtures.py`, не product code.

```python
def test_disks_are_independent(self):
    model, manifest = seeded_model(self.tmp)
    v1, v2 = manifest.ids['volume_ids']
    model.guest_write(b'B:ram', {v1: b'B:disk-1', v2: b'B:disk-2'})
    self.assertNotEqual(model.observe()['disks'][v1], model.observe()['disks'][v2])

def test_missing_completion_is_unknown(self):
    self.assertEqual('unknown', self.model.query(self.key)['status'])
```

  Добавить `test_duplicate_execute_is_visible`, `test_query_changes_only_audit`,
  `test_foreign_snapshot_cannot_revert_volume`, `test_resume_never_reseeds`,
  `test_new_interpreter_preserves_model_state`. Повторный execute повышает
  `execute_attempts` до 2, `committed_effects` остаётся 1 и бросает DuplicateDispatch.
- [ ] **Step 2: RED.** `python3.11 -m unittest discover -s tests/unit -p 'test_model.py' -v`;
  ожидается отсутствие model API либо нарушенное требование, не проблема зависимостей.
- [ ] **Step 3: Реализовать модель.** Separate SQLite `attempts`, `effects`, `volumes`,
  `snapshots`, `domain`, `auth_state`; foreign keys, `journal_mode=WAL`,
  `synchronous=FULL`, transaction на data+completion. Audit attempt коммитится
  отдельно перед effect; unique `(source, step_id)` плюс immutable action/resource/
  payload hash обнаруживает изменённый повтор. Отсутствие completion — unknown.
  Максимум 1 MiB синтетического payload; bytes по wire кодируются hex.
  Model server: `python -m checkpoint_lab.model.server --manifest PATH --mode bootstrap|resume --launch-id UUID`.
- [ ] **Step 4: GREEN и component crash.** Повторить unit command; добавить дочерний
  интерпретатор с `os._exit` до/после commit и проверить reopen прежней model DB.
  Namespace `nova-runtime`/`cinder-storage`/`cinder-attachment` разделяет ключи.
- [ ] **Step 5: Зафиксировать evidence и commit.**
  `git add pyproject.toml .gitignore checkpoint_lab tests docs`;
  `git commit -m "test: add durable checkpoint lab model"`.

### Task 2: Владение процессами, preflight и подготовка среды

**Files:** Create `checkpoint_lab/processes.py`, `preflight.py`,
`tools/prepare_environments.py`, `environments/{nova,cinder}/{requirements.txt,constraints.txt}`,
`environments/nova/constraints-lab.txt`, `environments/provenance.json`,
`tests/unit/test_processes.py`, `test_preflight.py`,
`test_prepare_environments.py`; Modify `tests/fixtures.py`, `README.md` (создать).

**Interfaces:** Consumes RunManifest. Produces
`ProcessSupervisor(manifest: RunManifest, deadline: float)` (deadline — absolute monotonic time),
`start(role: str, argv: tuple[str, ...], env: dict[str, str]) -> Json`,
`stop(role: str, *, kill: bool) -> Json`, `restart(role: str) -> Json`, `close() -> None`.
`preflight(nova_checkout: Path, cinder_checkout: Path, nova_python: Path, cinder_python: Path) -> Json`.
`prepare_environments(prefix: Path, nova_checkout: Path, cinder_checkout: Path) -> Json`.

- [ ] **Step 1: Написать failing tests с маленьким stdlib child fixture.**

```python
def test_second_owner_cannot_resume(self):
    with ProcessSupervisor(self.manifest, self.deadline):
        with self.assertRaises(LabError):
            ProcessSupervisor(self.manifest, self.deadline).__enter__()

def test_stale_ready_and_foreign_pid_are_rejected(self):
    self.assertRaises(LabError, self.supervisor.accept_ready, self.stale_ready)
    self.assertEqual([], self.signals_sent_to_foreign_process)
```

  `accept_ready(envelope: Json) -> None` — метод supervisor из этой задачи.
  Добавить `test_old_owner_does_not_release_new_lock`, timeout/dead child tests;
  preflight cases wrong SHA, tracked diff, untracked `.py` под import roots,
  Python !=3.11, отсутствующий import, non-Linux, несуществующий interpreter.
- [ ] **Step 2: RED.** `python3.11 -m unittest discover -s tests/unit -p 'test_processes.py' -v`;
  также `test_preflight.py`, `test_prepare_environments.py`; сохранить failures.
- [ ] **Step 3: Реализовать lifecycle.** Lock каталога через `flock` на удерживаемом FD;
  fresh `Popen`/launch UUID, ready пишется atomic replace; stale ready не принимается.
  Сигналы только живому собственному Popen handle, без kill по PID из файла.
  `restart` всегда mode=resume, ждёт завершения старого процесса и сохраняет manifest.
  Sanitized env исключает унаследованные OpenStack credentials/OS_TEST/GIT overrides;
  передаются только whitelisted locale/path/параметры среды L1. Все socket clients
  имеют bounded timeout; expiry deadline возвращает incomplete outcome.
- [ ] **Step 4: Реализовать isolated environment preparation.** Копировать public requirements
  и constraints из закреплённых C1.1/N2 evidence, записать SHA256/provenance. Создавать
  новые `prefix/nova`, `prefix/cinder` Python 3.11 venv; устанавливать public и test
  dependencies под сохранёнными constraints, import source через явный PYTHONPATH.
  Для Nova использовать отдельный `constraints-lab.txt`: ровно один исторически
  подтверждённый override `libvirt-python===11.10.0` вместо upstream `11.0.0`, как
  в N2 `functional-package-versions.txt`; сохранить машинный diff и SHA обоих
  файлов. Libvirt headers/libs должны соответствовать 11.10.0. Cinder использует
  исходный constraints. Другие изменения pins не выполняются молча.
  `prepare_environments` не обновляет существующий prefix и не меняет product requirements.
  Перед вызовом `prepare_environments` подготовить в отдельном новом каталоге
  автономные локальные Git clones закреплённых commits (`git clone --no-hardlinks
  --no-checkout`, затем detached checkout точного SHA), со сверкой tree. Клонирование
  выполняется там, где доступны исходные Git metadata; копии переносятся в Linux
  целиком и передаются как checkout inputs. Это исключает host-only `.git` указатели
  linked worktrees и даёт отдельные `.stestr` каталоги regression. Исходные checkout
  и Git metadata не изменяются; environment prefix создаётся отдельно от этих копий.
  ОС-пакеты/образ задаются README для disposable Rocky Linux 9; точный image digest,
  installed RPMs и pip freeze фиксируются при исполнении. При import `storage_client`
  допустим только прежний явно маркированный import-only sentinel с ошибкой при любом
  использовании; его срабатывание в L1 — failure. Не создавать фиктивный driver SDK.
  Скрипт сохраняет `L1_ENVIRONMENT` JSON с путями interpreters/checkouts и отдельный
  env-файл с `L1_NOVA_PYTHON`, `L1_CINDER_PYTHON`, `L1_NOVA_CHECKOUT`,
  `L1_CINDER_CHECKOUT`; shell значения экранируются штатным `shlex.quote`.
- [ ] **Step 5: GREEN.** Повторить три unit modules; затем реальная подготовка новых
  environments и preflight обоих interpreters с импортом будущих required product
  modules (`nova.checkpoints`, `nova.virt.libvirt.checkpoints.executor`, `cinder.checkpoints`).
  Недоступные downloads записать как environment blocker, не обходить constraints.
- [ ] **Step 6: Commit.**
  `git add checkpoint_lab tests tools/prepare_environments.py environments README.md`;
  `git commit -m "test: add isolated checkpoint lab process lifecycle"`.

### Task 3: Cinder HTTP со штатными continuation и постоянными БД

**Files:** Create `checkpoint_lab/cinder/{__init__,bootstrap,driver,auth,entry}.py`,
`tests/integration/test_cinder_process.py`, `tests/unit/test_cinder_scope.py`;
Modify `tests/fixtures.py`.

**Interfaces:** Consumes ModelClient, RunManifest, ProcessSupervisor. Produces
`bootstrap_cinder(manifest: RunManifest, *, mode: str) -> None`,
`LabCheckpointDriver(model: ModelClient, manifest: RunManifest)` (Cinder VolumeDriver),
`serve_cinder(manifest: RunManifest, model_endpoint: str, launch_id: str) -> None`.
Entry argv: `--manifest PATH --model-endpoint URL --mode bootstrap|resume --launch-id UUID`.
`observe` возвращает safe operation/step/grant/attachment projections и schema version;
`recover` запускает штатный reconcile, не переписывает строки.

- [ ] **Step 1: Написать failing integration tests.**
  `test_bootstrap_seeds_two_attached_volumes`: UUID совпадают с manifest, устройства
  `/dev/vda` и `/dev/vdb`, два worker hosts, разные bytes. `test_resume_keeps_cinder_rows`:
  после принятого reserve SIGKILL/new interpreter сохраняет operation/step/grant IDs.

```python
def test_incomplete_bootstrap_cannot_resume(self):
    self.interrupt_bootstrap_after_migrations()
    result = self.start_cinder(mode='resume')
    self.assertEqual(2, result['exit_code'])
    self.assertEqual(self.rows_before, self.read_cinder_rows())
```

  Helpers реализуются в tests/fixtures этой задачи через supervisor и read-only SQL.
  Добавить `test_expiry_survives_restart`, `test_effect_scope_is_local_to_worker`,
  `test_unauthenticated_and_foreign_service_denied`.
- [ ] **Step 2: RED.** Из новой Linux-среды:
  `python3.11 -m unittest tests.integration.test_cinder_process -v`.
  Fixtures получают interpreters/checkouts из `L1_ENVIRONMENT` JSON, созданного Task 2;
  отсутствие этого input — явная ошибка, не skip.
- [ ] **Step 3: Реализовать bootstrap/resume.** Использовать `cinder.db.migration.db_sync()`
  только bootstrap; открыть `cinder.sqlite`, включить synchronous FULL/WAL и зарегистрировать
  OVO/config до API/workers. Завершённый seed marker писать последним после проверки
  schema/IDs; незавершённый bootstrap требует нового case directory. В resume требовать
  наличие schema/head и всех исходных IDs, не создавать/заменять строки.
- [ ] **Step 4: Реализовать реальный middleware/worker путь.** Использовать router,
  AuthProtocol, CinderKeystoneContext и C1.1 continuation managers как в существующей
  fixture, без наследования её TestCase. Fake oslo transport/serializer оставить
  внутри группы; token-validation adapter читает persistent auth_state модели.
  LabCheckpointDriver возвращает явно test-only capabilities и передаёт native effects
  модели. Export/connection information корректны для volume/attachment; методы
  обычных driver hooks не заменяются безусловным success.
  Attachment scope берётся из аргументов test-wrapper `execute_attachment`, хранится
  в greenlet-local контексте и очищается в finally; wrappers вызывают оригинал ровно
  один раз. Model key не извлекается из произвольного глобального «текущего шага».
- [ ] **Step 5: GREEN.** Повторить integration module и scope unit tests; доказать новый
  PID/launch UUID и отсутствие DDL/seed в resume. API continuation от чужой identity
  отвергается настоящими middleware/policy/DB checks.
- [ ] **Step 6: Commit.**
  `git add checkpoint_lab/cinder tests`;
  `git commit -m "test: run persistent Cinder continuation service in lab"`.

### Task 4: Nova API и единый двухтомный lifecycle

**Files:** Create `checkpoint_lab/nova/{__init__,bootstrap,runtime,adapters,entry}.py`,
`checkpoint_lab/observe.py`, `oracle.py`, `tests/integration/test_lifecycle.py`,
`tests/integration/test_nova_resume.py`, `tests/unit/test_oracle.py`;
Modify `tests/fixtures.py`.

**Interfaces:** Consumes Task 1–3. Produces
`bootstrap_nova(manifest: RunManifest, *, mode: str) -> None`,
`serve_nova(manifest: RunManifest, model_endpoint: str, cinder_endpoint: str, launch_id: str) -> None`,
`observe_case(manifest: RunManifest, endpoints: Json) -> Json`,
`assert_point(observation: Json, expected: Json) -> None`.
Nova entry добавляет `--cinder-endpoint URL` к argv Cinder-style entry.
`ModelRuntime` реализует наблюдаемые методы host/guest из N2 test Backend,
но не владеет Cinder operations и не заменяет CheckpointClient.
Его методы: `get_guest(instance)`, `get_xml_desc(**kwargs)`,
`checkpoint_state() -> Json`, `checkpoint_block_job(name: str) -> Json`,
`checkpoint_set_marker(xml: str) -> None`, `checkpoint_save(path: str, xml: str) -> None`,
`checkpoint_stop() -> None`, `checkpoint_saved_xml(path: str) -> str`,
`checkpoint_restore(path: str, xml: str) -> None`, `resume() -> None`.

- [ ] **Step 1: Написать failing L1-01 и oracle tests.**

```python
def test_l1_01_full_http_lifecycle(self):
    a = self.nova_create('A')
    self.guest_write(b'B:ram', (b'B:disk-1', b'B:disk-2'))
    b = self.nova_create('B')
    for point in (a, b, a):
        self.nova_revert(point)
        assert_point(self.observe(), self.expected[point])
    self.nova_delete(b)
    self.assert_point_a_unchanged(a)
    self.nova_delete(a)
    self.assert_all_owned_resources_released()
```

  `nova_create/revert/delete` — HTTP helpers `tests/fixtures.py`, ждут terminal
  operation, не только 202; expected задаётся до операций. `test_swapped_disks_fail`
  меняет значения vda/vdb в synthetic observation и требует AssertionError.
  Добавить `test_default_sql_loaders_are_used`, `test_new_nova_process_reads_same_rows`,
  `test_incomplete_bootstrap_cannot_resume`, `test_no_user_token_in_durable_journal`.
- [ ] **Step 2: RED.**
  `python3.11 -m unittest tests.integration.test_lifecycle tests.integration.test_nova_resume tests.unit.test_oracle -v`.
- [ ] **Step 3: Реализовать Nova bootstrap.** `nova.db.migration.db_sync(database='api')`
  и `db_sync(database='main', context=...)` для cell0/cell1 в bootstrap-only mode;
  actual CellMapping/InstanceMapping, two BDMs с исходными Cinder attachment IDs,
  flavor и fresh network cache/port identities, Service versions закреплённого N2.
  Не использовать database copying или fixture cleanup как resume. Seed complete
  marker фиксируется только после сверки Nova и Cinder identities.
- [ ] **Step 4: Собрать runtime и HTTP путь.** Реальные N2 Runtime/StagingStore/ArtifactStore/
  Topology/AttachmentCoordinator/LibvirtCheckpointExecutor; filesystem observations,
  quota helpers, VIF/Neutron и libvirt/os-brick boundaries test-only. ModelRuntime
  работает через ModelClient, создаёт synthetic save file по выданному pinned path,
  сохраняет XML/markers и PAUSED → отдельный unpause. `CheckpointClient` получает
  session/auth factory, но оставляет штатные loaders; instance/operation contexts
  таргетируются в нужную cell. API/conductor/compute используют настоящие OVO/RPC
  serializers. Nova auth — явно тестовый RequestContext adapter; Cinder auth middleware
  остаётся настоящим. Recovery context не получает сохранённый user token.
- [ ] **Step 5: GREEN и independent oracle.** Проверить все три значения A/B,
  volume/LUN-model/port/server identity, неизменность A после delete B, guards/quotas
  после последнего delete, effect invocation counts и HTTP trace. Model observation
  читается только oracle; production runtime не получает дополнительные proofs.
- [ ] **Step 6: Commit.**
  `git add checkpoint_lab/nova checkpoint_lab/observe.py checkpoint_lab/oracle.py tests`;
  `git commit -m "test: exercise Nova checkpoints through real Cinder HTTP"`.

### Task 5: Детерминированные barriers и остановки процессов

**Files:** Create `checkpoint_lab/faults.py`, `proxy.py`,
`tests/unit/test_faults.py`, `test_proxy.py`, `tests/integration/test_process_faults.py`;
Modify `checkpoint_lab/{processes.py,model/server.py,cinder/driver.py,cinder/entry.py,nova/adapters.py}`.

**Interfaces:** `FaultSpec(case_id: str, target: str, boundary: str, action: str, resource_id: str | None)`
в faults.py; `BarrierController.arm(fault: FaultSpec) -> str`,
`wait(barrier_id: str, deadline: float) -> Json`, `release(barrier_id: str) -> None`;
`FaultProxy.start(upstream: str, manifest: RunManifest) -> str`,
`drop_once(method: str, path: str, *, after_upstream: bool) -> str`, `observe() -> Json`.
Proxy клиентский URL передаётся Nova вместо прямого Cinder URL; redirects/retry отсутствуют.
`BarrierController.start() -> str` открывает loopback control server supervisor;
`FaultClient(endpoint: str).hit(boundary: str, key: EffectKey) -> None` сообщает
достижение границы и ждёт release в пределах deadline. Новые child argv принимают
`--fault-endpoint URL`; без него fault hooks выключены. Сервер управляющего
канала обслуживает наблюдение/release независимо от заблокированного effect
запроса; ожидание barrier никогда не удерживает SQLite write transaction.

- [ ] **Step 1: Failing tests barriers/response loss.**
  `test_barrier_is_one_shot_and_scoped`, `test_deadline_without_ack_is_incomplete`,
  `test_drop_without_upstream_proof_is_incomplete`, `test_scope_does_not_leak_between_greenlets`.
  Post-effect assertion:

```python
def test_cinder_crash_after_model_commit_never_redispatches(self):
    self.reach('cinder_after_effect_before_reply', action='snapshot')
    before = self.model_counts()
    self.kill_and_resume('cinder_group')
    self.recover_to_expected_terminal()
    self.assertEqual(before['execute_attempts'], self.model_counts()['execute_attempts'])
    self.assert_new_launch('cinder_group')
```

  Integration fixtures реализуют reach/kill_and_resume через BarrierController/
  ProcessSupervisor, не sleep. Параметры restore/unpause используют сохранённый artifact.
- [ ] **Step 2: RED.**
  `python3.11 -m unittest tests.unit.test_faults tests.unit.test_proxy tests.integration.test_process_faults -v`.
- [ ] **Step 3: Реализовать точные boundaries.**
  `cinder_claimed_before_effect` — вход test driver после штатного claim до ModelClient.execute;
  `cinder_after_effect_before_reply` — model commit уже подтверждён, driver ещё не вернулся;
  `nova_runtime_effect_before_receipt` — host adapter завершил effect, Runtime ещё не получил ответ;
  `nova_receipt_before_observation` — test-wrapper `staging.RuntimeRecord.finish(result)`
  сперва вызывает оригинал, затем ждёт barrier до возврата executor/coordinator.
  `attachment_before_result` — model connection effect закончен, до Cinder DB intent success;
  `attachment_result_reply_lost` — upstream HTTP ответ получен proxy после durable child result.
  Wrappers только наблюдают/задерживают; не фальсифицируют return, journal или receipt.
- [ ] **Step 4: GREEN для L1-04/05/06/07.** `os.kill` не вызывается по произвольному PID;
  supervisor завершает собственный process handle через kill и wait. После restart
  штата query подтверждает только доступные ему proofs. Pre-receipt runtime и
  отсутствующий Cinder effect record остаются unknown; нет следующего unsafe child.
  Model component crash из Task 1 сохраняется отдельным тестом, не заменяет эти cases.
- [ ] **Step 5: Commit.**
  `git add checkpoint_lab tests`;
  `git commit -m "test: add checkpoint process crash and lost reply barriers"`.

### Task 6: Полная матрица сценариев и независимые assertions

**Files:** Create `checkpoint_lab/cases.py`, `tests/unit/test_case_inventory.py`,
`tests/integration/test_fault_matrix.py`; Modify `checkpoint_lab/{observe.py,oracle.py}`,
`tests/fixtures.py`.

**Interfaces:** `Case(id: str, spec_group: str, fault: FaultSpec | None, expected: Json)`;
`all_cases() -> tuple[Case, ...]`, `run_case(case: Case, manifest: RunManifest, supervisor: ProcessSupervisor) -> Json`.
`assert_case(case: Case, before: Json, after: Json, trace: list[Json]) -> None`
принадлежит oracle. Каждая case выбирает только собственный каталог; expected data
не читается из production response/model completion record.

- [ ] **Step 1: Зафиксировать failing inventory test и assertions отсутствующих cases.**
  Полная запланированная матрица — **24 cases**, это не число уже пройденных тестов:

```text
L1-01
L1-02
L1-03
L1-04
L1-05.snapshot
L1-05.revert
L1-05.delete_snapshot
L1-06.save
L1-06.restore_paused
L1-06.unpause
L1-07.save
L1-07.restore_paused
L1-07.unpause
L1-08
L1-09.create_result_lost
L1-09.update_result_lost
L1-09.update_before_result
L1-10.expired_user
L1-10.foreign_service
L1-10.foreign_project
L1-11.cleanup_pending
L1-11.live_writer
L1-12.exact_replay
L1-12.changed_payload
```

```python
def test_inventory_is_exact_and_nonempty(self):
    self.assertEqual(self.expected_ids, tuple(case.id for case in all_cases()))
    self.assertEqual(24, len(set(self.expected_ids)))
```

- [ ] **Step 2: RED.** `python3.11 -m unittest tests.unit.test_case_inventory tests.integration.test_fault_matrix -v`;
  сохранить конкретные отсутствующие/неудовлетворённые cases, не выдавать их за skips.
- [ ] **Step 3: Реализовать L1-02/03/09/10/12.** Accepted reserve loss подтверждается
  upstream 202 и Cinder operation; absent reserve — proxy обрывает до upstream,
  фоновый recovery не отправляет reserve, exact user API replay сохраняет original IDs.
  Expiry/revoke state находится в model DB и переживает restart; service auth свежий.
  Attachment create-after-result не создаёт второй attachment; update-after-result
  не повторяет export/initialize_connection. Только update-after-effect-before-result
  требует unknown; SQL-only create до commit не объявляется неизвестным native effect.
  Чужой project/service и changed payload отвергаются до model mutation.
- [ ] **Step 4: Реализовать L1-08/11.** В partial revert данные первого тома A, второго B;
  domain SHUTOFF/PAUSED, reconnect/unpause counters не растут, locks обоих volumes
  и соответствующие reservation сохраняются. Cleanup pending задерживает настоящее
  завершение файлового delete; live writer задерживает save callback до receipt.
  В варианте без kill release barrier позволяет исходному writer закончить и
  штатному workflow закрыть cleanup; до proof quota не возвращается, после — один раз.
  Не записывать completion/status из fault controller.
- [ ] **Step 5: GREEN и отрицательные контроли.** Полная матрица; подмена двух disks,
  дополнительный execute, мутирующий query и потеря outcome каждый дают failure
  именно oracle/inventory. Проверять постоянные/временные quotas по owner/phase;
  не требовать бессмысленного `все counters > 0`. Query trace может дописывать
  audit, но payload/effects/completion неизменны.
- [ ] **Step 6: Commit.**
  `git add checkpoint_lab tests`;
  `git commit -m "test: cover checkpoint lifecycle and recovery fault matrix"`.

### Task 7: CLI, отчёт, чистое воспроизведение и локальная поставка

**Files:** Create `checkpoint_lab/{evidence,runner,__main__}.py`,
`tests/unit/test_evidence.py`, `test_cli.py`, `tools/package.py`, `VALIDATION.md`;
Modify `README.md`, `pyproject.toml`, `checkpoint_lab/cases.py` при wiring.

**Interfaces:** `run_suite(options: Json) -> Json` (runner),
`write_report(root: Path, expected_ids: tuple[str, ...], outcomes: list[Json], environment: Json) -> Json`,
`verify_report(root: Path) -> Json` (evidence), `main(argv: list[str] | None = None) -> int`.
Outcomes: id/status=`pass|fail|incomplete`, launch IDs, checks, evidence refs;
нет автоматического pass по exit 0 дочернего процесса.

- [ ] **Step 1: Написать failing CLI/evidence tests.**

```python
def test_subset_is_not_full_acceptance(self):
    report = self.run_selected(('L1-01',))
    self.assertEqual(0, report['exit_code'])
    self.assertFalse(report['suite_complete'])
    self.assertFalse(report['production_qualified'])
    self.assertFalse(report['hardware_qualified'])
```

  Добавить `test_missing_duplicate_or_unexpected_outcome_fails`,
  `test_timeout_preserves_evidence_and_reaps_children`, `test_hash_mismatch_fails`,
  `test_output_path_cannot_overwrite_existing_run`, `test_report_is_not_v1_qualification`.
- [ ] **Step 2: RED.** `python3.11 -m unittest tests.unit.test_cli tests.unit.test_evidence -v`.
- [ ] **Step 3: Реализовать CLI и report.** Commands `preflight`, `run`, `verify-report`;
  flags `--nova-checkout`, `--cinder-checkout`, `--nova-python`, `--cinder-python`,
  `--output`, repeatable `--case`, `--case-timeout` (default 160 seconds),
  `--startup-timeout` (default 60 seconds). Положительные конечные таймауты проверяются.
  Для run требуется новый output dir; supervisor создаёт отдельный manifest/root
  каждого case. Полный expected inventory фиксируется до исполнения. `results.json`
  хранит expected/selected/completed IDs, environment/source hashes, scope и exit code.
  SHA256SUMS исключает только себя; verifier отклоняет missing/changed evidence.
- [ ] **Step 4: GREEN.** Unit/component tests плюс CLI smoke. Запуск полного L1
  в новой Linux-среде по README, затем второй полный запуск в новом output каталоге.
  Пример (значения `/lab/...` — явные пути внутри одноразовой среды, не жёстко
  закодированные значения в runner):

```sh
python3.11 -m checkpoint_lab preflight --nova-checkout /lab/nova --cinder-checkout /lab/cinder --nova-python /lab/env/nova/bin/python --cinder-python /lab/env/cinder/bin/python
python3.11 -m checkpoint_lab run --nova-checkout /lab/nova --cinder-checkout /lab/cinder --nova-python /lab/env/nova/bin/python --cinder-python /lab/env/cinder/bin/python --output /lab/results/run-1
python3.11 -m checkpoint_lab verify-report --output /lab/results/run-1
```

  Expected: exit 0, 24/24 expected cases pass, `suite_complete=true`, оба qualification
  flags false. Это ожидаемый результат будущей проверки, не факт текущего исполнения.
- [ ] **Step 5: Affected regression и baseline boundary.** Сохранить product HEAD/status;
  запустить N2 `nova.tests.functional.test_checkpoint_executor`,
  `nova.tests.functional.test_checkpoint_cinder_contract`,
  `nova.tests.functional.test_checkpoint_store` и Cinder
  `cinder.tests.functional.test_checkpoint_continuation` в их interpreters.
  Из `$L1_NOVA_CHECKOUT` выполнить:
  `"$L1_NOVA_PYTHON" -m stestr --test-path=./nova/tests/functional run --concurrency 1 'nova.tests.functional.test_checkpoint_(executor|cinder_contract|store)'`.
  Из `$L1_CINDER_CHECKOUT` выполнить:
  `"$L1_CINDER_PYTHON" -m unittest cinder.tests.functional.test_checkpoint_continuation -v`.
  Настроить `N2_CINDER_*` на закреплённую dependency и доступные
  Git metadata; это fixture endpoints, не production. Raw outcomes сравнить с
  сохранёнными pinned baseline по test IDs: новых skips/xfails/failures нет.
  Не повторять все 17 тысяч product unit tests без новой причины.
- [ ] **Step 6: Финальная проверка кода и документации.** `python3.11 -m compileall -q checkpoint_lab tests tools`;
  `git diff --check`; проверить README запуском, links, hashes, отсутствие credentials
  и import-only SDK invocation. VALIDATION содержит фактические counts/versions,
  failure history, синтетические boundaries и необследованные runtime gates.
- [ ] **Step 7: Commit и одно независимое review всей серии.**
  `git add checkpoint_lab tests tools pyproject.toml README.md VALIDATION.md`;
  `git commit -m "test: package reproducible checkpoint integration lab"`.
  Review по `superpowers:requesting-code-review` получает spec/plan, diff, пять
  Review Focus, RED/GREEN/evidence и ограничения. Findings проверяются по коду;
  после исправлений повторяются затронутые проверки и финальный report verifier.
- [ ] **Step 8: Сформировать и проверить локальный пакет.** `tools/package.py` создаёт
  новый `deliverables/checkpoint-local-integration-l1/` рядом с существующими
  bundles, включает source archive, commit/tree, spec/plan, README/VALIDATION,
  review/resolution и evidence с SHA256SUMS. Existing destination не перезаписывается.
  Проверить hashes и запуск из распакованного archive с теми же pinned dependencies;
  публикацию/merge/deployment этот план не включает.

## 3. Критерий завершения и передача

Успех объявляется только по свежему полному L1 отчёту, выполненным отрицательным
контролям, воспроизводимому повтору, affected regression без новых исключений,
review resolution и проверенному пакету. RED из-за найденного production defect
сохраняется вместе с минимальным воспроизведением; изменение expectations ради
успеха запрещено. Исправление production code — отдельное предложение с границей
изменения, а не незаметное расширение тестового пакета.

Самопроверка плана: spec §§1–3/11 покрыты scope/pins и финальным gate;
§§4–6 — Tasks 1–4; §7 — Task 5; §8 — Task 6; §§9–10 — Tasks 2/6/7;
§12 — последовательностью зависимостей. Все пять Review Focus назначены тестам.
Ни один пункт execution checklist пока не выполнен.
