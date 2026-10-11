# Nova Instance Checkpoint N1 Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking. Для этой работы пользователь уже выбрал Native: использовать `superpowers:executing-plans`, без повторного выбора метода.

**Goal:** Реализовать в форке Nova общий, проверяемый без реальной СХД контур Instance Checkpoint: cell-aware API, durable journal, глобальные резервы квот, guards и recovery.

**Architecture:** Жизненный цикл и владение ВМ хранятся в cell DB, проектные quota tickets — в API DB. Conductor координирует переходы; compute использует OVO remoting и отдельный executor contract, базовая реализация которого возвращает unsupported. Вызовы libvirt, RAM-store и Cinder остаются пакету N2.

**Tech Stack:** Python 3.11/Linux, Nova fork, oslo.versionedobjects/messaging/db, SQLAlchemy/Alembic, SQLite/MySQL/PostgreSQL, stestr, Nova WSGI/functional fixtures.

**Spec:** [Согласованная спецификация N1](../specs/2026-10-07-nova-checkpoint-model-n1-design.md). Ответ пользователя «ок» после ссылки на эту спецификацию разрешает подготовку данного плана; план утверждён пользователем ответом «да»; начато Native-исполнение.

## Global Constraints

- «Архивы и `sources/` остаются эталоном.» Checkout: `work/nova-checkpoints`, ветка `feature/nova-checkpoint-model`; не изменять готовый C1, не push/deploy, не обращаться к действующим OpenStack/СХД.
- «N1 **не создаёт работающие RAM snapshots сам по себе**.» Production executor unsupported; fake разрешён только тестам.
- «Нельзя держать SQL transaction через RPC, Cinder или libvirt.» Нет распределённой SQL transaction API DB ↔ cell DB.
- «Compute использует OVO/conductor remoting; прямой доступ nova-compute к cell SQL не добавляется.»
- «Поколение 1 неизменно в N1; takeover не реализуется.» Timeout и выключение флага не снимают locks/claims/quota holds.
- «Частичный revert/неизвестный эффект оставляет ВМ остановленной и guard занятым.» Повтор destructive step запрещён после durable claim; только query.
- «Нельзя освобождать ticket по timeout, отсутствию ответа или однократному `operation not found`»; отмена admission требует durable cancellation.
- «max_checkpoints_per_instance=8», «max_checkpoints_per_project=100», «max_artifact_bytes_per_project=0», «enabled=false», «reconcile_interval=60». Ноль bytes запрещает новые create; отрицательные лимиты запрещены.
- Частные версии: Nova API **2.106**, Compute RPC **6.5**, service **69**, compute_task RPC **1.26**; общий conductor RPC остаётся **3.0**, обычный Instance OVO — **2.8**.
- «Само сохранение user UUID не даёт Nova права получить токен от его имени.» Не сохранять user token и не подменять C1 owner; фоновая Cinder-авторизация — N2.
- Профиль общей spec: KVM/libvirt, BIOS, тот же compute; только Cinder-backed writable disks, без ephemeral/swap/shares/multiattach/passthrough/vTPM/confidential VM. Подробная физическая проверка профиля — N2 probe; неизвестное свойство не означает поддержку.
- До эксплуатации должны быть обновлены все писатели. Fake transport и успешная SQL-проверка не доказывают работу Keystone/RabbitMQ/libvirt/SAN.

## Review Focus

1. Поздний admission после отмены quota ticket, включая пересечение request UUID разных действий: refunded ticket никогда не должен разрешить новый эффект. Тесты Task 2/3.
2. Обычный worker, прошедший API раньше checkpoint, и вложенные remoted OVO saves: backend I/O и checkpoint не получают владение одновременно; чужой completion не снимает claim. Task 4.
3. Смена mapping, BDM, network identity или runtime key между probe и admission: устаревший fingerprint не допускается, quota остаётся корректной. Task 4/6.
4. Restart при частичном revert и включённых power-sync/external events: guest не стартует, квота/guard не освобождаются по времени. Task 7.
5. Смешанные RPC/service versions, поддельный scope и истёкший user token: отказ до эффекта, read-only recovery не превращается в impersonation. Task 5/8.

---

## 1. Исходная точка и воспроизводимость

Nova ZIP SHA-256 `5e20485c94a525a758141bf1d2dadfd78c88c9166fac704e8e18346f207957a7`, comment revision `21072e1cd39a70ac5490624177c41e4fb116da9e`. Локальный Git baseline создаётся из ZIP отдельно; это не commit upstream. C1 остаётся на `d2d14a5251fa7f61771f83bfbf50f12215dd051d`.

Все пути Task 1–9 ниже относительны будущему `work/nova-checkpoints/`, кроме явно названных внешних документов. До начала Task 1 использовать using-git-worktrees для изоляции/проверки checkout; сохранить symlinks из ZIP, точные bytes и выбранные Git modes в manifest. Не копировать `.git` другого проекта и не записывать в `sources/`.

Исполнение в одноразовом Linux container, `/work` — mount checkout. Python 3.11, constraints из исходного `tox.ini`: `https://releases.openstack.org/constraints/upper/2025.1`. Системные зависимости — по `bindep.txt`/`tools/test-setup.sh`. Сеть и контейнеры тестовых MySQL/PostgreSQL отдельные, без публикации портов; адреса/пароли только тестовые. Host Python не изменять. Сохранять freeze и версии среды, не менять production requirements ради локального запуска.

Настройка внутри test container: `tox -e py311,functional --notest`. Для дальнейших команд установить `PBR_VERSION=0.0.0`, `OS_TEST_TIMEOUT=160`, `OS_STDOUT_CAPTURE=1`, `OS_STDERR_CAPTURE=1`, `PYTHONDONTWRITEBYTECODE=1`, `NOVA_RAISE_ON_GREENLET_LEAK=True`, корректную UTF-8 locale. Задать `OS_TEST_DBAPI_ADMIN_CONNECTION` со **всеми тремя** disposable URL: SQLite, MySQL/PyMySQL и PostgreSQL/psycopg2. Прямой запуск stestr ниже наследует эту переменную; не полагаться на отсутствующий passenv tox. Для запуска через tox добавить только test-environment passenv override либо явный тестовый конфиг, записать точную команду в ledger.

Команды Task 1–9 выполняются внутри container, независимо от способа доставки `docker exec`. `stestr` запускает штатный Nova test runtime. Большие логи писать в scratch текущего плана; RED считать доказанным только при ожидаемой ошибке продукта, не при отсутствии зависимостей. На финальной ветке нужны все unit и functional suites с исходными skips; сравнивать неожиданные отказы с baseline. Никакое исключение SberCloud из C1 не переносится автоматически на Nova.

