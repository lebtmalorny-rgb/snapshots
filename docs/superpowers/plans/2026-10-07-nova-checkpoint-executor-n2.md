# Nova Checkpoint Executor N2 Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking. Сохраняется Native-исполнение с одним финальным review серии; план и Native-исполнение согласованы пользователем.

**Goal:** Реализовать libvirt save/restore, постоянный RAM-store на существующем NFS/NAS и интеграцию с Cinder C1.1 с доказуемым учётом частичных эффектов.

**Architecture:** N1 journal расширяется дочерними intents и contract v2; conductor управляет ownership и квотами, compute исполняет только разрешённые эффекты. RAM сначала записывается в зашифрованный staging, затем публикуется в общем store. Cinder вызывается через отдельный scoped client; неизвестные эффекты только сверяются.

**Tech Stack:** Python 3.11/Linux, Nova fork, libvirt/QEMU KVM, os-brick, Cinder REST 3.73, OVO/RPC, SQLAlchemy/Alembic, shared POSIX filesystem и encrypted XFS staging, stestr.

**Spec:** [Согласованная спецификация N2](../specs/2026-10-07-nova-checkpoint-executor-n2-design.md). Протокол Cinder задан [планом C1.1 §2](2026-10-07-cinder-checkpoint-continuation-c11.md); это обязательный совместный интерфейс, не независимый вариант API.

## Global Constraints

- Nova base `f6ead340524374ea9d287068124c0add89137c6f`; отдельная ветка `feature/nova-checkpoint-executor`, checkout `work/nova-checkpoint-executor`.
- C1.1 завершён и проверен: HEAD `d065a55cbe5cb8cd59489752846a41145d72bd27`, tree `53888d9cea7fa1b12d35a0b9afd894eeee270843`, локальный пакет `deliverables/cinder-checkpoint-continuation-c11`. Эта зависимость обязательна для Task 5. До этого допустимы только явно тестовые contract fixtures; production fallback на C1 3.72 запрещён.
- «Готовые серии C1/N1 не переписываются.» Архивы, `sources/`, готовые deliverables и базовые checkout сохраняются.
- «Первый профиль — `kvm-bios-same-host-v1`: KVM/libvirt, BIOS, восстановление на исходном compute, все записываемые диски принадлежат Cinder.» Исключения устройств — spec §1.
- «generation остаётся 1; takeover отсутствует.» Timeout, `enabled=false`, restart и admin reset не освобождают ownership/квоты.
- «Новые child/temporary-ticket OVO — 1.0»; изменённые checkpoint OVO — 1.1. Nova public API **2.106**, compute RPC **6.6**, compute-task RPC **1.27**, service **70**, общий conductor RPC **3.0**, Instance OVO **2.8**.
- «Никаких SQL transactions через RPC/driver I/O»; compute использует OVO/conductor, не прямой SQL. API DB и cell DB не имеют общей transaction.
- «Service continuation живёт до завершения операции. После release разрешено только чтение tombstone.» User bearer не сохраняется. Cinder actor не подменяется сервисом.
- Shared RAM-store — основной кандидат существующий NFS/NAS; encrypted staging на compute. NAS model/versions/crypto и runtime profile ещё не квалифицированы.
- «Все production adapters по умолчанию unsupported, feature выключена.» Новые positive profiles появляются только при подтверждённых предпосылках; fake profiles доступны только тестам.
- N1 defaults сохраняются: enabled=false, count limits 8/100, artifact bytes=0, reconcile interval=60. Bytes=0 не разрешает новые записи; cleanup уже зарезервированных ресурсов остаётся возможным.
- Ранее разрешённый private SDK skip сохраняется только для `storage_client>=1.3` и `nova.tests.unit.virt.libvirt.volume.test_sbd`; не исключать остальные libvirt tests.
- Разработка локальная; план не разрешает mount, форматирование дисков, выдачу ключей, изменение NAS или операции с действующими ВМ. V1 оформляется отдельным запуском на предоставленном стенде.

## Review Focus

1. NAS размонтирован, а каталог mountpoint остался: ни одного байта в локальный fallback directory; Task 4.
2. Две cells используют одинаковые display names/ошибочно совпавшие handles: чужой artifact нельзя прочитать, заменить или удалить; Task 2/4.
3. Parent query видит завершённый child, но следующий не начат: переход `ready → continue` без повторного эффекта; Task 2/8.
4. Потерян ответ restore/unpause или остался managed-save: не запускать вторую ВМ и не принимать marker за completion proof; Task 6/8.
5. RAM downloaded, temporary quota refunded, поздний writer ещё жив: запрет refund без fence и cleanup proof; Task 3/4/8.

---

## 1. Исполнение, среда и структура

Перед Task 1 проверить изоляцию через `superpowers:using-git-worktrees`. Новый checkout создаётся от указанного N1 SHA; сохранить BASE_COMMIT/BASE_TREE в ledger. Native выполняет Tasks 1–9 последовательно. Независимый reviewer нужен один раз после всей серии; при доработках повторно проверяется затронутая функциональность.

Среда — новый disposable Linux container, Python 3.11, checkout `/work`, исходные constraints OpenStack 2025.1, SQLite/MySQL 8/PostgreSQL 16 без published ports. Восстановить public-only окружения по N1 VALIDATION (`deliverables/nova-checkpoint-model-n1/VALIDATION.md`), включая import-only private SDK sentinel и исходные test extras. Старые N1 containers удалены: не ссылаться на них как на готовые. Production requirements не менять ради локальных зависимостей.

Команды ниже запускаются из `/work` в новой среде с подготовленными `.tox/py311` и `.tox/functional`. `OS_TEST_DBAPI_ADMIN_CONNECTION` должен содержать все три disposable DB URL; записать его тестовые endpoints и версии в evidence.

```sh
export PBR_VERSION=0.0.0
export OS_STDOUT_CAPTURE=1 OS_STDERR_CAPTURE=1 OS_TEST_TIMEOUT=160
export PYTHONDONTWRITEBYTECODE=1 NOVA_RAISE_ON_GREENLET_LEAK=True
.tox/py311/bin/stestr run --concurrency 4 nova.tests.unit.checkpoints
```

Expected baseline: exit 0, неожиданных failures=0. Импортный SDK sentinel должен выбрасывать ошибку при любом SDK вызове; не изображать реализацию SBD. RED — ожидаемый product assertion, не отсутствие зависимостей. Ledger: `.superpowers/sdd/2026-10-07-nova-checkpoint-executor-n2/progress.md` с RED/GREEN, commands, versions, commits и RULINGS.