## 2. Структура и границы модулей

| Файлы | Ответственность |
|---|---|
| `nova/checkpoints/{__init__,contracts,workflow}.py` | Чистые типы, validation, canonical hashes и разрешённые переходы; без SQL/libvirt/import драйверов |
| `nova/conf/checkpoint.py`, `nova/conf/__init__.py` | Централизованные config options, register/list по шаблону Nova |
| `nova/db/main/checkpoints.py`, `nova/db/main/models.py` | Cell journal, claims, guards, cancellation proofs; writer/reader facade main DB |
| `nova/db/api/checkpoints.py`, `nova/db/api/models.py` | API DB usage/tickets под отдельным facade |
| `nova/objects/checkpoint.py`, `nova/objects/__init__.py` | Versioned projections и remotable DB operations; compute не импортирует DB helper |
| `nova/checkpoints/{quota,guards,api,coordinator,compute,recovery}.py` | Межбазовые последовательности, scope, API service, orchestration, executor boundary, query-only repair |
| `nova/context.py`, `nova/compute/{api,manager,rpcapi,task_states}.py`, `nova/conductor/{api,manager,rpcapi}.py` | Узкая интеграция в существующие входы; без общей переписки больших модулей |
| `nova/virt/driver.py`, `nova/objects/service.py` | Три default-unsupported hooks, версии и service history |
| `nova/api/openstack/compute/{server_checkpoints,views/server_checkpoints,schemas/server_checkpoints}.py` | Routes/controller, sanitization и строгие schemas |
| `nova/policies/server_checkpoints.py`, `nova/policies/__init__.py` | Reader/member rules с server project target |
| `nova/tests/unit/checkpoints/` | Тесты контрактов, SQL, quota, guards, orchestration, recovery |
| `nova/tests/functional/{test_checkpoints,db/test_checkpoints,api_sample_tests/test_checkpoints}.py` | Две cells, WSGI/RPC/SQL, samples |
| `doc/source/admin/instance-checkpoints.rst`, `api-ref/source/server-checkpoints.inc` | Границы N1, API и операторская диагностика |

### 2.1. Общие структуры (Task 1)

Типы ниже — immutable dataclasses внутри процесса; для RPC/JSON используются строгие `to_primitive()/from_primitive()` с `schema_version=1`. Тип `Context` означает Nova RequestContext. Не принимать произвольные class names/pickle и не включать credentials.

- `RequestKey(project_id: str, instance_uuid: str, request_id: str, actor_user_id: str, kind: str, payload_hash: str)`. SQL uniqueness: `(project_id, instance_uuid, request_id)`; actor/kind/hash проверяются при replay.
- `Admission(key: RequestKey, operation_id: str, checkpoint_id: str, cell_uuid: str, host: str, node: str, fingerprint: str, device_manifest: dict, artifact_upper_bound: int)`. IDs генерируются сервером; для exact create replay брать выигравшие IDs из ticket, не новые candidate IDs.
- `Probe(supported: bool, reasons: tuple[str, ...], executor_id: str, executor_profile: str, fingerprint: str, device_manifest: dict, artifact_upper_bound: int, contract_version: int=1)`. Unsupported не содержит фиктивных положительных evidence. Все reason codes из закрытого списка; неизвестный ответ → `invalid_executor_result`.
- `Scope(instance_uuid: str, kind: str, owner_uuid: str, generation: int, action: str, token: str)`; kind=`checkpoint|ordinary`, token генерируется сервисом. Переданный scope не даёт права без текущей matching DB записи.
- `StepIntent(id: str, operation_id: str, generation: int, phase: str, executor_id: str, payload_hash: str, payload: dict)`; stable ID до I/O.
- `StepResult(status: str, evidence: dict, error_code: str | None)`; terminal `succeeded` принимается только с полным evidence конкретной фазы. Исключение после claim → `unknown`, не `failed_no_effect`.
- `CloseProof(create_operation_id: str, closing_operation_id: str, checkpoint_id: str, cell_uuid: str, kind: str, proof_version: int=1)`; kind=`admission_cancelled|resources_deleted`. Получается из durable cell records; не из тела запроса или клиентского bool.

Fingerprint: SHA-256 canonical JSON с сортировкой mappings и resource UUID. Включать instance UUID, host/node, flavor resource/device fields, image/device properties, config-drive identity, BDM UUID/volume UUID/serial/device/boot index, port UUID/MAC/network UUID/vNIC/model, share IDs, runtime key fingerprint set. Не включать task timestamps, counters, IP/DHCP/security groups, bearer token, connection secrets. N1 атомарно сверяет DB-проекцию; N2 обязан дополнительно сверять реальную конфигурацию и физическую топологию до destructive effect.

### 2.2. Cell schema и ownership (Task 2)

Additive migration `4a8c2d7e9101_add_instance_checkpoints.py`, parent `2903cd72dc14`. Models со стандартными Nova timestamps, без автоматического soft-delete/archive для durable journal. Сохранять deleted/tombstone status, не физически удалять operation/request rows. Migration model-sync учитывает только реально создаваемые shadow tables; не добавлять их автоматически новым durable моделям.

Шесть таблиц из spec: `instance_checkpoints`, `instance_checkpoint_disks`, `instance_checkpoint_artifacts`, `instance_checkpoint_operations`, `instance_checkpoint_steps`, `instance_checkpoint_guards`. Поля — spec §6; дополнительно закрепить `operation.quota_ready` и `operation.quota_released` bool default false, `checkpoint.terminal_operation_id`, `guard.ordinary_claim_uuid/token/action/state`, `guard.checkpoint_count`, `operation/step.version` для CAS, внутренний result JSON и закрытые error codes. Instance/host/project identity неизменяемы; ссылки не cascade-delete при удалении server.

Добавить маленькую `instance_checkpoint_admission_tombstones`: `operation_id` PK, `cell_uuid`, `instance_uuid`, `project_id`, `request_id`, `payload_hash`, timestamps. Это физическое представление уже требуемой spec cancellation: оно необходимо отдельно от уникального request key операции, чтобы отклонённый create ticket можно было отменить, даже если тот же request UUID выиграла другая операция. Отмена чужого **принятого** operation UUID запрещена; наличие tombstone и принятой operation для одного UUID взаимно исключается под Instance lock.