| Файлы | Граница ответственности |
|---|---|
| `nova/checkpoints/contracts_v2.py`, `workflow_v2.py` | Строгие immutable types, v2 phase order/evidence, без SQL/virt |
| `nova/db/main/checkpoint_effects.py`, `nova/objects/checkpoint_effect.py` | Child claims/ресурсы/продолжение и явные OVO transitions |
| `nova/db/api/checkpoint_temporary.py`, `nova/objects/checkpoint_temporary.py` | Temporary quota tickets и proofs |
| `nova/checkpoints/{api,quota,compute,coordinator,recovery}.py` | Общая orchestration, версия контракта, lifecycle |
| `nova/virt/libvirt/checkpoints/{executor,topology,runtime,store,staging,profiles}.py` | Физические эффекты, runtime/store проверки, чтение profile records |
| `nova/privsep/checkpoint.py` | Узкие проверки mount/dm-crypt/XFS и project-quota операции с фиксированными аргументами |
| `nova/volume/checkpoints.py` | C1.1 клиент; обычный `nova/volume/cinder.py` не переключает все вызовы на 3.73 |
| `nova/checkpoints/attachments.py` | BDM + Cinder + os-brick choreography под child intents |
| `nova/virt/libvirt/driver.py`, `nova/virt/virtapi.py`, `nova/compute/manager.py` | Wiring к существующим helpers, без переноса всей orchestration в driver.py |

## 2. Общие интерфейсы и формат данных

### 2.1. Contracts v2 — Task 1

Immutable dataclasses имеют строгие `to_primitive()/from_primitive()` с `schema_version=2`; v1 код и fixtures продолжают работать по schema 1. `Context` — Nova RequestContext. UUID canonical, hashes SHA-256; bool не является integer, отрицательные размеры и неизвестные keys запрещены.

- `Binding(deployment_id: str, cell_uuid: str, project_id: str, instance_uuid: str, checkpoint_id: str, host: str, node: str, executor_id: str, db_fingerprint: str, runtime_fingerprint: str, runtime_profile_id: str, store_profile_id: str, staging_profile_id: str)`.
- `ArtifactRef(id: str, kind: str, logical_id: str, bytes: int, checksum: str, location: str)`; kind=`ram|runtime|config`, location=`staging|shared`. Pending artifact reservations хранят nullable bytes/checksum отдельно и не выдаются за завершённый ArtifactRef.
- `ProbeV2(supported: bool, reasons: tuple[str,...], executor_id: str, executor_profile: str, fingerprint: str, device_manifest: dict, artifact_upper_bound: int, runtime_fingerprint: str, runtime_profile_id: str, store_profile_id: str, staging_profile_id: str, save_bytes_limit: int, restore_bytes_limit: int, contract_version: int=2)`.
- `ChildIntent(id: str, parent_step_id: str, operation_id: str, generation: int, action: str, resource_id: str, executor_id: str, payload_hash: str, payload: dict)`.
- `EffectResult(status: str, evidence: dict, error_code: str|None)`; statuses=`pending|succeeded|failed_no_effect|failed_with_effects|unknown|ready`. Physical child validation принимает свои action-specific evidence (например PAUSED для restore_paused); parent restore success отдельно требует RUNNING после unpause.
- `Continuation(step_id: str, journal_revision: int, next_child_id: str)`; `ready` evidence ровно эти поля. `next_child_id` выдан cell DB; исполнитель не выбирает новый resource произвольно.
- `TemporaryProof(ticket_id: str, operation_id: str, cell_uuid: str, version: int, kind: str, closed_child_ids: tuple[str,...])`; kind=`cancelled_before_write|staging_deleted`.

`StepIntentV2` сохраняет поля v1 StepIntent, но payload содержит binding, operation kind, generation=1, budgets/version, references на дочерний journal и prior phase evidence. Не пересылать XML/connection_info/user bearer. `validate_result(phase: str, value: dict) -> EffectResult` проверяет форму, DB transition дополнительно связывает значения с durable rows.

`AdmissionV2(key: RequestKey, operation_id: str, checkpoint_id: str, cell_uuid: str, host: str, node: str, fingerprint: str, device_manifest: dict, artifact_upper_bound: int, executor_id: str, name: str, runtime_fingerprint: str, runtime_profile_id: str, store_profile_id: str, staging_profile_id: str, save_bytes_limit: int, restore_bytes_limit: int)` использует неизменённый RequestKey v1; outer schema_version=2. Task 1 определяет тип, Task 2/3 добавляют version-aware admission в cell/API DB и quota ledger. Decode v1/v2 по явному discriminator; не передавать новый тип старому `Admission.from_primitive` и не менять смысл сохранённых v1 tickets.

`failed_no_effect`: точно `no_effect=true,late_effect_impossible=true`. `failed_with_effects`: `resource_ids`, `terminal_child_ids`, `late_effect_impossible=true`; весь набор дочерних intent должен быть разрешён/зафенсен. `unknown/pending` не содержит выдуманной success-evidence. Поля `ready` допустимы только для parent, не для физического child.

### 2.2. Порядок v2 фаз

```python
CREATE = ('preflight', 'cinder_reserve', 'runtime_save', 'disk_snapshot',
          'artifact_publish_shared', 'runtime_restore_current',
          'staging_cleanup', 'cinder_release')
REVERT = ('preflight', 'cinder_reserve', 'artifact_stage', 'runtime_stop',
          'volume_disconnect', 'disk_revert', 'volume_reconnect',
          'runtime_restore_checkpoint', 'staging_cleanup', 'cinder_release')
DELETE = ('cinder_reserve', 'disk_snapshot_delete', 'artifact_delete',
          'cinder_release')
```

Compensation create: сначала восстановить доказанно сохранённый current runtime, затем Cinder cleanup созданных snapshot-остатков, удалить shared/staging artifacts и release abort. Порядок определяется подтверждёнными children; partial/unknown save не разрешает автоматический restore. Revert допускает отмену только до dispatch runtime_stop/disk/attachment effects; после этого — continuation или recovery_required, не возврат к прежней RAM. Query может записывать наблюдения в journal, но не выполнять ресурсные effects.

### 2.3. Profiles/config — Task 1 и Task 4/6

Новые `[checkpoint]` options: `deployment_id=''`, `store_root=''`, `staging_root=''`, `profile_file=''`, `max_concurrent_transfers=1` (min=1). Первые четыре обязательны для supported executor; feature defaults N1 сохраняются. Credentials не добавлять в profile JSON.

`profiles.load(path: str) -> ProfileSet`; `ProfileSet.match(observed: dict) -> ProfileMatch|None`. `ProfileMatch(id: str, kind: str, identity_sha256: str, qualification_sha256: str, observed: dict, limits: dict)` и `ProfileSet(records: tuple[ProfileMatch,...])` — frozen dataclasses; kind=`runtime|store|staging`. JSON schema version 1 содержит записи этих типов с UUID id, canonical identity SHA, qualification report SHA, exact observed tuple и положительными limits. Tuple runtime: architecture, kernel/libvirt/QEMU/os-brick versions, machine type, CPU/features, device allowlist и disk cache/format; store: export identity, server/profile identity, client mount identity, durable publish/transport/at-rest capability; staging: encrypted device identity, filesystem, quota enforcement и writer identity. `save_bytes_limit`, `restore_bytes_limit`, `peak_create_bytes` задаются квалифицированным профилем для ограниченного VM shape; memory/device values вне этого shape unsupported.

Config file — root-controlled allowlist квалифицированных профилей, не доказательство, полученное от пользователя ВМ. Реальная readiness сверяется с observation; одного `qualified=true` не существует. Sample config пустой, поддерживаемые production profiles в патчи не выдумываются. Temporary limits и фактические allocated/logical bytes должны укладываться в API DB reservation; формула `flavor.memory_mb` не заменяет writer bound.

## Task 1: Contracts, версии и реальные предпосылки probe

**Files:** Create `nova/checkpoints/{contracts_v2,workflow_v2}.py`, `nova/virt/libvirt/checkpoints/{__init__,profiles}.py`, `nova/tests/unit/checkpoints/test_contracts_v2.py`; Modify `nova/conf/checkpoint.py`, `nova/objects/service.py`, `nova/compute/rpcapi.py`, `nova/conductor/{api,manager,rpcapi}.py`, `nova/virt/driver.py`.

**Interfaces:** §2.1–2.3; `workflow_v2.steps_for(kind: str) -> tuple[str,...]`; new driver hooks `checkpoint_probe_v2(context, instance, manifest: dict, purpose: str, checkpoint_id: str|None=None, operation_id: str|None=None) -> dict`, `checkpoint_execute_v2(context, instance, intent: dict) -> dict`, `checkpoint_query_v2(context, instance, intent: dict) -> dict`. purpose=`create|revert|delete|recover`. Compute RPC probe зеркалит probe signature на version 6.6; conductor compute-task probe `(context, instance_uuid: str, purpose: str, checkpoint_id: str|None=None, operation_id: str|None=None) -> dict` на 1.27 сначала проверяет project/cell/resource binding. Checkpoint ID нужен для preflight revert/delete до появления новой operation; operation ID — для recovery принятой операции. Пока hooks возвращают unsupported/unknown; default поддержка не включается.

- [x] **Step 1 — RED.** В test module обычные Nova test.TestCase и literal primitives; v1 regression запускается вместе с новым.

```python
def test_generation_bool_and_unknown_ready_fields_rejected(self):
    value = dict(schema_version=2, status='ready', error_code=None,
                 evidence=dict(step_id=uuids.step, journal_revision=1,
                               next_child_id=uuids.child, force=True))
    self.assertEqual('unknown', v2.validate_result('runtime_save', value).status)
    self.assertRaises(ValueError, v2.validate_generation, True)

def test_phase_order_keeps_staging_until_runtime_restored(self):
    phases = workflow_v2.steps_for('create')
    self.assertLess(phases.index('runtime_restore_current'),
                    phases.index('staging_cleanup'))
    self.assertLess(phases.index('staging_cleanup'),
                    phases.index('cinder_release'))
```

`validate_generation(value: int) -> None` принадлежит contracts_v2. Дополнить tests: unsupported probe не содержит positive identity; fingerprint/runtime hash разделены; старые service=69/RPC=6.5 отказывают до effects; v1 serialize roundtrip unchanged; default config не включает executor.

- [x] **Step 2 — Run RED.** `.tox/py311/bin/stestr run nova.tests.unit.checkpoints.test_contracts_v2`.
- [x] **Step 3 — Implement.** Exact versions из Global Constraints, service history 70. Новые RPC методы не скрывать за старым cap. API 2.106 response schema не меняется; внутренний probe выбирается по service support. Добавить schemas/reasons `store_unavailable`, `profile_unverified`, `authorization_required`, `artifact_corrupt` в v2 safe list; публичный API отображает стабильные существующие категории `executor_unavailable/unsupported_profile/recovery_required`, без raw diagnostics.
- [x] **Step 4 — GREEN.** Новые tests + `nova.tests.unit.checkpoints.test_contracts`, `test_rpc`; expected exit 0.
- [x] **Step 5 — Commit:** `feat: define checkpoint executor v2 contracts`.

## Task 2: Дочерние intents, partial resources и continuation CAS

**Files:** Create `nova/db/main/checkpoint_effects.py`, `nova/objects/checkpoint_effect.py`, migration `nova/db/main/migrations/versions/5d9b2ea784c1_add_checkpoint_effects.py`, `nova/tests/functional/db/test_checkpoint_effects.py`; Modify `nova/db/main/{models,checkpoints}.py`, `nova/objects/{checkpoint,__init__}.py`, `nova/tests/unit/objects/test_objects.py`.

**Interfaces:** OVO `CheckpointEffect` 1.0 methods:
`prepare(context, parent_step_id: str, action: str, resource_id: str, payload: dict) -> dict`,
`claim(context, child_id: str, generation: int, executor_id: str, expected_revision: int) -> dict|None`,
`observe(context, child_id: str, generation: int, result: dict) -> dict`,
`get_children(context, parent_step_id: str) -> list[dict]`,
`continuation(context, parent_step_id: str) -> dict|None`.
`CheckpointArtifactReservation` 1.0 methods `reserve(context, child_id: str, artifact_id: str, location: str, upper_bound: int) -> dict`, `confirm(context, artifact_id: str, result: dict) -> dict`, `deleted(context, artifact_id: str, child_id: str) -> dict`.

Новые cell tables: `instance_checkpoint_effects` (child UUID PK, parent FK, operation FK, generation, action, resource_id, payload/hash, executor, status/result, revision), UNIQUE `(parent_step_id,action,resource_id)`; `instance_checkpoint_artifact_reservations` (artifact UUID PK, child FK, namespace hash, location, upper_bound, actual/checksum nullable, writer state, cleanup child ID). Existing operation/checkpoint получают contract_version и binding/runtime_fingerprint/profile IDs additive, OVO 1.1. Migration parent — N1 `4a8c2d7e9101`.

Lock order: Instance → guard → checkpoint → operation → parent → children/resources по UUID. `ready` не завершает parent: только полный success всех required children разрешает parent success. Новая revision меняется при prepare/claim/observe; stale continuation не dispatch. Generic save отсутствует. Release/close-proof учитывает даже reservations с failed/unknown save.

При первоначальном claim v2 parent полный допустимый child plan создаётся **в той же cell transaction**. Child IDs и ссылки на результаты предшественников известны до первого I/O; plan закрывается hash/count. Это не позволяет query создавать отсутствующего «следующего child» и не оставляет наполовину подготовленный graph после crash. `CheckpointEffect.prepare` предназначен для этого writer path и не добавляет произвольные nodes после закрытия плана. Child payload ссылается на immutable predecessor result, а не содержит заранее выдуманный attachment/snapshot UUID.

- [x] **Step 1 — RED.** Fixture `EffectDBTest` использует существующие `test_cell_db.make_instance/admission`, преобразует admission в v2 через Task 1 validator и создаёт parent под quota-ready owner. Task-local helpers `prepare_child(action, resource_id)`, `finish_child(child, result)` оборачивают OVO выше.