OVO `InstanceCheckpoint`, `InstanceCheckpointDisk`, `InstanceCheckpointArtifact`, `InstanceCheckpointOperation`, `InstanceCheckpointStep`, `InstanceCheckpointGuard` и нужные List classes начинают с 1.0. Cancellation model обслуживается remotable method Operation; лишний публичный OVO для него не нужен. Generic create/save не обходят переходы. Обычные Instance/BDM/VIF/ShareMapping OVO fields/version не менять ради checkpoint.

Task states: `checkpoint_creating`, `checkpoint_reverting`, `checkpoint_deleting`, `checkpoint_recovering`. Обычный vm_state не ставить ACTIVE как замену доказательству runtime; прежние состояния сохраняются в operation.

### 2.3. Quota schema и межбазовый протокол (Task 3)

API DB migration `71be90d2c402_add_checkpoint_quota_ledger.py`, parent `cdeec0c85668`.
`checkpoint_project_usage(project_id PK, checkpoint_count BIGINT, artifact_bytes BIGINT, version)` и `checkpoint_quota_tickets(id PK, operation_id UNIQUE, checkpoint_id, project_id, instance_uuid, actor_user_id, request_id, payload_hash, cell_uuid, reserved_bytes BIGINT, state, version, timestamps)`, unique create request key. Значения count/bytes неотрицательны, верхняя граница signed BIGINT проверяется до SQL.

Ticket states `held|confirmed|released`; held и confirmed одинаково занимают count=1 и reserved_bytes. Живой ticket сохраняется на весь срок checkpoint, включая error/cleanup. Revert/delete не создают второй count ticket: они завершают работу с исходным create ticket. Реализация глобального refund принимает ticket UUID, самостоятельно запрашивает immutable CloseProof из закреплённой cell и лишь затем начинает API DB writer; нельзя держать обе DB sessions или верить переданному наружным API proof.

Create порядок: preflight → API DB `reserve` → cell `admit` → API DB `confirm` → cell `mark_quota_ready` → dispatch. Если confirmation потеряно — usage уже занят; recovery подтверждает тот же ticket. До quota_ready step_claim запрещён. Отмена cell admission и разрешение первого step_claim сериализуются одним Instance lock; quota release идёт после cancellation. Cell unavailable удерживает ticket.

Для delete/abort cell сначала сохраняет terminal resource proof и closing operation UUID, сохраняя guard. QuotaLedger.release адресует original create ticket, находит именно closing operation по checkpoint, освобождает usage один раз; затем `mark_quota_released` и `finalize` снимают guard/слот. Для успешного create ticket остаётся confirmed, refund не выполняется. Отмена admission возвращает proof с совпадающими create/closing IDs.

Расширение bytes: сначала API DB CAS на увеличение, затем cell artifact budget CAS, потом запись N2. Shrink/refund: сначала cell фиксирует подтверждённое удаление staging/артефактов и новый immutable бюджетный proof, затем API DB уменьшает charge ровно один раз по proof revision. Не менять размер непосредственно по dict ответа клиента. Per-instance limit учитывает creating/available/error/deleting, пока ресурсы не очищены; tombstones released/deleted не занимают слот.

### 2.4. Фазы и доказательства (Task 5)

`workflow.steps_for(kind: str) -> tuple[str, ...]`:

- create: `preflight`, `cinder_reserve`, `runtime_save`, `disk_snapshot`, `artifact_publish`, `runtime_restore_current`, `cinder_release`;
- revert: `preflight`, `cinder_reserve`, `runtime_stop`, `volume_disconnect`, `disk_revert`, `volume_reconnect`, `runtime_restore_checkpoint`, `cinder_release`;
- delete: `cinder_reserve`, `disk_snapshot_delete`, `artifact_delete`, `cinder_release`;
- create cleanup при подтверждённых исходах: `cleanup_runtime_restore_current` (только если runtime_save имел эффект и runtime ещё не восстановлен), `cleanup_disk_snapshots`, `cleanup_artifacts`, `cleanup_cinder_release`.

DB finalization и quota refund — локальные переходы, не произвольные executor effects. Manifest дисков всегда полный, операция на части набора не разрешена.

Evidence — versioned phase-specific schema: preflight подтверждает тот же fingerprint/host/node; runtime_save — immutable artifact IDs/checksums/bytes и прекращение записей; snapshots — исходный набор volume IDs и подтверждённые snapshot IDs/status; disconnect — все исходные подключения отключены; revert/delete — результат каждого исходного ресурса; reconnect — те же volume/port identities и новые валидные attachments; restore — нужный runtime fingerprint и RUNNING; release — совпадающая Cinder operation/generation с terminal outcome. Для cleanup требуется подтверждённое отсутствие каждого созданного ресурса/завершённое возвращение текущего runtime. Эти сведения принимает только доверенный executor, не REST клиент. N1 fake проверяет форму/связи, не доказывает физическую истинность.

## 3. Задачи реализации

### Task 1: Контракт, настройки и исходная тестовая среда

**Files:** Create `nova/checkpoints/{__init__,contracts,workflow}.py`, `nova/conf/checkpoint.py`, `nova/tests/unit/checkpoints/{__init__,test_contracts}.py`. Modify `nova/conf/__init__.py`, `nova/exception.py`.

**Interfaces:** Produces типы §2.1, `canonical_payload_hash(payload: dict) -> str`, `device_fingerprint(manifest: dict) -> str`, `validate_probe(value: dict) -> Probe`, `validate_result(phase: str, value: dict) -> StepResult`, `steps_for(kind: str) -> tuple[str, ...]`; config opts из Global Constraints. Exceptions `CheckpointConflict` 409, `CheckpointNotSupported` 409, `CheckpointUnavailable` 503, `CheckpointQuotaExceeded` 413; безопасные reason codes.