```python
def test_concurrent_claim_has_one_dispatch(self):
    child = self.prepare_child('save', uuids.ram)
    first = objects.CheckpointEffect.claim(self.ctx, child['id'], 1,
                                          self.executor_id, child['revision'])
    second = objects.CheckpointEffect.claim(self.ctx, child['id'], 1,
                                           self.executor_id, child['revision'])
    self.assertIsNotNone(first)
    self.assertIsNone(second)

def test_partial_save_reservation_prevents_close_proof(self):
    child = self.prepare_child('save', uuids.ram)
    objects.CheckpointArtifactReservation.reserve(
        self.ctx, child['id'], uuids.ram, 'staging', 4096)
    self.finish_child(child, dict(schema_version=2, status='unknown',
                                 evidence={}, error_code='recovery_required'))
    self.assertIsNone(self.close_proof())
```

`close_proof()` вызывает существующий Operation.close_proof для create operation. Дополнить parallel-session claim race на MySQL/PostgreSQL, foreign cell/instance artifact collision, malformed partial list, cleanup before last writer fenced, `ready` со stale revision и downgrade v2 OVO→1.0 reject.

- [x] **Step 2 — Run RED.** `.tox/functional/bin/stestr --test-path=./nova/tests/functional run nova.tests.functional.db.test_checkpoint_effects`.
- [x] **Step 3 — Implement.** Persistence child/resource до I/O; отдельная отмена ещё не dispatched child блокирует late claim. Не переписывать v1 terminal evidence. Query-only observe не создаёт новый child. Scope проверяется после remoting в cell DB.
- [x] **Step 4 — GREEN.** Новые DB tests, N1 `test_checkpoints`, OVO tests и `nova.tests.unit.db.main.test_migrations`; три СУБД, expected exit 0.
- [x] **Step 5 — Commit:** `feat: journal checkpoint child effects and partial resources`.

## Task 3: Временные quota tickets и учёт двух копий

**Files:** Create `nova/db/api/checkpoint_temporary.py`, `nova/objects/checkpoint_temporary.py`, migration `nova/db/api/migrations/versions/82e7b3c951d0_add_checkpoint_temporary_quota.py`, `nova/tests/functional/db/test_checkpoint_temporary_quota.py`; Modify `nova/checkpoints/quota.py`, `nova/db/api/{models,checkpoints}.py`, cell effect/operation proof functions, objects exports/hashes.

**Interfaces:** `QuotaLedger.reserve_temporary(context, operation_id: str, cell_uuid: str, total_bytes: int) -> CheckpointTemporaryTicket`; `.confirm_temporary(context, ticket_id: str) -> CheckpointTemporaryTicket`; `.release_temporary(context, ticket_id: str) -> CheckpointTemporaryTicket`; `.reconcile_temporary(context, limit: int=100, marker: str|None=None) -> dict`. Cell OVO `temporary_close_proof(context, operation_id: str, ticket_id: str) -> dict|None`, `cancel_temporary(context, operation_id: str, ticket_id: str) -> dict`.

Table ticket: UUID PK, operation UUID UNIQUE, immutable cell/project/instance/checkpoint, positive bytes/version, state=`held|confirmed|released`, bound payload hash и closing proof. Counts не увеличиваются; bytes включаются в ту же locked project usage, что N1 permanent tickets. Migration parent `71be90d2c402`. Mapping identity сверяется под той же API DB блокировкой, что N1. Confirm и cancellation — двухбазовый протокол с durable tombstone; никакой timeout refund.

Для create peak покрывается исходным ticket. V2 settle proof допустим только после shared publication, успешного runtime restore, staging cleanup и отсутствия известных незавершённых writers. Для revert temporary ticket покрывает staging/служебные записи до первого download. Cleanup использует уже занятый budget; новое увеличение требует подтверждения до записи.

- [x] **Step 1 — RED.** Fixture на двух cells по `CheckpointTestCase`; helpers `reserve_temp(operation, bytes)`, `mark_writer_unknown(operation)` вызывают новые ledger/child методы, `usage_bytes()` читает API DB usage.

```python
def test_permanent_and_restore_copy_share_project_limit(self):
    self.flags(max_artifact_bytes_per_project=8192, group='checkpoint')
    self.set_permanent_usage(4096)
    first = self.reserve_temp(self.revert_a, 4096)
    self.assertEqual(8192, self.usage_bytes())
    self.assertRaises(exception.CheckpointQuotaExceeded,
                      self.reserve_temp, self.revert_b, 1)
    self.assertEqual(8192, self.usage_bytes())

def test_unknown_download_cannot_refund(self):
    ticket = self.reserve_temp(self.revert_a, 4096)
    self.mark_writer_unknown(self.revert_a)
    self.assertRaises(exception.CheckpointConflict,
                      self.ledger.release_temporary, self.ctx, ticket.id)
    self.assertEqual('confirmed', self.ledger.get_temporary(
        self.ctx, ticket.id).state)
```

`QuotaLedger.get_temporary(context,ticket_id)->CheckpointTemporaryTicket` — read counterpart; fixture `set_permanent_usage` создаёт реальный settled checkpoint, не правит usage обходом ledger. Также test cancelled ticket rejects late download, lost confirm, cell down, exact replay и concurrent reservations across cells на MySQL/PostgreSQL.

- [x] **Step 2 — Run RED.** `.tox/functional/bin/stestr --test-path=./nova/tests/functional run nova.tests.functional.db.test_checkpoint_temporary_quota`.
- [x] **Step 3 — Implement.** Bounds/version CAS, permanent/temporary close proofs разделены. Temporary release не снимает permanent checkpoint count и topology guard. Нельзя уменьшить charged bytes при живом local writer или partial shared file.
- [x] **Step 4 — GREEN.** Новые tests + N1 quota DB/ledger, API DB migration/model sync/OVO; expected exit 0 на трёх СУБД.
- [x] **Step 5 — Commit:** `feat: account checkpoint staging with temporary quota tickets`.

## Task 4: Shared store и encrypted bounded staging

**Files:** Create `nova/virt/libvirt/checkpoints/{store,staging}.py`, `nova/privsep/checkpoint.py`, `nova/tests/unit/virt/libvirt/checkpoints/{__init__,test_store,test_staging}.py`, `nova/tests/functional/test_checkpoint_store.py`; Modify profiles module, conf checkpoint и privsep exports только по существующему шаблону.

**Interfaces:**
`ArtifactStore(root: str, profile: ProfileMatch)` с `.probe() -> dict`, `.publish(context, binding: Binding, child: ChildIntent, source: ArtifactRef, budget: int) -> EffectResult`, `.query(context, binding, child) -> EffectResult`, `.download(context, binding, child, artifact: ArtifactRef, workspace: dict) -> EffectResult`, `.delete(context, binding, child, artifacts: tuple[ArtifactRef,...]) -> EffectResult`.
`StagingStore(root: str, profile: ProfileMatch)` с `.reserve(context, binding, child, limit_bytes: int) -> dict`, `.observe(context, binding, child) -> EffectResult`, `.cleanup(context, binding, child) -> EffectResult`.
Workspace dict — path только внутри executor, mount/device identity, quota project ID, budget/version, namespace hash; paths не уходят в journal/public APIs.

`privsep.checkpoint.inspect_mount(path: str) -> dict`, `inspect_staging(path: str) -> dict`, `set_project_limit(root: str, workspace_id: str, project_id: int, bytes_limit: int) -> dict`. Fixed root allowlist; нет shell-команд или путей от API пользователя. Quota project IDs выделяются атомарным local registry с binding к cell artifact reservation; не переиспользуются до cleanup/fence, restart сверяет обе стороны.

Shared publication использует уникальный temp file и атомарный no-replace publish. Предпочтительный POSIX алгоритм: fsync temp → `link(temp, final)` без follow_symlinks → fsync parent → unlink temp → fsync parent. Два hardlinks допустимы только внутри этого конкретного intent и известного inode; неожиданные links отвергаются. Профиль без поддержки/доказанной долговечности этого алгоритма unsupported. Journal success только после полного завершения; partial link/unlink восстанавливается наблюдением владельца/inode/hash, не заменой existing file.

- [x] **Step 1 — RED.** Tests используют TemporaryDirectory, фиксированные binding/UUID, injected observer/quota helper. `store_for(root, observed_profile)` — fixture constructor этих классов; fake profile разрешён только dependency injection теста.

```python
def test_lost_mount_never_falls_back_to_local_directory(self):
    self.observer.mount_identity = 'unexpected-local-fs'
    result = self.store.publish(self.ctx, self.binding, self.child,
                                self.source, 8192)
    self.assertNotEqual('succeeded', result.status)
    self.assertEqual([], list(self.shared_root.iterdir()))

def test_existing_foreign_object_is_never_replaced(self):
    self.place_foreign_final(b'foreign')
    result = self.publish_own(b'ram-state')
    self.assertNotEqual('succeeded', result.status)
    self.assertEqual(b'foreign', self.read_final())
```

Fixture functions build exact namespace using deployment/cell/hash(project)/instance/checkpoint/artifact IDs; `publish_own` reserves a real child through fixture journal before invoking store. Добавить symlink/traversal, alias across cells, partial copy, checksum mismatch, changed inode, fsync failure, duplicate publish, actual allocated vs logical bytes, limit overflow и late writer before cleanup tests.

- [x] **Step 2 — Run RED.** `.tox/py311/bin/stestr run nova.tests.unit.virt.libvirt.checkpoints.test_store nova.tests.unit.virt.libvirt.checkpoints.test_staging`.
- [x] **Step 3 — Implement.** Copy loops ограничивают bytes до записи, temporary/final bytes одновременно учтены. Pure query читает seals/file identity; недоступный mount не означает deletion. Immutable manifests содержат format=2/binding/runtime fingerprint/artifact refs, без tokens/connection_info. Store access не меняет источник RAM при restore.
- [x] **Step 4 — GREEN.** Unit команда + `.tox/functional/bin/stestr --test-path=./nova/tests/functional run nova.tests.functional.test_checkpoint_store`; expected exit 0. Local POSIX process-crash тесты не называются NAS durability. Реальный libvirt writer/XFS quota/encryption — обязательная отдельная квалификация Task 9.
- [x] **Step 5 — Commit:** `feat: add shared checkpoint artifacts and bounded staging`.

## Task 5: C1.1 клиент и доступ после истечения user token

**Files:** Create `nova/volume/checkpoints.py`, `nova/tests/unit/volume/test_checkpoints.py`, `nova/tests/functional/test_checkpoint_cinder_contract.py`; Modify `nova/checkpoints/{api,coordinator,compute}.py` и минимум session helpers в `nova/volume/cinder.py`.

**Interfaces:** `CheckpointClient` methods:
`preflight_capabilities(context, volume_ids: tuple[str,...]) -> dict[str,dict]`,
`preflight_resources(context, volume_ids: tuple[str,...], snapshot_ids: tuple[str,...]) -> dict`,
`reserve(context, nova_operation_id: str, checkpoint_id: str, instance_uuid: str, kind: str, volume_ids: tuple[str,...]) -> dict`,
`get(context, nova_operation_id: str) -> dict`,
`action(context, nova_operation_id: str, name: str, generation: int, outcome: str|None=None) -> dict`,
`capabilities(context, nova_operation_id: str, volume_id: str) -> dict`,
`submit_attachment(context, nova_operation_id: str, request: dict) -> dict`,
`get_attachment_request(context, nova_operation_id: str, child_id: str) -> dict`,
`get_attachment(context, nova_operation_id: str, attachment_id: str) -> dict`.
Exact bodies/routes — C1.1 §2; Nova не придумывает второй serializer. Primary reserve — user+service, остальное — свежая собственная service authentication. User actor/project не записываются в fabricated context.

Исключение для первоначального preflight до reserve: `preflight_capabilities` использует существующий `/volumes/{id}/checkpoint-capabilities` с действующими user+service identity; `preflight_resources` читает штатные project-scoped volume/snapshot views тем же способом. Grant на этом этапе ещё нет. Background query использует только уже принятый continuation grant и его связанный resource set. Новый actor того же проекта может начать собственный revert/delete; он не читает чужую operation через старый actor-bound C1 route.

`resume_reserve_with_user(context, operation_id: str) -> None` — узкий API replay path: действующая policy + точный исходный actor/request/hash, уже сохранённый Cinder reserve payload и UUID. Допускается повтор **идемпотентного reserve с тем же ключом**, когда после timeout принятие неизвестно; не повтор destructive driver effect. Из другого actor или background admin context этот путь недоступен.

- [x] **Step 1 — RED.** HTTP test adapter записывает headers/body и имеет фиксированные C1.1 responses; реальные tokens заменены markers, не credentials. Setup binding соответствует accepted operation из response.

```python
def test_background_uses_service_identity_without_saved_bearer(self):
    self.user_session.token = 'expired-user-marker'
    result = self.client.get(self.recovery_ctx, uuids.nova_operation)
    self.assertEqual(uuids.nova_operation,
                     result['continuation']['nova_operation_id'])
    self.assertNotIn('expired-user-marker', self.http.last_headers.values())
    self.assertEqual('service-marker', self.http.last_headers['X-Auth-Token'])

def test_missing_grant_does_not_fall_back_to_reserve(self):
    self.http.respond(404, {})
    self.assertRaises(exception.CheckpointConflict,
                      self.client.get, self.recovery_ctx, uuids.nova_operation)
    self.assertEqual(['GET'], self.http.methods)
```

Test fixture service auth factory возвращает authenticated marker, recovery_ctx не имеет user token. Дополнить response binding mismatch, unknown/missing fields, wrong resource set, 3.72 capability, API endpoints с service project, no request logging, `202` pending, exact user replay after lost reserve, чужой actor replay forbidden.