- [ ] **Подготовить изолированный baseline и среду §1.** Проверить SHA ZIP, создать локальный baseline commit и ветку; сохранить source manifest/tree. Run: `tox -e py311,functional --notest`. Expected: среды созданы без изменения tracked requirements. Run `.tox/py311/bin/stestr run '^nova.tests.unit.(test_quota|compute.test_rpcapi|db.main.test_migrations|db.api.test_migrations)'`; expected исходный baseline без новых N1 failures; сохранить failures/skips отдельно, не приписывать их N1.
- [ ] **Написать RED.** `test_defaults_disabled`: `assertFalse(enabled)`, limits `(8, 100, 0)`, interval 60; `test_invalid_probe_fails_closed`: missing/extra/unknown identity → unsupported; `test_phase_success_requires_evidence`: `succeeded` без evidence → unknown; `test_canonical_request_hash`: перестановка keys даёт тот же hash, изменение actor/kind — другой; `test_no_secrets_in_public_reason`: произвольный backend exception не попадает в response.
- [ ] **Запустить RED:** `.tox/py311/bin/stestr run '^nova.tests.unit.checkpoints.test_contracts'`. Expected: failures из-за отсутствующего контракта/настроек, а не dependency errors.
- [ ] **Реализовать чистые типы и validators.** Central config registration; закрытый список полей и кодов. Определить exact phase evidence keys из §2.4, запрещать частичный disk set и произвольные serialized objects. Не импортировать libvirt/Cinder SDK/DB в contracts.
- [ ] **Запустить GREEN:** та же команда. Expected: все contract tests pass, config defaults сохранены. Run `git diff --check`; expected exit 0.
- [ ] **Commit:** `git add nova/checkpoints nova/conf/checkpoint.py nova/conf/__init__.py nova/exception.py nova/tests/unit/checkpoints`; `git commit -m "feat: define Nova checkpoint contracts and safe defaults"`.

### Task 2: Cell journal, отмена admission и OVO remoting

**Files:** Create `nova/db/main/checkpoints.py`, `nova/objects/checkpoint.py`, `nova/db/main/migrations/versions/4a8c2d7e9101_add_instance_checkpoints.py`, `nova/tests/unit/checkpoints/test_cell_db.py`, `nova/tests/unit/objects/test_checkpoint.py`. Modify `nova/db/main/models.py`, `nova/objects/__init__.py`, `nova/compute/task_states.py`, `nova/tests/unit/{db/main/test_migrations,objects/test_objects}.py`.

**Interfaces:** Consumes Admission/StepIntent/Result из Task 1. DB functions зеркалируются remotable classmethods `InstanceCheckpointOperation`:
`admit(context, admission: Admission, ticket_id: str | None) -> InstanceCheckpointOperation`;
`cancel_admission(context, admission: Admission) -> CloseProof`;
`mark_quota_ready(context, operation_id: str, ticket_id: str) -> None`;
`get_by_uuid(context, operation_id: str) -> InstanceCheckpointOperation`;
`find_request(context, key: RequestKey) -> InstanceCheckpointOperation | None`;
`claim_step(context, operation_id: str, generation: int, phase: str, executor_id: str) -> StepIntent | None`;
`finish_step(context, step_id: str, generation: int, result: StepResult) -> None`;
`finalize(context, operation_id: str, generation: int, outcome: str) -> InstanceCheckpointOperation`;
`close_proof(context, create_operation_id: str) -> CloseProof | None`;
`mark_quota_released(context, closing_operation_id: str, ticket_id: str) -> None`.
Step/List projections provide `get_by_uuid(context, step_id)` и `get_by_operation(context, operation_id)`.
Checkpoint/List projections provide `get_by_uuid`/`get_by_instance_uuid(context, instance_uuid, marker=None, limit=100)`; filters always bound to server/project. Mutation helpers используют context scope, который будет подключён Task 4.

- [ ] **Написать RED.** `test_admit_atomic_with_instance_task`: journal + guard + task_state либо все, либо ни одного; `test_same_instance_two_requests_one_winner`: `sum(winners)==1`; `test_cancel_then_late_admit`: conflict и `step_count==0`; `test_cancel_create_when_other_kind_owns_request`: cancellation proof сохраняется отдельно, победившая операция неизменна; `test_scope_identity_immutable`, `test_wrong_generation_cannot_finalize`; `test_deleted_tombstone_replay_same_uuid`; migration upgrade/model-sync и старые Instance/BDM rows неизменны.
- [ ] **Запустить RED:** `.tox/py311/bin/stestr run '^nova.tests.unit.(checkpoints.test_cell_db|objects.test_checkpoint|db.main.test_migrations)'`. Expected: новые journal tests fail, baseline migrations остаются различимыми.
- [ ] **Реализовать models/migration и transitions §2.2.** Instance → guard → checkpoint → operation → stable-order children. Insert cancellation и первый admit имеют один serialization lock; exact duplicate возвращает прежнюю запись. `claim_step` разрешён только для следующей фазы, правильного executor/generation и quota_ready; SQL retry не содержит I/O. При выпуске guard исключить чужое ownership. Tombstones не удалять.
- [ ] **Реализовать remotable projections.** Все compute-facing mutations через методы OVO; стандартные `obj_make_compatible`, serializer, List relationships; Hash changes только ожидаемых новых OVO. Нельзя передать произвольное final status через generic save.
- [ ] **Запустить GREEN:** та же команда плюс `.tox/py311/bin/stestr run '^nova.tests.unit.objects.test_objects'`. Expected: pass на SQLite/MySQL/PostgreSQL, реальные races не skipped; OVO roundtrip/backport expected behavior. `git diff --check` exit 0.
- [ ] **Commit:** stage перечисленные файлы; `git commit -m "feat: persist cell checkpoint journal and admission cancellation"`.

### Task 3: Глобальные quota tickets и компенсация между БД

**Files:** Create `nova/db/api/checkpoints.py`, `nova/checkpoints/quota.py`, `nova/db/api/migrations/versions/71be90d2c402_add_checkpoint_quota_ledger.py`, `nova/tests/unit/checkpoints/test_quota.py`, `nova/tests/functional/db/test_checkpoint_quota.py`. Modify `nova/db/api/models.py`, `nova/objects/checkpoint.py`, `nova/tests/unit/{db/api/test_migrations,objects/test_objects}.py`.

**Interfaces:** Consumes Task 2 admit/cancel/close_proof. `QuotaLedger.get(context, ticket_id: str) -> CheckpointQuotaTicket`; `reserve(context, admission: Admission) -> CheckpointQuotaTicket`; `confirm(context, ticket_id: str) -> CheckpointQuotaTicket`; `grow(context, ticket_id: str, expected_version: int, total_bytes: int) -> CheckpointQuotaTicket`; `settle(context, ticket_id: str) -> CheckpointQuotaTicket`; `release(context, ticket_id: str) -> CheckpointQuotaTicket`; `reconcile(context, limit: int=100) -> dict`. Ticket OVO 1.0 + List; `settle` получает новый actual-byte proof из cell самостоятельно, `release` — immutable terminal proof. Cell OVO дополняется `set_artifact_budget(context, operation_id, ticket_version, total_bytes)`, `confirm_artifact_usage(context, operation_id, manifest: dict)` и `artifact_usage_proof(context, create_operation_id) -> dict | None`; proof имеет monotonic revision и полный набор surviving artifacts, произвольное подтверждение без phase evidence запрещено. DB helper methods частные; entrypoints не принимают пользовательский proof/charge.