- [x] **Step 2 — Run RED.** `.tox/py311/bin/stestr run nova.tests.unit.volume.test_checkpoints`.
- [x] **Step 3 — Implement.** Reuse keystoneauth sessions из Nova; service catalog lookup не использует untrusted response URL. Повтор GET на 401 может обновить только собственный service token; POST after timeout не повторять автоматически, кроме явно описанного keyed reserve. No user bearer в SQL/disk. Ordinary 3.44 wrappers сохраняются.
- [x] **Step 4 — GREEN.** Unit + `.tox/functional/bin/stestr --test-path=./nova/tests/functional run nova.tests.functional.test_checkpoint_cinder_contract`. Для последнего fixture запускает `cinder.tests.functional.checkpoint_continuation_fixture` из Task 6 C1.1 отдельным test process в его venv, читает ready manifest и проверяет dependency HEAD. Сверить реальные C1.1 schemas/responses, actor/generation/scope; не подменять C1 второй Nova fake-реализацией. Live Keystone остаётся V1.
  Test environment задаёт `N2_CINDER_CHECKOUT` и `N2_CINDER_PYTHON` на закреплённый checkout/его interpreter внутри той же disposable Linux-среды; fixture очищает Nova-specific test env для Cinder subprocess и завершает его после теста. Отсутствующие переменные — ошибка обязательного contract gate, не молчаливый skip.
- [x] **Step 5 — Commit:** `feat: integrate scoped Cinder checkpoint continuation`.

## Task 6: Topology validation и libvirt runtime effects

**Files:** Create `nova/virt/libvirt/checkpoints/{topology,runtime}.py`, `nova/tests/unit/virt/libvirt/checkpoints/{test_topology,test_runtime}.py`; Modify `nova/virt/libvirt/{driver,guest,host}.py` только для узких libvirt wrappers.

**Interfaces:** `Topology.inspect(context, instance, db_manifest: dict, purpose: str, saved_binding: Binding|None=None) -> dict` возвращает normalized runtime manifest/fingerprint/profile IDs без secrets; `Topology.restore_xml(saved_xml: str, current: dict, binding: Binding) -> str` разрешает только host-specific substitutions.
`Runtime.save(context, instance, child: ChildIntent, workspace: dict, topology: dict) -> EffectResult`, `.stop(context, instance, child, topology) -> EffectResult`, `.restore_paused(context, instance, child, artifact: ArtifactRef, workspace: dict, xml: str) -> EffectResult`, `.unpause(context, instance, child, binding: Binding) -> EffectResult`, `.query(context, instance, child) -> EffectResult`.

Save wrapper использует dedicated `virDomain.saveFlags(path, dxml, flags)`, restore — connection `restoreFlags(path,dxml,VIR_DOMAIN_SAVE_PAUSED)`; никаких managedSave/saveImageDefineXML на постоянном artifact. Проверка actual library API/flags обязательна в Linux runtime test. XML whitelist меняет disk host source/auth reference, текущие network host bindings и runtime metadata marker; CPU/machine/device addresses/serial/guest topology неизменны.

- [x] **Step 1 — RED.** Fixture libvirt connection/guest mocks связаны с изменяемым state и job lifecycle, не всегда возвращают success. Table-driven topology tests: ephemeral/swap/shares/multiattach/UEFI/vTPM/passthrough/block job/writable config drive → unsupported до save.

```python
def test_restore_always_starts_paused_before_validation(self):
    result = self.runtime.restore_paused(
        self.ctx, self.instance, self.child, self.artifact,
        self.workspace, self.current_xml)
    self.conn.restoreFlags.assert_called_once_with(
        self.workspace['ram_path'], self.current_xml,
        libvirt.VIR_DOMAIN_SAVE_PAUSED)
    self.guest.resume.assert_not_called()
    self.assertEqual('succeeded', result.status)

def test_marker_without_completion_evidence_is_unknown(self):
    self.guest.marker = self.matching_marker
    self.guest.job_outcome = 'unknown'
    result = self.runtime.query(self.ctx, self.instance, self.child)
    self.assertEqual('unknown', result.status)
    self.conn.restoreFlags.assert_not_called()
```

Setup создаёт filesystem artifact/seal через Task 4 fixture, bound child через Task 2 fixture. Дополнить save partial image vs complete seal, save reply loss, unpause reply loss, guest dies immediately, expected SHUTOFF recovery, changed CPU/keys/ports, active managed-save slot, old domain with same name but foreign UUID/marker. Valid header/checksum без доказательства завершения writer недостаточен.

- [x] **Step 2 — Run RED.** `.tox/py311/bin/stestr run nova.tests.unit.virt.libvirt.checkpoints.test_topology nova.tests.unit.virt.libvirt.checkpoints.test_runtime`.
- [x] **Step 3 — Implement.** Строгий device allowlist, два fingerprints, stable runtime marker и completion seals. Перед save/stop и unpause повторить ownership + актуальный topology snapshot; внешний drift после preflight остаётся обнаруживаемым, не объявляется невозможным. Neutron read использует существующую настроенную service auth; capability не следует из пустого admin context. Unknown job удерживает child; query не resume/destroy.
- [x] **Step 4 — GREEN.** Новые tests + stock suspend/resume/driver/guest/host tests; expected exit 0, существующий managed-save lifecycle не меняется. Реальный повторный restore и SAN flush semantics — Task 9 qualification/V1.
- [x] **Step 5 — Commit:** `feat: implement guarded libvirt checkpoint runtime effects`.

## Task 7: Scoped disconnect/reconnect и BDM identity

**Files:** Create `nova/checkpoints/attachments.py`, `nova/tests/unit/checkpoints/test_attachments.py`; Modify `nova/compute/manager.py`, `nova/virt/{virtapi,block_device}.py`, `nova/virt/libvirt/driver.py`, BDM OVO/guard paths только для operation-owned connection fields.

**Interfaces:** `AttachmentCoordinator.disconnect(context, instance, binding: Binding, parent_step_id: str) -> EffectResult`, `.reconnect(context, instance, binding, parent_step_id) -> EffectResult`, `.query(context, instance, binding, parent_step_id) -> EffectResult`; dependency injection C1.1 client, Effect OVO facade, BDM updater и driver `_connect_volume/_disconnect_volume` wrappers.
`ComputeVirtAPI.checkpoint_update_bdm(context, instance_uuid: str, operation_id: str, generation: int, bdm_uuid: str, attachment_id: str|None, connection_info: dict|None) -> None`: remoted mutation повторяет ownership, разрешает только эти connection fields. Отдельный hash canonical guest device identity неизменен.

- [x] **Step 1 — RED.** Setup два BDM/volumes, original/new attachment IDs, fake os-brick с effect counters и unknown response режимом; SQL-backed BDM для functional regression.

```python
def test_partial_revert_never_reconnects(self):
    self.cinder.results = {uuids.volume1: 'reverted', uuids.volume2: 'unknown'}
    result = self.coordinator.reconnect(self.ctx, self.instance,
                                        self.binding, self.parent_id)
    self.assertNotEqual('succeeded', result.status)
    self.driver._connect_volume.assert_not_called()
    self.assertEqual(self.original_bdm_ids, self.current_bdm_ids())

def test_new_attachment_changes_only_connection_fields(self):
    before = self.bdm_guest_identity()
    self.complete_reconnect()
    self.assertEqual(before, self.bdm_guest_identity())
    self.assertEqual(self.new_attachment_ids, self.current_attachment_ids())
```

Helpers здесь читают UUID/volume/device_name/serial/boot_index из BDM; `complete_reconnect()` исполняет все child intents через Task 2/5 interfaces. Добавить lost create/update/delete replies, wrong instance scope, stale BDM save после ownership release, incomplete attachment_complete, os-brick unknown disconnect, запрет обычного detach с destroy_bdm.

- [x] **Step 2 — Run RED.** `.tox/py311/bin/stestr run nova.tests.unit.checkpoints.test_attachments`.
- [x] **Step 3 — Implement.** Порядок disconnect: доказанный domain stop → host disconnect → scoped Cinder delete. Reconnect: все disk reverts confirmed → keyed attachment create → update/current connection_info → host connect → BDM save → complete. Connection_info не включать в child evidence, manifests или diagnostic logs; его хранение в обычном защищённом BDM следует существующему Nova пути.
- [x] **Step 4 — GREEN.** Новые tests + N1 guard races/review regressions и штатные block_device/volume-attachment tests; expected exit 0. Разрыв после любого перехода не удаляет BDM/volume и не запускает смешанные диски.
- [x] **Step 5 — Commit:** `feat: reconnect checkpoint volumes through scoped attachments`.

## Task 8: Полный executor, continuation и компенсация

**Files:** Create `nova/virt/libvirt/checkpoints/executor.py`, `nova/tests/unit/checkpoints/test_execution_v2.py`, `nova/tests/functional/test_checkpoint_executor.py`; Modify `nova/checkpoints/{api,compute,coordinator,recovery,quota}.py`, `nova/compute/{manager,rpcapi}.py`, `nova/virt/libvirt/driver.py`, main DB finalize/close-proof paths.

**Interfaces:** `LibvirtCheckpointExecutor(driver, runtime, store, staging, cinder, attachments)` реализует v2 hooks Task 1; отдельная registry сопоставляет фазы §2.2 с конечным набором child actions. `ComputeExecutor.continue_step(context, instance, operation_id: str, generation: int, step_id: str, journal_revision: int) -> dict`; Compute RPC `checkpoint_continue` с той же сигнатурой, version 6.6. Caller не передаёт произвольный action/resource. Conductor получает `ready`, сохраняет observation и инициирует continue с проверенной revision.

Child dispatch IDs стабильны; для runtime restore phase два разных children `restore_paused` и `unpause`, между ними topology validation. Для Cinder parent phases отдельные submit/query и проверка полного resource set. `artifact_publish_shared` не выдаёт `staging_deleted`; settle proof появляется после `staging_cleanup`.

`workflow_v2.child_plan(kind: str, phase: str, resources: dict) -> tuple[dict,...]` — чистый builder разрешённого graph для DB Task 2. Фиксированные actions: preflight=`inspect`; cinder_reserve=`reserve`; runtime_save=`reserve_staging,copy_config,save`; disk phases=`submit_disk_operation` (один Cinder op с его собственными per-volume steps); publish/stage/delete/cleanup=`publish|download|delete` по каждому перечисленному artifact; runtime_stop=`stop`; runtime restore=`restore_paused,unpause`; volume disconnect/reconnect — per-volume steps Task 7; cinder_release=`release`. При отсутствии config-drive `copy_config` отсутствует. Предшественники обязательны, resource set берётся из locked manifest и уже подтверждённых предыдущих фаз. Query только наблюдает существующий graph.

- [x] **Step 1 — RED.** `ExecutorV2FunctionalTest` расширяет N1 CheckpointTestCase с SQL/WSGI/RPC, но использует настоящий N2 executor и компоненты с fake libvirt/storage backends. `self.advance()` — один coordinator pass; fake effect backends имеют счётчик write calls, не обход journal.

```python
def test_ready_advances_next_child_without_reexecuting_previous(self):
    self.start_create_with_pending_save()
    self.backend.confirm_save()
    self.advance()  # query -> ready
    self.assertEqual(1, self.backend.save_calls)
    self.advance()  # continue -> next prepared child
    self.assertEqual(1, self.backend.save_calls)
    self.assertEqual(1, self.backend.snapshot_calls)

def test_partial_snapshot_cleanup_keeps_bytes_until_all_deleted(self):
    self.backend.fail_second_snapshot_terminally()
    self.run_create_until_wait()
    self.assertGreater(self.charged_bytes(), 0)
    self.assertFalse(self.checkpoint_is_available())
    self.backend.allow_owned_cleanup()
    self.run_until_terminal()
    self.assertEqual(0, self.charged_bytes())
    self.assertEqual(0, self.backend.foreign_deletes)
```

Task-local helpers drive create API and coordinator until no progress, with bounded iteration count; hang is failure, не бесконечный loop. Дополнить A/B/revert/delete, service token expiry, failed save with residues, late upload after cancellation, stale continuation revision, delete без работающего domain, revert pre-stop abort, partial revert, compute restart before autostart, `enabled=false` existing operations, exact user reserve replay, down cell и host mismatch.

- [x] **Step 2 — Run RED.** `.tox/functional/bin/stestr --test-path=./nova/tests/functional run nova.tests.functional.test_checkpoint_executor`.
- [x] **Step 3 — Implement.** Wire new executor only для libvirt с положительными profiles. Не ослаблять N1 guards при включении runtime. Прежде компенсации зафиксировать fence ещё не dispatched children. Если один writer/result неизвестен — recovery_required, никаких success/refund. Origin host remains sole executor. Startup query выполняется раньше ordinary power-sync/autostart; обычные workers сохраняют N1 claims.
- [x] **Step 4 — GREEN.** Functional команда, v2 unit suite, N1 functional/test_checkpoint_review и все quota/guard suites; expected exit 0, полная матрица failures содержит safety assertion на каждом переходе.
- [x] **Step 5 — Commit:** `feat: orchestrate libvirt checkpoints with durable continuation`.

## Task 9: Проверки профиля, regression, review и пакет

**Files:** Create `nova/checkpoints/qualification.py`, `tools/checkpoint/qualification.py`, `tools/checkpoint/profile.schema.json`, `nova/tests/unit/checkpoints/test_qualification.py`, `doc/source/admin/instance-checkpoint-executor.rst`, `doc/source/admin/instance-checkpoint-qualification.rst`; Modify admin index, N1 admin/API reference только для нового поведения и версий, без переписывания исторических deliverables.