- [ ] **Написать RED.** `test_two_cells_share_one_project_limit`: limit=1, две cell DB → один hold; `test_lost_confirm_retains_charge`; `test_absent_operation_not_refund_proof`: missing record → usage прежний; `test_cancel_then_queued_admit_never_executes`; `test_duplicate_create_uses_winning_ids`: оба клиента получают тот же ticket/operation/checkpoint; `test_refund_once_after_delete`; `test_staging_counts_before_write`; `test_budget_growth_conflict`: stale revision не меняет bytes; `test_cell_down_does_not_release`; `test_revert_delete_do_not_charge_second_checkpoint`.
- [ ] **Запустить RED:** `.tox/py311/bin/stestr run '^nova.tests.unit.checkpoints.test_quota'`; `.tox/functional/bin/stestr --test-path=./nova/tests/functional run '^nova.tests.functional.db.test_checkpoint_quota'`. Expected: новые quota assertions fail до реализации.
- [ ] **Реализовать API DB ledger §2.3.** Отдельные session/context objects для API и cell DB, без вложения facades. Уникальные key/operation IDs, exact replay сравнивает actor/hash, лимиты и charge обновляются под project row lock. Ticket хранит выигравшие IDs и точный cell. Ни expire, ни scheduler restart не снимают holds.
- [ ] **Реализовать confirm/settle/refund через durable cell proof.** Cancellation конфликтует с уже начатым эффектом; API refund не выполняется при unavailable/ambiguous cell. Proof revisions нельзя использовать дважды или от другого ticket. Для перезапуска между любыми двумя БД-переходами предусмотрен идемпотентный reconcile.
- [ ] **Запустить GREEN:** обе команды RED и `.tox/py311/bin/stestr run '^nova.tests.unit.(db.api.test_migrations|test_quota)'`. Expected: новые проверки проходят; старые Nova quotas не изменены, model-sync зелёный на трёх СУБД. `git diff --check` exit 0.
- [ ] **Commit:** stage перечисленные файлы; `git commit -m "feat: reserve checkpoint quotas across cells with durable tickets"`.

### Task 4: Guards API/DB/workers и защита identity

**Files:** Create `nova/checkpoints/guards.py`, `nova/tests/unit/checkpoints/{test_guards,test_guard_races}.py`. Modify `nova/context.py`, `nova/objects/checkpoint.py`, `nova/db/main/{api,checkpoints}.py`, `nova/compute/{api,manager}.py`, `nova/conductor/manager.py`, `nova/conductor/tasks/{live_migrate,migrate,cross_cell_migrate}.py`, `nova/api/openstack/compute/admin_actions.py`, `nova/tests/unit/{compute/test_api,compute/test_compute_mgr,api/openstack/compute/test_server_keypairs}.py`. Если конкретный effect path проходит через другой task module, добавить только доказанную точку в ledger и guard matrix, без общей реорганизации.

**Interfaces:** `guards.check(context, instance_uuid: str, action: str) -> None`; `guards.worker(context, instance_uuid: str, action: str)` — context manager с guarded Context; `guards.scope(context, scope: Scope)` — validated internal context; OVO `InstanceCheckpointGuard.acquire(context, instance_uuid: str, action: str) -> Scope`, `release(context, scope: Scope) -> None`, `handoff(context, scope: Scope, target: str) -> Scope`, `adopt(context, scope: Scope, target: str) -> Scope`, `get_by_instance_uuid(context, instance_uuid)`; DB `guard_mutation(context, instance_uuid: str, action: str, values: dict) -> None` вызывается **в текущей writer transaction**, не отдельной read transaction.

- [ ] **Написать RED matrix tests.** Вложенный helper проверяет каждый путь spec §8 при active operation/live checkpoint/без защиты. `test_delayed_worker_and_checkpoint_have_one_winner` останавливает worker между API dispatch и I/O: `assertFalse(checkpoint_admitted and driver_called)`; `test_other_worker_cannot_clear_claim`; `test_nested_remoted_save_uses_current_scope`; `test_api_io_claim_handoff_blocks_checkpoint`: pause между API side effect и RPC consumption не открывает admission; `test_forged_scope_admin_cannot_bypass`; `test_runtime_key_cas_races_admission` вызывает реальный `instance_extra_update_keypairs_if_current`; `test_reboot_allowed_between_checkpoints`; `test_topology_denied_until_last_checkpoint_deleted`.
- [ ] **Запустить RED:** `.tox/py311/bin/stestr run '^nova.tests.unit.checkpoints.test_guard'`. Expected: раньше оба конкурента могут пройти либо protected mutation проходит; это продуктовый RED, не только отсутствие decorator.
- [ ] **Встроить atomic DB protections.** Защитить Instance update/destroy, BDM create/update/destroy, VIF mutations, ShareMapping, InstanceExtra flavor/keys/device data, immutable часть info_cache/network identity. Сначала блокировать Instance, потом связанные rows. В update_keypairs_if_current сохранить прежний optimistic CAS после guard. Пользовательские metadata/tags и актуальные security groups не становятся источником ownership. Guard context сериализуется для OVO remoting, но каждый раз проверяется по текущей DB записи и action/phase; REST headers/body не задают его.
- [ ] **Встроить admission/worker scopes до первого эффекта.** API checks до quota/Neutron/Manila/RPC side effects; если API само делает I/O без атомарного pending state, оно также получает ordinary claim; ordinary worker claim до driver/attachment/migration I/O, включая late deliveries и обе стороны cross-cell migration. Сохранить existing state validation и callbacks; вложенные вызовы одного claim reentrant, чужой completion не может его освободить. При асинхронной передаче без другого durable pending barrier `handoff` переводит API claim в queued, `adopt` атомарно queued → running на закреплённом target; повторный consumer не принимает уже running claim. API finally не выпускает handed-off claim. Старый RPC без scope получает новый claim только при отсутствии текущего владельца. Для штатных многошаговых migration/callback paths матрица фиксирует уже существующий pending barrier и допустимый scope по шагу, без новых обязательных RPC arguments. При исключении claim удерживается; обычный reset не становится force-unlock. Зафиксировать таблицу entrypoint → первое I/O → persistent barrier в `doc/source/admin/instance-checkpoints.rst` (черновой раздел дополняет Task 9).
- [ ] **Запустить GREEN:** команда RED; `.tox/py311/bin/stestr run '^nova.tests.unit.(compute.test_api|compute.test_compute_mgr|conductor.test_conductor|api.openstack.compute.test_server_keypairs|api.openstack.compute.test_server_shares|api.openstack.compute.test_admin_actions|objects.test_block_device|objects.test_virtual_interface|objects.test_share_mapping|db.main.test_api)'`. Expected: API/worker/SQL races pass на MySQL/PostgreSQL, legacy поведение сохранено без checkpoint; `git diff --check` exit 0. Отдельно проверить реальные RPC context serializers, не ограничиваться прямыми Python вызовами.
- [ ] **Commit:** stage изменённые файлы и guard matrix; `git commit -m "feat: guard checkpoint instances against ordinary mutations"`.

### Task 5: Версионированный RPC и engine исполнения контракта

**Files:** Create `nova/checkpoints/{coordinator,compute}.py`, `nova/tests/unit/checkpoints/{test_rpc,test_execution}.py`, `nova/tests/unit/checkpoints/fake_executor.py`. Modify `nova/virt/driver.py`, `nova/compute/{manager,rpcapi}.py`, `nova/conductor/{api,manager,rpcapi}.py`, `nova/objects/service.py`, `nova/objects/checkpoint.py`, `nova/tests/unit/{compute/test_rpcapi,objects/test_service,virt/test_virt_drivers}.py`.

**Interfaces:** Driver hooks `checkpoint_probe(context, instance, manifest: dict) -> dict`, `checkpoint_execute(context, instance, intent: dict) -> dict`, `checkpoint_query(context, instance, intent: dict) -> dict`; defaults unsupported, no driver I/O. Compute RPC `checkpoint_probe(context, instance, manifest: dict) -> dict`, `checkpoint_execute(context, instance, operation_id: str, generation: int) -> None`, `checkpoint_query(context, instance, operation_id: str, generation: int) -> dict`. Conductor compute_task `checkpoint_probe(context, instance_uuid: str) -> dict`, `checkpoint_run(context, instance_uuid: str, operation_id: str, generation: int) -> None`. `Coordinator.probe(context, instance_uuid) -> Probe`; `Coordinator.run(context, instance_uuid, operation_id, generation) -> None`; `ComputeExecutor.execute(context, instance, operation_id, generation) -> None`, `query(...) -> dict`. Все источники effect intent — journal, не RPC user payload.

- [ ] **Написать RED.** `test_versions_and_old_endpoint_reject_before_admission`, `test_default_driver_unsupported_even_enabled`, `test_no_sql_access_from_compute` с DatabasePoisonFixture и настоящим OVO remoting, `test_execute_twice_calls_effect_once`, `test_lost_reply_queries_same_step_uuid`, `test_wrong_host_generation_profile_never_calls_driver`, `test_success_missing_disk_proof_is_unknown`, `test_quota_not_ready_never_claims`, `test_partial_revert_never_reaches_runtime_restore`.
- [ ] **Запустить RED:** `.tox/py311/bin/stestr run '^nova.tests.unit.checkpoints.(test_rpc|test_execution)'`. Expected: отсутствие protocol/claim/evidence validation проявляется проверяемым failure.
- [ ] **Реализовать version gates и live probe.** Compute RPC 6.5/service 69/history, compute_task 1.26; base conductor 3.0. Probe выполняется через оба живых endpoints до quota reserve. Unsupported method, exception, unknown contract/profile запрещают новый admission; `can_send_version` не заменяет ответ сервера. Существующие RPC не получают новые обязательные аргументы.
- [ ] **Реализовать phase engine.** Перед первым effect — durable StepIntent + claim; при повторе claim-нутого шага только query. Validate typed evidence каждого ресурса/phase/identity. Не превращать исключение после claim в доказанный no-effect. Между steps подтверждать ownership и обязательные предыдущие результаты; external I/O вне transactions. При известных ошибках create использовать отдельные cleanup steps, не переиспользовать effect UUID другой команды. Quota finalization вызывает Task 3 только после terminal proof.
- [ ] **Запустить GREEN:** команда RED и `.tox/py311/bin/stestr run '^nova.tests.unit.(compute.test_rpcapi|objects.test_service|virt.test_virt_drivers)'`. Expected: pass, новая service history согласована; fake зарегистрирован только в тестовом коде. `git diff --check` exit 0.
- [ ] **Commit:** stage перечисленные файлы; `git commit -m "feat: add gated checkpoint RPC and durable execution engine"`.

### Task 6: Cell-aware API, schemas, policy и idempotent admission

**Files:** Create `nova/checkpoints/api.py`, `nova/api/openstack/compute/{server_checkpoints,schemas/server_checkpoints,views/server_checkpoints}.py`, `nova/policies/server_checkpoints.py`, `nova/tests/unit/{checkpoints/test_api,api/openstack/compute/test_server_checkpoints,policies/test_server_checkpoints}.py`. Modify `nova/api/openstack/{api_version_request,compute/routes}.py`, `nova/api/openstack/compute/rest_api_version_history.rst`, `nova/policies/__init__.py`.

**Interfaces:** `API.capabilities(context, instance_uuid: str) -> dict`; `create(context, instance_uuid: str, body: dict) -> (InstanceCheckpoint, InstanceCheckpointOperation)`; `list(context, instance_uuid, marker=None, limit=100) -> list`; `get(context, instance_uuid, checkpoint_id) -> InstanceCheckpoint`; `revert(context, instance_uuid, checkpoint_id, request_id) -> InstanceCheckpointOperation`; `delete(...) -> InstanceCheckpointOperation`; `operation_get(context, instance_uuid, operation_id) -> InstanceCheckpointOperation`. Consumes Coordinator.probe/run, QuotaLedger, Task 2 OVO. Точный API spec §5; pagination максимум 1000, сортировка `(created_at, uuid)`, marker только из той же server/project области; неизвестные query fields → 400.