**Interfaces:** `nova.checkpoints.qualification.inspect(config_path: str) -> dict`, `.validate_report(report: dict, evidence_root: str) -> dict`; CLI wrapper `inspect --config PATH --output PATH` и `validate-report --report PATH --evidence-root PATH --output PATH`. Inspect читает Nova config, versions/mount/store identities; exit 0 означает только сбор данных. CLI не содержит VM-mutating команд. Фактические испытания описываются отдельным V1-runbook с явно указанными disposable instance UUID/host/volumes и разрешёнными actions.

Report schema version=1: `scope=components|stack`, measured tuple, checks с pass/fail/skipped и именованными evidence files/SHA-256, timings/peak bytes; `qualified=false` при любом обязательном unrun/fail/missing evidence. `components` подтверждает runtime/store/staging primitives до интеграции; на его основе формируется allowlist Task 1. `stack` дополнительно требует реальный C1.1/Keystone/Neutron/D-profile и A/B lifecycle; без него не заявлять V1/production readiness. Это устраняет цикл «для квалификации уже нужен включённый полностью квалифицированный стек». Inspect никогда не создаёт positive profile. Readiness storage crypto, не наблюдаемая с клиента, требует отдельного проверенного инфраструктурного evidence, а не автоматически заполненного true.

- [x] **Step 1 — RED qualification tests.** Fake inspectors являются dependency injection tests, не production backend. Проверить `test_inspect_never_changes_vm`, `test_missing_evidence_never_qualifies_profile`, `test_components_report_is_not_stack_acceptance`, `test_skipped_required_check_never_qualifies_profile`, `test_profile_tuple_mismatch_is_unsupported`.

```python
def test_skipped_required_check_never_qualifies_profile(self):
    checks = {'libvirt_repeat_restore': 'pass',
              'writer_hard_limit': 'skipped',
              'nas_durability': 'pass'}
    report = qualification.summarize(checks, required=tuple(checks))
    self.assertFalse(report['qualified'])
    self.assertEqual(['writer_hard_limit'], report['unverified'])
```

`qualification.summarize(checks: dict[str,str], required: tuple[str,...]) -> dict` должен также отвергать отсутствующий required key и invalid status; missing не равен pass.

- [x] **Step 2 — Run RED.** `.tox/py311/bin/stestr run nova.tests.unit.checkpoints.test_qualification`.
- [x] **Step 3 — Implement qualification/docs.** Component checks: save→restore paused→unpause и повторное чтение immutable RAM, shared durability/partial publish, реальный libvirt writer quota overflow, отсутствие cleartext spill и XML/device identity. Stack checks: A/B/revert/delete, RAM-only marker + каждый disk marker, сохранение UUID, partial revert, истёкший user token с реальным Keystone после restart, current Neutron bindings, pause/peak bytes. Для каждого runbook check дать команду/действие на явно выбранном тестовом объекте, ожидаемый результат и evidence filename; unrun остаётся unverified. Operator recovery guide описывает evidence collection и запрет force unlock; не содержит непроверенного автоматического восстановления unknown backend effect.
- [x] **Step 4 — GREEN local gates.** Ниже полный unit/functional и штатные profiler/docs/API-ref/genconfig/genpolicy/redirect checks в подготовленной среде. Новые tests не пропускать; failures и skip identities сверить с точным N1 baseline. Linters сравнить на disposable base/feature copies без переносов unrelated formatter edits.

```sh
.tox/py311/bin/stestr run --concurrency 8 --exclude-regex '^nova.tests.unit.virt.libvirt.volume.test_sbd'
TEST_OSPROFILER=1 .tox/py311/bin/stestr run --combine --no-discover nova.tests.unit.test_profiler
.tox/functional/bin/stestr --test-path=./nova/tests/functional run --concurrency 8
.tox/docs/bin/sphinx-build -W --keep-going -b html -j auto doc/source doc/build/html
.tox/docs/bin/sphinx-build -W --keep-going -b html -j auto api-ref/source api-ref/build/html
.tox/py311/bin/oslo-config-generator --config-file=etc/nova/nova-config-generator.conf
.tox/py311/bin/oslopolicy-sample-generator --config-file=etc/nova/nova-policy-generator.conf
git diff --check
```

Expected exit 0 для suites; original baseline lint diagnostics не считаются новым PASS всей pep8 среды. Новый/изменённый код не добавляет diagnostics. Отдельно сохранять raw subunit, module/test inventory и machine outcomes. Документировать unrun KVM/NAS/SAN/Keystone gates, не превращать их в штатные skips, подтверждающие production.

- [x] **Step 5 — Commit:** `test: qualify checkpoint executor and document operational limits`.
- [x] **Step 6 — Native final review.** Один свежий reviewer base..HEAD, включая C1.1 wire compatibility, authority boundary, all partial effects, quotas и storage namespace. Один fix pass с RED/GREEN по findings и необходимой regression. Финальный авторский прогон не называть вторым независимым review.
- [x] **Step 7 — Package.** `deliverables/nova-checkpoint-executor-n2/`: patches/series, BASE/HEAD commit/tree, C1.1 dependency commit/tree, SOURCE_SHA256, README, DESIGN/PLAN, VALIDATION, REVIEW/RESOLUTION, RULINGS, evidence, SHA256SUMS. Проверить N1→N2 patches на чистом ZIP с точным конечным tree и git modes. Локальное завершение N2 отдельно от V1; не push/merge/deploy.

## 3. Покрытие и порядок приёмки

| Spec | Tasks / внешний gate |
|---|---|
| §1 границы VM/profile и UUID | 1, 6, 7, 8 |
| §2 разрывы N1/C1 | 2, 3, 5, 8; C1.1 |
| §3 отдельные пакеты | C1.1 → N2, D/V1 отдельно |
| §4 shared/staging/crypto/namespace/квоты | 1, 3, 4; qualification 9 |
| §5 service continuation и actor | C1.1; 5, 7, 8 |
| §6 children/ready/partial results | 1, 2, 8 |
| §7 probe/create/revert/delete | 4–8 |
| §8 restart/unknown/late effects | 2–8 |
| §9 compatibility и default-off | 1, 2, 3, 5, 8, 9 |
| §10 local tests и live acceptance | 9; actual D1/D2/V1 запуск отдельно |
| §11 инфраструктурные решения | Существующий NAS — кандидат; точный профиль и реальные результаты требуются для включения |

В отчёте выполнения отдельно указать: готовность патчей/локальных tests; готовность qualification tools/runbook; какие реальные profiles квалифицированы. Отсутствующий стенд не оправдывает фиктивный positive probe и не мешает закончить локальный пакет с явным unsupported profile. Изменение алгоритма ради уменьшения pause, cross-host restore или новый store backend требует отдельного проекта.

План сохранён вне Git. После пользовательского review выполнить сначала C1.1 Tasks 1–6, затем N2 Tasks 1–9, сохраняя Native метод.