- [ ] **Написать RED.** WSGI tests всех семи routes spec: microversion 2.105 → 404, 2.106 → documented responses; create true/default accepted, false/quiesce/partial disks/extra fields → 400; exact replay после выключения flag → прежний operation; другой actor → 404 без захвата operation, другое тело/kind того же actor → 409; reader не создаёт; admin не обходит guards; wrong server marker не раскрывается; operation_show после delete работает, после удаления server → 404. `test_probe_then_db_fingerprint_change_aborts_ticket` отдельно меняет BDM/host/key/network identity.
- [ ] **Запустить RED:** `.tox/py311/bin/stestr run '^nova.tests.unit.(checkpoints.test_api|api.openstack.compute.test_server_checkpoints|policies.test_server_checkpoints)'`. Expected: новые маршруты/schema/policy/admission failures.
- [ ] **Реализовать routes/schema/views/policy.** API 2.106, spec payloads/headers. Правильный InstanceMapping и project target до cell access; unavailable/unmapped/cell0 не превращать в пустой успешный список. Whitelist response fields; внутренний scope/token/path/host/evidence не сериализовать. 202 только после durable cell operation и quota-ready (для create); отсутствие ответа клиенту не отменяет принятую операцию.
- [ ] **Реализовать admission/replay.** Read-only preflight прежде quota; C1 не вызывается в N1. Apply §2.3, adopting winning ticket IDs. Если cell admission отклонён — cancellation proof до refund; unknown RPC исход оставляет guard/hold. Existing checkpoint revert/delete не занимают новый ticket: после cell admit проверяют исходный confirmed create ticket и отмечают quota_ready своей новой operation. Чужой ticket/project/instance и released ticket отвергаются. Все retry выполняются после авторизации; raw RequestContext из manifest не принимается.
- [ ] **Запустить GREEN:** команда RED и `.tox/py311/bin/stestr run '^nova.tests.unit.(api.openstack.test_api_version_request|api.openstack.compute.test_api|policies.test_servers)'`. Expected: pass, стандартные старые версии сохранены. `git diff --check` exit 0.
- [ ] **Commit:** stage перечисленные файлы; `git commit -m "feat: expose cell-aware Nova checkpoint API"`.

### Task 7: Restart/reconciliation и защита от фоновых изменений

**Files:** Create `nova/checkpoints/recovery.py`, `nova/tests/unit/checkpoints/test_recovery.py`. Modify `nova/conductor/manager.py`, `nova/compute/manager.py`, `nova/db/main/{api,checkpoints}.py`, `nova/checkpoints/{coordinator,quota}.py`, `nova/objects/checkpoint.py`, `nova/tests/unit/{compute/test_compute_mgr,test_service}.py`.

**Interfaces:** `RecoveryManager.reconcile_cell(context, limit: int=100) -> dict`; `reconcile_instance(context, instance_uuid: str) -> dict`; `reconcile_quota(context, limit: int=100) -> dict`. Operation/List query methods используют keyset cursor `(updated_at, uuid)` и bounded batches; no global unbounded scatter/gather. Existing query methods Task 5 не создают новый effect claim для неизвестного шага. Periodic interval 60 seconds из config, работает при enabled=false.

- [ ] **Написать RED.** `test_restart_during_partial_revert_never_starts_guest`: `_init_instance`, `_sync_instance_power_state`, reboot/rescue polls и relevant external power events не вызывают driver.start/resume/reboot; `test_feature_off_preserves_recovery`; `test_lost_quota_confirm_repaired_once`; `test_cell_missing_operation_requires_cancel_tombstone`; `test_terminal_quota_refund_replays_once`; `test_archive_does_not_remove_guards_or_tombstones`; `test_expired_user_token_no_cinder_impersonation`: никакого Cinder вызова/сохранения токена; `test_stuck_ordinary_claim_is_not_timeout_released`.
- [ ] **Запустить RED:** `.tox/py311/bin/stestr run '^nova.tests.unit.checkpoints.test_recovery'`. Expected: фоновые действия/неверное освобождение воспроизводятся до guard integration.
- [ ] **Встроить recovery и фоновые guards.** Conductor init/periodic после DB readiness; compute init проверяет ownership до штатного восстановления/cleanup, query после driver readiness. Prepared intent можно доставить до первого durable claim; claimed unknown не выполняется снова. Записывать наблюдение безопасным кодом, не стирать task_state чужой операции. Не импортировать DB в compute recovery path.
- [ ] **Сохранить durable ownership при purge/обычных ошибках.** New journal не включать в generic archive/delete cascade; protected instance/related records не удалять. Ordinary claim не снимается admin reset без доказательства завершения эффекта; в N1 нет универсальной force-recovery команды. Query-only диагностика показывает владельца/фазу оператору без credentials; future adapter-specific repair остаётся отдельной работой.
- [ ] **Запустить GREEN:** команда RED и `.tox/py311/bin/stestr run '^nova.tests.unit.(compute.test_compute_mgr|test_service|db.main.test_api)'`. Expected: recovery и обычные restart/power tests pass; ни quota, ни guard не освобождены по времени. `git diff --check` exit 0.
- [ ] **Commit:** stage перечисленные файлы; `git commit -m "feat: reconcile checkpoint journals without destructive replay"`.

### Task 8: Сквозные две cells, SQL races и широкая регрессия

**Files:** Create `nova/tests/functional/test_checkpoints.py`, `nova/tests/functional/db/test_checkpoints.py`, `nova/tests/functional/api_sample_tests/test_checkpoints.py`, `nova/tests/functional/api_sample_tests/api_samples/server-checkpoints/v2.106/` (request/response templates). Использовать fake_executor Task 5; при необходимости вынести общие fixtures в `nova/tests/fixtures/checkpoints.py`, без production регистрации.

**Interfaces:** Consumes реальный REST API Task 6, Conductor/Compute managers и serializers Task 5, DB/quotas/guards/recovery. Produces reproducible lifecycle evidence и regression selection/count, не новый production API.

- [ ] **Написать RED functional assertions.** A/B → revert A → delete A при сохранном B: стабильные server/volume/port UUID, ожидаемые fake RAM/disk markers, snapshots независимы. Отдельно два server в двух cells одного project, последний quota slot конкурентен; down cell; потеря RPC до/после каждого claim; old compute/conductor; late worker; revoked authorization/forged scope; cleanup остатков учитывается до подтверждённого удаления. Samples сверяются с настоящим ответом WSGI, host/path/token отсутствуют.
- [ ] **Запустить RED:** `.tox/functional/bin/stestr --test-path=./nova/tests/functional run '^nova.tests.functional.(test_checkpoints|db.test_checkpoints|db.test_checkpoint_quota|api_sample_tests.test_checkpoints)'`. Expected: выявленный несовпадающий end-to-end контракт; если всё сразу pass, зафиксировать отсутствие нового дефекта и не создавать искусственный product RED.
- [ ] **Исправить только обнаруженные несовпадения N1.** Реальные две cell DB и API DB; реальный Nova RPC serializer/fake transport и настоящие managers. Не подменять ключевые SQL transactions mock-ответами; fake только для N2 effects/внешних сервисов. Сохранять durable SQL state при пересоздании managers, query восстанавливает исходный step UUID.
- [ ] **Запустить GREEN functional** той же командой. Expected: все новые сценарии pass, реальные MySQL/PostgreSQL races не skipped. Проверить counts и отсутствие дважды исполненного эффекта при RPC replay.
- [ ] **Выполнить широкую регрессию:** `.tox/py311/bin/stestr run`; `TEST_OSPROFILER=1 .tox/py311/bin/stestr run --combine --no-discover 'nova.tests.unit.test_profiler'`; `.tox/functional/bin/stestr --test-path=./nova/tests/functional run`. Expected: ноль новых failures/errors, исходные skips перечислены, все выбранные tests отчитались. Не объявлять success после worker crash/timeout. Исходные ошибки подтверждать на baseline; новые исправлять и повторять затронутый набор, затем общий согласованный gate. `git diff --check` exit 0.
- [ ] **Commit:** stage новые tests и проверенные fixes; `git commit -m "test: exercise checkpoint lifecycle across Nova cells"`.

### Task 9: Документация, независимое review и проверяемый пакет

**Files:** Create/complete `doc/source/admin/instance-checkpoints.rst`, `api-ref/source/server-checkpoints.inc`, `api-ref/source/samples/server-checkpoints/` (samples), `releasenotes/notes/instance-checkpoints-n1-4a8c2d7e9101.yaml`. Modify `doc/source/admin/index.rst`, `api-ref/source/{index.rst,parameters.yaml}`. External delivery: `deliverables/nova-checkpoint-model-n1/{README,VALIDATION,RULINGS,REVIEW,REVIEW-RESOLUTION}.md`, patches/series/hash/tree files и verifier; исходный ZIP отдельно не дублировать.

**Interfaces:** Consumes completed Tasks 1–8 и их ledger. Produces docs/API samples, исходный и конечный tree, воспроизводимую серию патчей, фактический validation report. Публичный продуктовый интерфейс не расширяется.

- [ ] **Дополнить документацию по проверенным результатам.** Чётко обозначить N1 unsupported без N2, числа версий, limits 8/100/0, DB migration order и всех писателей, отсутствие автоматического unlock, two-DB cancellation, quarantine ordinary claims, список entrypoint barriers, отсутствие token impersonation, границы Neutron/libvirt/SAN и Cinder N2 auth. API примеры соответствуют Task 8, не вручную выдуманным состояниям.
- [ ] **Проверить документы и весь diff.** `tox -e genconfig,genpolicy,api-ref,docs,pep8`; `git diff --check`. Expected: новые docs/code checks зелёные. `pep8` использует pre-commit с автоисправлениями: сначала сравнить на disposable baseline, не принимать массовые несвязанные изменения; все изменённые N1 файлы должны пройти. Если baseline сам красный, сохранить точный diff диагностик и не объявлять весь pep8 pass. Команды docs используют исходные зависимости своего tox env; tool-only workaround требует записанного решения, не переписывания requirements.
- [ ] **Commit delivery documentation:** stage перечисленные repository docs; `git commit -m "docs: document Nova checkpoint model and recovery boundaries"`. Expected: чистая feature branch перед review.
- [ ] **Провести единственный fresh whole-branch review Native.** Пакет reviewer содержит base..HEAD, spec, этот план, пять Review Focus и все ledger rulings. Reviewer рассматривает DB/RPC/authorization/ordinary paths, не только tests. Re-grade findings; Critical/Important — один fix pass с observed RED→GREEN и общим зелёным gate, Minor — явно отложить, каждый Declined to judge — явное решение с ценой ошибки. Без повторного reviewer. Исправления закоммитить.
- [ ] **Собрать и проверить патчи.** Generate format-patch base..final HEAD; verifier проверяет SHA исходного ZIP, извлечение без path traversal с корректными symlinks/modes, исходный tree, `git apply --check --index`/apply каждой patch и итоговый tree. Run `python3 deliverables/nova-checkpoint-model-n1/verify_patches.py nova-pvs.zip` из корня workspace. Expected: PASS с точным final tree; `shasum -a 256 -c SHA256SUMS` из bundle — все файлы OK. Verifier не обращается к runtime services.
- [ ] **Завершить локальную передачу.** В отчёте указать точные команды/counts/skips и пределы проверки; сохранить review и его resolution раздельно. Перенести evidence до удаления только scratch текущего плана; удалить лишь собственную disposable test environment. Сохранить checkout/ветку/архивы, не merge/push/deploy. Финальная формулировка: N1 готов локально, production RAM checkpoints требуют N2/D/V1.

## 4. Проверка связности и покрытия плана

| Требование spec | Реализующая задача |
|---|---|
| Контракт/default unsupported/config/profile | 1, 5 |
| Cell journal, identity, idempotency, cancellation tombstones | 2 |
| API DB quotas, две cells, late admission/refund | 3, 8 |
| Конкуренция API/DB/worker, ключи форка, shares/network/BDM | 4 |
| Compute без SQL, version gates, durable step и evidence | 2, 5 |
| Все REST routes, RBAC, replay/pagination/sanitization | 6 |
| Feature-off, restart/power sync/purge, query-only recovery | 7 |
| Fake A/B, partial failures, старые endpoints, real SQL | 8 |
| Документы, patch reproduction, review и честные границы | 9 |
| RAM/libvirt/Cinder orchestration/vendor runtime | Осознанно N2/D1/D2/V1; N1 не заявляет поддержку |

Межзадачные зависимости: 1 → 2 → 3; 2/3 → 4; 1–4 → 5; 2–5 → 6; 2–6 → 7; 1–7 → 8 → 9. Контракты и названия едины по §2 и Interfaces каждой задачи. Семь cell tables вместо шести логических сущностей — явная детализация требуемой cancellation, не новый пользовательский ресурс. Квотный ledger не заменяет штатные Nova quotas и не притворяется единой transaction двух БД.

**Статус self-review:** сверены spec coverage, межзадачные signatures, точные версии/limits, пять Review Focus с тестами и существующие пути форка. Команды тестов являются планом проверки, не результатом запуска. Никакой dependency baseline Nova пока не объявляется зелёным.

**Передача:** после рассмотрения этого плана — непрерывное Native-исполнение Tasks 1–9, одно review в конце. Способ исполнения уже выбран; повторно выбирать его не нужно.
