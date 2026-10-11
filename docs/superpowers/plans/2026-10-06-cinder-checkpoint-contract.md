# Cinder Checkpoint Contract C1 Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Реализовать в предоставленном форке Cinder проверяемый сервисный контракт снимков для RAM checkpoints Nova: capabilities, исключительное владение томами, durable operations, native-only execution и recovery без слепого повтора.

**Architecture:** Nova будет координировать ВМ и RAM, а Cinder — собственные ресурсы и вызовы драйверов. Новый пакет `cinder.checkpoints` хранит защищённые связи и журнал в общей Cinder DB; `cinder-volume` выполняет команды и проверяет результат через новый driver contract. По умолчанию функция отключена, существующие драйверы возвращают отсутствие поддержки; тестовый драйвер доступен только тестам.

**Tech Stack:** Python 3.11, существующие SQLAlchemy/Alembic, oslo.versionedobjects, oslo.messaging, oslo.policy, oslo.config, Keystone service tokens, unittest/stestr/tox из Cinder; новые production-зависимости не добавляются.

**Spec:** [Согласованная общая спецификация](../specs/2026-10-06-instance-checkpoints-dorado-tatlin-design.md), прежде всего разделы 3–5, 8–11. Этот план реализует только C1; N1/N2, D1/D2 и V1 остаются отдельными пакетами.

## Global Constraints

- «Nova не обращается к REST API массивов напрямую.» В C1 вообще не изменять Nova.
- «Общий fallback с копированием для новой функции запрещён.» Существующий обычный Cinder `revert` без checkpoint-защиты сохраняет поведение.
- «Истечение lease или timeout не снимает защиту автоматически». Нет force-unlock или unlock по возрасту записи.
- «Изменяемая пользователем metadata не служит источником владения, блокировки или доказательством возможности отката.»
- «Cinder attachment UUID может измениться при переподключении.» Volume UUID остаётся исходным.
- «Функция по умолчанию отключена.» Отключение новых операций не отключает guard, read, cleanup и recovery существующих.
- «Unit/fake integration tests подтверждают код и контракт.» Поддержку Dorado/TATLIN C1 не объявляет.
- Исходная база: `cinder-pvs.zip`, SHA-256 `a635f0e994c786c27c729d07e14eeaa8c92a887652f2cb69daca808d8754eaed`; ZIP revision comment `324314cb487365bd895071c8e786b855ac2efa51` не является локальной Git-историей.
- Рабочая копия при исполнении: `work/cinder-checkpoints/`, ветка `feature/cinder-checkpoint-contract`. `sources/cinder-pvs/` остаётся проверяемой исходной базой; не менять архивы и не подключаться к СХД.

## Review Focus

1. **Потеря ответа после реального эффекта:** число destructive calls остаётся равным одному, release запрещён до доказанного исхода — Task 5/7.
2. **Гонка прямого attachment/retype с reserve:** ровно одна сторона получает разрешение на изменение; проигравшая не вызывает driver — Task 2/3.
3. **Подделка operation UUID или сервисной роли tenant:** чужая операция и bypass guards недоступны даже администратору без доверенного service token — Task 4/6.
4. **Удалённый checkpoint и повтор старого request ID:** tombstone возвращает прежний результат, новый snapshot/operation не создаётся — Task 2/8.
5. **Частично обновлённые сервисы или выключенная функция:** новый reserve отклоняется до эффектов, защита уже существующих ресурсов работает — Task 5/7/8.

---

## 1. Решения C1, фиксируемые этим планом

Это детализация общего контракта для реализации и review плана, а не утверждение об уже существующем API.

### 1.1. Версии, настройки и доступ

- В этой зафиксированной базе занять Cinder microversion **3.72**, Volume RPC **3.21**, object manifest **1.40**. Сейчас в архиве максимумы `3.71`, `3.20`, `1.39`. Значения — частное расширение форка, не заявление о совместимости с одноимёнными upstream microversions. При изменении исходной базы проверить коллизии до исполнения.
- Новые OVO `CheckpointInstance`, `CheckpointOperation`, `CheckpointResource`, `CheckpointStep`, `CheckpointVolumeLock` и их List-классы получают `VERSION = '1.0'`.
- `[checkpoint] enabled = false`; `nova_service_user_ids = []`; `reconcile_interval = 60` секунд; `max_volumes_per_operation = 64`.
- `enabled` ограничивает новые `create` reservations. Read, release, reconcile, delete и revert уже существующих checkpoint доступны при прежней авторизации; guard действует независимо от флага.
- Авторизация: существующий `volume.API.is_service_request(context)` **и** `context.service_user_id` из `nova_service_user_ids`, плюс project ownership всех ресурсов и policy. Пустой список запрещает новый сервисный доступ; пользовательская роль `service` сама по себе не подходит.
- Policy `checkpoint:read` и `checkpoint:operate` допускает project member/admin своей области, но не отменяет обязательную проверку service identity. Права на server остаются обязанностью Nova.
- Включение требует обновления **всех** API и сервисов, способных изменить защищённые ресурсы. RPC gate не обнаруживает старый API-процесс. До завершения обновления функция остаётся выключенной и checkpoints отсутствуют; downgrade допускается только после завершения операций и cleanup всех ссылок.

### 1.2. Сервисный REST API

Все новые routes используют `/v3/{project_id}` и требуют 3.72. Tenant-facing Nova API будет добавлен отдельно.

| Route | Контракт |
|---|---|
| `GET /volumes/{id}/checkpoint-capabilities` | `200`, capability для конкретного volume и backend; неизвестный driver → `supported=false` |
| `POST /checkpoint-operations` | Создать или вернуть reservation по request ID; `202` с operation UUID, generation и ресурсами |
| `GET /checkpoint-operations/{id}` | `200`, состояние, ресурсы и безопасные результаты шагов |
| `GET /checkpoint-operations` | Project-scoped список; фильтры `checkpoint_id`, `instance_uuid`, `status`; стандартная pagination |
| `POST /checkpoint-operations/{id}/action` | Ровно одно действие: `execute`, `cleanup`, `reconcile` или `release`; все с `generation`; ответ `202` с текущей operation |

Request reserve:

```json
{
  "checkpoint_operation": {
    "request_id": "11111111-1111-4111-8111-111111111111",
    "checkpoint_id": "22222222-2222-4222-8222-222222222222",
    "instance_uuid": "33333333-3333-4333-8333-333333333333",
    "kind": "create",
    "volume_ids": ["44444444-4444-4444-8444-444444444444"]
  }
}
```

`kind` — `create|revert|delete`. Список UUID непустой, уникальный, максимум 64. Для revert/delete он обязан точно совпадать с сохранённым набором checkpoint; snapshot IDs берутся только из защищённых записей. Для create проверяется, что тома принадлежат тому же проекту и имеют единственное attachment к данному instance. При reserve revert/delete допустимы отсутствие attachments или единственное attachment к записанному instance; исходные attachment IDs сохраняются в шагах операции. Multiattach/group/replication/migration/transitional resources в C1 отклоняются. Отсутствие group support не препятствует индивидуальным native snapshots.

Существование нескольких checkpoints одних томов разрешено только для одного project/instance и неизменного набора томов. Одна активная операция удерживает весь набор; длительные ссылки других checkpoints не считаются конкурирующей операцией.

Action body: `{"execute":{"generation":1}}`. `cleanup` и `reconcile` имеют ту же форму. `release` дополнительно содержит `outcome: "commit"|"abort"`.

- `execute`: durable dispatch native операций для всех ресурсов. Для create Nova обязана предварительно остановить записи; Cinder проверяет attachment ownership, но не выдаёт эту проверку за доказательство остановленного QEMU. Revert дополнительно требует `available` и отсутствия attachments всех томов. Delete допускает текущие attachments, но только независимо удаляемые snapshots.
- `cleanup`: только для незавершённого create; удаляет доказанно созданные snapshots после выяснения исхода всех create. Не делает автоматический обратный revert.
- `reconcile`: query driver state без повтора эффекта и без автоматического release.
- `release(commit)`: разрешён только после подтверждённого результата всех шагов и завершённого со стороны Nova этапа; при revert также проверяются attachments к исходному instance. Снимает временные locks, сохраняет живые checkpoint references.
- `release(abort)`: только до эффектов или после подтверждённого cleanup create. После частичного revert/неопределённого исхода возвращает `409`.

Повтор действия с теми же параметрами возвращает тот же результат; другая generation/outcome даёт `409`, кроме точного replay уже завершённого действия. В C1 Cinder назначает `generation=1` каждой новой operation UUID и не предоставляет передачу незавершённой операции новому владельцу. Все проверки используют пару `(operation_id, generation)`: старый UUID не получает доступ к новой reservation, даже если значение generation совпадает. Смена поколения внутри operation предусмотрена форматом, но не реализуется в C1.

Ошибки: `400` schema, `403` нет доверенного service identity/policy, `404` отсутствующий/чужой ресурс, `409` conflict/unsupported/stale generation, штатная quota-ошибка `413`, `503` недоступный backend. Неподдержанная версия не должна создавать operation, lock или Snapshot.

### 1.3. Persistence и идемпотентность

Пять новых таблиц используют стандартные timestamps Cinder и совместимые SQL-типы:

| Таблица / OVO | Поля и ограничения |
|---|---|
| `checkpoint_instances` / `CheckpointInstance` | `instance_uuid` primary key, `project_id`, `volume_ids_json`, `active_operation_id`; сериализует операции одной ВМ, включая запросы с непересекающимися наборами томов |
| `checkpoint_operations` / `CheckpointOperation` | `id`, `project_id`, `user_id`, `service_user_id`, `request_id`, `checkpoint_id`, `create_checkpoint_id`, `instance_uuid`, `kind`, `payload_hash`, `generation`, `status`, `release_outcome`, `error_code`; unique `(project_id, request_id)`, unique nullable `create_checkpoint_id` |
| `checkpoint_resources` / `CheckpointResource` | `id`, `project_id`, `instance_uuid`, `checkpoint_id`, `volume_id`, `snapshot_id`, `volume_size`, `backend_profile`, `capabilities_json`, `status`; unique `(checkpoint_id, volume_id)`, unique snapshot ID |
| `checkpoint_steps` / `CheckpointStep` | `id`, `operation_id`, `resource_id`, `phase`, `generation`, `status`, `executor_id`, `backend_reference`, `error_code`, `quota_reservations_json`, `original_attachment_ids_json`; unique `(operation_id, resource_id, phase)` |
| `checkpoint_volume_locks` / `CheckpointVolumeLock` | `volume_id` primary key, `operation_id`, `generation`; строка существует только при активном владении |

`phase = snapshot|revert|delete_snapshot`; create cleanup использует отдельный delete-step той же operation. Driver получает **step UUID** как стабильный идентификатор побочного эффекта. Ресурсы checkpoint доступны новой revert/delete operation без копирования manifest из запроса.

Operation states: `reserved`, `running`, `succeeded`, `failed`, `recovery_required`, `released`. Step states: `prepared`, `dispatching`, `pending`, `succeeded`, `failed_no_effect`, `unknown`. Ошибка вызова после начала dispatch по умолчанию `unknown`, а не доказательство отсутствия эффекта. Phase и итог освобождения сохраняются после `released`.

Tombstones операций и удалённых resource references в C1 не удаляются автоматически; quota освобождается отдельно от журнала. Reuse request UUID с тем же canonical payload возвращает прежнюю запись, с другим payload — `409`. Canonical payload: JSON с сортировкой ключей и нормализованным сортированным набором UUID; поля request ID и service identity учитываются по зафиксированным значениям записи. Новый пользователь/сервис не получает доступ к старой operation только по знанию ID.

Повторное создание того же checkpoint UUID с новым request ID запрещено, в том числе после удаления checkpoint. При точном replay исходного request ID возвращается tombstone. Проверка всего набора защищает также от попытки дополнить существующий checkpoint другим volume.

`create_checkpoint_id` равен checkpoint UUID только у create operation, у revert/delete — NULL. Уникальность действует и для tombstones; UUID checkpoint нельзя повторно использовать для другой ВМ. Строка `checkpoint_instances` сохраняется после удаления checkpoints. Изменение её набора томов допускается только при отсутствии живых references и незавершённых операций. При первом insert гонка разрешается PK constraint и повтором короткой транзакции после rollback.

Reservation всего набора выполняется одной короткой DB writer transaction: сначала блокировка строки instance, затем строк volumes в порядке UUID, проверка состояния и pending работ, ссылки и уникальные locks. Это закрывает одновременные reservations одной ВМ с непересекающимися наборами. Release использует тот же порядок. SQL-транзакция не удерживается во время driver/RPC вызовов. Атомарные admission-проверки обычных mutating API блокируют те же строки volumes в порядке UUID и не захватывают instance после volume; отдельный `if not locked` перед записью не обеспечивает корректность.

Штатные snapshot quotas резервируются до dispatch, их IDs сохраняются в журнале. Snapshot UUID выделяется и объект Cinder сохраняется до обращения к driver. Commit/rollback квот сверяется с этими объектами при restart: нельзя повторно начислить или освободить quota из-за replay. При конфликте admission новые снимки/locks/квоты не должны остаться; запись безопасного отказа для дедупликации допустима.

### 1.4. Python-интерфейсы между задачами

Тип `Context` означает `cinder.context.RequestContext`; `Operation` и `Resource` — новые OVO. Сигнатуры ниже обязательны для всех задач.

| Модуль | Публичные интерфейсы |
|---|---|
| `cinder/checkpoints/contracts.py` | `unsupported_capabilities(reason: str) -> dict`; `validate_capabilities(value: dict) -> dict`; `canonical_payload_hash(payload: dict) -> str`; `validate_driver_result(value: dict) -> dict` |
| `cinder/volume/driver.py` | `get_checkpoint_capabilities(context, volume) -> dict`; `checkpoint_create_snapshot(context, volume, snapshot, operation_id: str) -> dict`; `checkpoint_revert_to_snapshot(context, volume, snapshot, operation_id: str) -> dict`; `checkpoint_delete_snapshot(context, volume, snapshot, operation_id: str) -> dict`; `checkpoint_query_operation(context, volume, snapshot, operation_id: str, action: str) -> dict` |
| `cinder/checkpoints/api.py` | `API.capabilities(context, volume_id: str) -> dict`; `reserve(context, payload: dict) -> Operation`; `get(context, operation_id: str) -> Operation`; `list(context, filters: dict, marker=None, limit=None) -> list`; `action(context, operation_id: str, name: str, generation: int, outcome=None) -> Operation` |
| `cinder/checkpoints/guards.py` | `require_service(context) -> None`; `check_volume(context, volume_id: str, action: str) -> None`; `check_snapshot(context, snapshot_id: str, action: str) -> None`; `validate_operation_scope(context, operation_id: str, generation: int, volume_id: str, action: str) -> None` |
| `cinder/checkpoints/manager.py` | `CheckpointManager(volume_manager)`; `.capabilities(context, volume_id: str) -> dict`; `.execute(context, step_id: str, generation: int) -> None`; `.reconcile(context, step_id: str, generation: int) -> None` |
| `cinder/db/api.py` + backend | `checkpoint_reserve(context, values: dict, volume_ids: list[str]) -> dict`; `checkpoint_get(context, operation_id: str) -> dict`; `checkpoint_get_all(context, filters: dict, marker=None, limit=None) -> list`; `checkpoint_step_claim(context, step_id: str, generation: int, executor_id: str) -> bool`; `checkpoint_step_finish(context, step_id: str, generation: int, result: dict) -> dict`; `checkpoint_release(context, operation_id: str, generation: int, outcome: str) -> dict`; `checkpoint_resource_guard(context, volume_id: str, action: str) -> None` |

Driver capabilities включают все поля раздела 4 spec. `snapshot_semantics=unknown` и отсутствие обязательного поля не дают поддержку. Групповые snapshots необязательны. Driver result: `status=succeeded|pending|failed_no_effect|unknown`, optional `backend_reference`, `error_code`; любые исключения/некорректный result после dispatch переводятся в `unknown`. Внешние ответы не содержат `backend_reference` и сырые driver exception messages.

## 2. Структура файлов

Все пути Task 1–8 ниже относительны `work/cinder-checkpoints/`.

| Файлы | Ответственность |
|---|---|
| `cinder/checkpoints/{__init__,contracts,config,api,guards,manager}.py` | Контракт, настройки, admission и исполнение; отдельный процесс не создаётся |
| `cinder/objects/checkpoint.py`, существующие `objects/{__init__,base}.py` | Пять новых объектов/List и manifest 1.40 |
| `cinder/db/{api.py,sqlalchemy/api.py,sqlalchemy/models.py}` | DB atomic reserve, claims, guards, journal |
| `cinder/db/migrations/versions/7fa82e10c601_add_checkpoint_contract.py` | Новая additive migration, parent `9c74c1c6971f`; существующие таблицы не переименовывать |
| `cinder/api/{schemas/checkpoints.py,v3/checkpoints.py,v3/views/checkpoints.py,v3/router.py}` | Schema, controller, безопасные ответы и routes |
| `cinder/policies/checkpoints.py`, `cinder/policies/__init__.py`, `cinder/context.py` | Policy и проверенный operation scope для RPC |
| `cinder/volume/{api,manager,rpcapi,driver}.py` | Интеграция нового пути и защита старых операций |
| `cinder/{group/api,transfer/api,backup/api}.py`, `cinder/api/contrib/{volume_actions,admin_actions}.py` | Закрытие путей обхода guards |
| `cinder/api/v3/attachments.py`, `cinder/api/schemas/attachments.py` | Проверенный scope для переподключения |
| API version/history, `cinder/opts.py`, tests, api-ref, releasenote | Объявление и проверка публичного контракта |

Не создавать предполагаемые `cinder/attachments/api.py` или `manager.py`: в этой базе attachment logic находится в `cinder/volume/api.py` и `manager.py`, несмотря на имена test directories.

## 3. Исполнение по задачам

### Task 1: Capability и native-only driver contract

**Files:** Create `cinder/checkpoints/{__init__,contracts,config}.py`, `cinder/tests/unit/checkpoints/{__init__,test_contracts,fake_driver}.py`; Modify `cinder/volume/driver.py`, `cinder/exception.py`, `cinder/opts.py`; Test existing `cinder/tests/unit/volume/test_driver.py`.

**Interfaces:** Produces contracts/config/driver signatures из §1.4 и `FakeCheckpointDriver` с программируемыми result/query outcomes и счётчиком native calls. Fake driver импортируется только тестами. Исключения: `CheckpointConflict`, `CheckpointNotSupported`, `CheckpointOperationNotFound`, `CheckpointServiceForbidden` с безопасными code/reason.

- [x] **Подготовить изолированную рабочую копию по using-git-worktrees и проверить базу.** Если Git отсутствует, импортировать архив в `work/cinder-checkpoints`, создать локальный baseline commit и ветку; не выдавать его SHA за ZIP revision. На Linux/Python 3.11 выполнить baseline `tox -e py311 -- cinder.tests.unit.volume.test_driver`; использовать constraints 2025.1 из tox.ini. Для установки архивного дерева без release metadata допустим только тестовый `PBR_VERSION=0.0.0`. Если baseline не стартует из-за окружения, исправить окружение и записать причину до RED, а не считать это тестом функции.
- [x] **Написать RED-тесты defaults/schema/семантики.** В `test_default_driver_is_unsupported`: `self.assertFalse(caps['native_snapshot'])`; в `test_unknown_semantics_rejected`: `self.assertRaises(CheckpointNotSupported, validate_capabilities, caps_unknown)`; в `test_row_remains_row`: `self.assertEqual('row', result['snapshot_semantics'])`. Проверить обязательные booleans, неправильные типы и стабильность hash при перестановке volume IDs.
- [x] **Запустить RED:** `tox -e py311 -- cinder.tests.unit.checkpoints.test_contracts`. Ожидается отсутствие новых interfaces/неверное поведение, а не ошибка зависимостей.
- [x] **Реализовать interfaces §1.4, config §1.1 и test fake.** Defaults всех реальных drivers неподдержанные; native methods базового driver выбрасывают `CheckpointNotSupported`. Не вызывать legacy `revert_to_snapshot` или generic copy. Список методов snapshot/revert/delete/query у fake записывает step UUID и вызовы.
- [x] **Запустить GREEN:** `tox -e py311 -- 'cinder.tests.unit.(checkpoints.test_contracts|volume.test_driver)'`; ожидается 0 failures. Перегенерировать opts штатным `tox -e genopts`, проверить, что изменения относятся к checkpoint config.
- [x] **Commit:** `feat: define native Cinder checkpoint driver contract`.

### Task 2: Durable journal, OVO и атомарное владение набором томов

**Files:** Create migration и `cinder/objects/checkpoint.py`; Create `cinder/tests/unit/checkpoints/test_db.py`, `cinder/tests/unit/objects/test_checkpoint.py`; Modify `cinder/db/{api.py,sqlalchemy/api.py,sqlalchemy/models.py}`, `cinder/objects/{__init__,base}.py`, `cinder/tests/unit/{db/test_migrations.py,objects/test_objects.py}`.

**Interfaces:** Consumes contracts/hash. Produces пять OVO/List и DB interfaces §1.4; OVO используют `create/save/get_by_id/get_all` по существующему образцу. Guard helper вызывается внутри той же writer transaction, где выполняется admission/изменение ресурса.

- [x] **Написать RED-тесты persistence, migration и гонок.** `test_reserve_is_all_or_nothing`: при занятом втором volume `self.assertEqual([], newly_acquired_locks)`; `test_two_reservations_have_one_winner`: после двух транзакций `self.assertEqual(1, winner_count)`; `test_disjoint_sets_for_one_instance_have_one_winner`: один согласованный набор и одна operation; `test_replay_after_release_returns_tombstone`: `self.assertEqual(first.id, replay.id)`; `test_old_generation_cannot_release`: `self.assertRaises(CheckpointConflict, release_old)`; payload/service/project mismatch даёт отказ. Проверить pending обычный snapshot/attachment как конфликт admission.
- [x] **Запустить RED:** `tox -e py311 -- 'cinder.tests.unit.(checkpoints.test_db|objects.test_checkpoint|db.test_migrations)'`.
- [x] **Реализовать пять таблиц и DB transitions §1.3.** FK/unique constraints не теряют действие при soft delete. `step_claim` — compare-and-set только `prepared → dispatching`; повтор claim false, native dispatch не повторяется. Reconciliation может зафиксировать подтверждённый исход `unknown`, но не возвращает step в `prepared` автоматически. Cinder назначает generation согласно §1.2; проверить отказ старому operation UUID при новой reservation и запрет повторного создания checkpoint UUID другим request.
- [x] **Реализовать OVO 1.0 и manifest 1.40.** Использовать `fields.UUIDField`/versioned serialization; сохранить старые Volume/Snapshot versions, не добавлять фиктивные поля в их metadata. Зарегистрировать новые классы через `register_all`, обновить hashes тестов OVO после проверки фактических полей.
- [x] **Запустить GREEN и реальные DB проверки:** та же команда; дополнительно migration/model-sync и concurrency на MySQL и PostgreSQL через имеющиеся oslo.db fixtures, с явно настроенными disposable DB. Пропуск этих tests считать незакрытой проверкой, SQLite не доказывает row-lock поведение. Старые записи volume/snapshot сохраняются после upgrade.
- [x] **Commit:** `feat: persist checkpoint operations and resource reservations`.

### Task 3: Защита от обхода через обычные Cinder операции

**Files:** Create `cinder/checkpoints/guards.py`, `cinder/tests/unit/checkpoints/test_guards.py`; Modify volume API/manager, DB backend admission из Task 2, `cinder/{group/api,transfer/api,backup/api}.py`, `cinder/api/contrib/{volume_actions,admin_actions}.py`.

**Interfaces:** Consumes durable references/locks. Produces `check_volume`, `check_snapshot`, `validate_operation_scope` из §1.4. На незатронутых ресурсах поведение прежнее. Service validation окончательно подключается Task 4; до неё scope bypass закрыт.

- [x] **Написать RED параметризованную матрицу обходов.** `test_protected_resource_rejects_legacy_mutation`: для каждого действия ниже `self.assertRaises(CheckpointConflict, invoke)` и `driver.assert_not_called()`. `test_guard_applies_when_feature_disabled`: false config не меняет отказ. `test_reserve_races_attachment`: два порядка транзакций, только один разрешённый эффект.
- [x] **Запустить RED:** `tox -e py311 -- cinder.tests.unit.checkpoints.test_guards`.
- [x] **Встроить admission и worker checks по матрице.** Check и соответствующая DB mutation должны сериализоваться с reserve на той же строке volume; обычная pending operation видима checkpoint admission до RPC. Повторная проверка manager закрывает запоздавшие сообщения. Не держать DB transaction во время backend I/O.

| Объект | Запрещённые чужие изменения при защите |
|---|---|
| Volume | delete/force-delete/unmanage, extend, retype, migrate/completion, revert/reimage, transfer create/accept, backup restore, attach/detach и legacy initialize/terminate/reserve/unreserve |
| Snapshot | delete/force/unmanage/reset-status, создание volume/clone из защищённого snapshot, export/initialize-connection snapshot |
| Group/replication | Добавление защищённого volume в group, group snapshot/delete/restore, volume/group failover/retype; существующий group/replicated volume не допускается в C1 |
| Admin обход | reset-status, force-detach, bypass по `is_admin`; admin без подтверждённой operation не снимает защиту |
| На время активной operation | Дополнительно обычные snapshot/create-clone/backup и upload-to-image, меняющие подключения/статусы исходного volume |

- [x] **Проверить чтение и стандартные операции вне защиты.** `GET/list` работают; обычные snapshot/revert/attach незатронутого volume не получают новых ограничений. Между operations не блокировать read-only backup исходного volume, если он не меняет защищённые attachments и не использует защищённый snapshot; конкретные mutating пути из таблицы остаются закрыты.
- [x] **Запустить GREEN:** `tox -e py311 -- 'cinder.tests.unit.(checkpoints.test_guards|attachments|api.contrib.test_volume_actions|policies.test_volume_actions)'`, плюс tests затронутых transfer/group/backup путей. До закрытия всей матрицы не включать feature.
- [x] **Commit:** `feat: guard checkpoint resources across Cinder mutation paths`.

### Task 4: Project/service authorization и REST reservation API

**Files:** Create `cinder/checkpoints/api.py`, `cinder/api/{schemas/checkpoints.py,v3/checkpoints.py,v3/views/checkpoints.py}`, `cinder/policies/checkpoints.py`, `cinder/tests/unit/{api/v3/test_checkpoints.py,policies/test_checkpoints.py,checkpoints/test_api.py}`; Modify policy registry, router, `cinder/api/{microversions.py,openstack/api_version_request.py,openstack/rest_api_version_history.rst}`, guards; include version API tests touched by maximum bump.

**Interfaces:** Consumes DB/OVO/guards; produces API §1.4 and routes §1.2. `reserve` возвращает operation только после preflight и all-volume admission. Capability query обращается к конкретному backend через RPC Task 5; tests этого Task mock RPC boundary.

- [x] **Написать RED API/policy tests.** `test_tenant_service_role_is_not_service_token`: `self.assertEqual(403, response.status_int)`; `test_wrong_project_is_not_disclosed`: 404; `test_reused_request_with_changed_volumes`: 409; `test_unknown_fields_rejected`: 400; `test_duplicate_uuid_rejected`: 400. Проверить роли/default policy, allowlist service user, 3.71, feature false, empty/65 volumes и неправильную generation без DB/native effects.
- [x] **Запустить RED:** `tox -e py311 -- 'cinder.tests.unit.(api.v3.test_checkpoints|policies.test_checkpoints|checkpoints.test_api)'`.
- [x] **Реализовать controller/schema/view и проверки §1.1–1.2.** Проверять ownership до подробностей о capability/состоянии, не доверять project ID в JSON. Права service token считывать только из контекста после штатного Keystone middleware. Allowlist owner записывается в operation; read/replay также проверяют владельца. Все native IDs, backend_reference и сырые exceptions исключить из публичного view.
- [x] **Связать quotas/Snapshot objects с reservation.** Пользоваться штатными `QUOTAS.reserve/commit/rollback` и Snapshot OVO; сохранять UUID и reservations до worker dispatch. Duplicate reserve не повторяет начисление; отказ одного volume оставляет остальные без нового активного lock и snapshot.
- [x] **Запустить GREEN:** команда RED + существующие API version tests, найденные через `rg --files cinder/tests | rg version`. Ожидается 0 failures, route 3.71 не выполняет новой логики.
- [x] **Commit:** `feat: expose authenticated checkpoint reservation API`.

### Task 5: RPC 3.21, native execution и неизвестный результат

**Files:** Create `cinder/checkpoints/manager.py`, `cinder/tests/unit/checkpoints/test_manager.py`; Modify `cinder/volume/{manager,rpcapi}.py`, `cinder/tests/unit/volume/test_rpcapi.py` и API dispatch из Task 4.

**Interfaces:** Consumes contracts/OVO/step claims. Produces RPC `get_checkpoint_capabilities(ctxt, volume_id, host) -> dict`, `execute_checkpoint_step(ctxt, step_id, generation, host) -> None`, `reconcile_checkpoint_step(ctxt, step_id, generation, host) -> None`; manager entrypoints делегируют `CheckpointManager` из §1.4. `host` определяется сервером из текущего volume.service_topic_queue, не клиентом.

- [x] **Написать RED execution/version tests.** `test_lost_reply_never_reexecutes`: после эффекта и исключения `self.assertEqual(1, fake.native_call_count)` и `self.assertEqual('recovery_required', op.status)`; `test_two_rpc_deliveries_have_one_claim`: один вызов; `test_rpc_320_rejected_before_reserve`: нет records/locks; `test_native_failure_never_calls_copy_fallback`: `_revert_to_snapshot_generic.assert_not_called()`.
- [x] **Запустить RED:** `tox -e py311 -- 'cinder.tests.unit.(checkpoints.test_manager|volume.test_rpcapi)'`.
- [x] **Добавить RPC history/version 3.21 и gates.** Использовать существующие `assert_min_rpc_version`, `can_send_version`, OVO cap и service lookup. Недоступный/неподтверждённый target не считается новым по умолчанию только потому, что `determine_rpc_version_cap` выбрал локальную версию. Старые calls сохраняют прежние versions/kwargs.
- [x] **Реализовать native execution:** verify owner/generation/resource/state → durable step claim → новый driver method с step UUID → durable result. Revert принимает только доступный том без attachments и snapshot этого volume/размера. Никакого вызова legacy manager revert. Внешний exception/result записывается как безопасный код, исходные секреты не логируются.
- [x] **Реализовать read-only reconciliation:** pending/unknown → driver query → подтверждённый status; неизвестный результат удерживает locks. Не повторять effect, не освобождать ресурсы автоматически, не считать snapshot DB `404` доказательством отсутствия backend-объекта. Cleanup имеет собственный idempotent delete-step.
- [x] **Запустить GREEN:** команда RED; отдельно кейсы asynchronous pending, неверный result, partial multi-volume success и stale callback. Ожидается 0 failures.
- [x] **Commit:** `feat: execute and reconcile native checkpoint steps over RPC`.

### Task 6: Переподключение томов в контексте операции

**Files:** Modify `cinder/context.py`, `cinder/api/v3/attachments.py`, `cinder/volume/{api,manager,rpcapi}.py`, guards; Create `cinder/tests/unit/checkpoints/test_attachment_scope.py`; Test `cinder/tests/unit/test_context.py`, существующие tests attachments API/manager.

**Interfaces:** Проверенные `context.checkpoint_operation_id: str|None` и `context.checkpoint_generation: int|None`, сохранённые в `to_dict/from_dict` для RPC. Заголовки `OpenStack-Checkpoint-Operation-ID` и `OpenStack-Checkpoint-Generation` принимаются attachment controller только на 3.72 после `require_service`; сами заголовки не дают bypass.

- [x] **Написать RED tests.** `test_tenant_cannot_forge_attachment_scope`: 403 и нет attachment; `test_other_volume_or_instance_rejected`: 409; `test_valid_revert_reconnect_preserves_volume_uuid`: прежний volume ID, новый attachment ID; `test_context_scope_roundtrip`: поля не теряются при RPC; stale generation/foreign service/no operation — отказ.
- [x] **Запустить RED:** `tox -e py311 -- 'cinder.tests.unit.(checkpoints.test_attachment_scope|test_context|attachments)'`.
- [x] **Реализовать scope validation на API и worker.** Разрешать disconnect только исходного attachment данной operation и reconnect только к записанному instance после подтверждённого revert всех томов. Чужой volume, новый instance, уже released operation или старое поколение не дают права. Legacy initialize/terminate APIs scope не получают и остаются закрыты для protected volumes.
- [x] **Сохранить upgrade boundary:** scoped attachment RPC требует 3.21; без scope отправляется прежний вызов. Admission attachment reservation и checkpoint locks сериализуются на одном volume; проверить оба порядка гонки.
- [x] **Запустить GREEN:** команда RED + `cinder.tests.unit.api.v3.test_attachments`; ожидается 0 failures. Cinder не вызывает os-brick на compute и не имитирует доказательство физического отключения SAN: это обязанность N2.
- [x] **Commit:** `feat: authorize checkpoint-scoped volume reconnections`.

### Task 7: Restart recovery, cleanup и сохранение защиты

**Files:** Modify checkpoint API/manager, `cinder/volume/manager.py`, DB transitions; Create `cinder/tests/unit/checkpoints/test_recovery.py`; Test `cinder/tests/unit/volume/test_init_host.py`.

**Interfaces:** `CheckpointManager.recover_pending(context) -> None` — выбор durable steps данного backend, только reconciliation. Periodic вызов с интервалом 60 секунд и вызов при init_host после готовности driver, до cleanup, способного удалить checkpoint artifacts. Старые worker cleanup пути не должны удалять protected Snapshot как orphan.

- [x] **Написать RED tests.** `test_restart_after_native_effect_only_queries`: destructive count не растёт; `test_partial_revert_cannot_release`: 409 и все locks на месте; `test_disable_retains_guards_and_recovery`: flag false сохраняет защиту; `test_cleanup_is_repeatable`: повтор не меняет квоту и не удаляет чужой snapshot; `test_expired_wait_does_not_unlock`: locks сохраняются.
- [x] **Запустить RED:** `tox -e py311 -- 'cinder.tests.unit.(checkpoints.test_recovery|volume.test_init_host)'`.
- [x] **Реализовать recovery/cleanup/release.** Create abort до effect безопасен; после effect сначала reconcile + cleanup. Delete завершён только после подтверждённого backend удаления всех связанных snapshots; quota и references освобождаются единожды. Revert commit не разблокирует ресурсы до подтверждённого переподключения к записанной ВМ. Новую operation поверх recovery_required не принимать.
- [x] **Проверить legacy cleanup и purge:** ordinary purge не удаляет active references, steps или tombstones дедупликации. Сырые driver errors заменить стабильными error codes; в response достаточно resource IDs и последней подтверждённой фазы. В C1 нет метода «снять всё принудительно».
- [x] **Запустить GREEN:** команда RED и `cinder.tests.unit.db.test_purge`; ожидается 0 failures. Общая автоматическая HA/передача операции другому executor вне C1; query/reconcile на том же backend допустим без повторного dispatch.
- [x] **Commit:** `feat: recover interrupted checkpoint operations without replaying effects`.

### Task 8: Сквозная проверка C1, документация и воспроизводимый патч

**Files:** Create `cinder/tests/functional/test_checkpoint_contract.py`, `cinder/tests/functional/api_sample_tests/test_checkpoints.py` и samples в `samples/checkpoints/`; Create `api-ref/source/v3/checkpoints.inc`, `doc/source/admin/checkpoint-contract.rst`, `releasenotes/notes/checkpoint-contract-7fa82e10c601.yaml`; Modify соответствующие index/include и `api-ref/source/v3/parameters.yaml`.

**Interfaces:** Consumes весь C1. Produces документированный сервисный API, проверяемую migration/patch цепочку и отчёт о локальных результатах. Клиент/CLI, Nova и native vendor adapters не добавляются.

- [x] **Написать RED functional scenarios через настоящий WSGI/DB/RPC boundary и test driver.** `test_two_checkpoints_revert_old_and_keep_new`: A/B имеют разные snapshot IDs, A native revert не меняет volume UUID и B; `test_delete_a_preserves_b_and_current_data`; `test_cross_backend_partial_failure_holds_all_locks`; `test_released_request_replay_creates_nothing`; `test_legacy_snapshot_api_unchanged`.
- [x] **Запустить RED:** `tox -e functional-py311 -- cinder.tests.functional.test_checkpoint_contract`. Fake storage моделирует данные и счётчики, но тест не считается настоящим RAM/SAN restore.
- [x] **Закрыть обнаруженные интеграционные дефекты минимальными изменениями и заполнить API samples/docs.** Документировать service-token/allowlist, protected resources, native-only behavior, feature disable, смешанные версии, коды recovery и ограничения C1. Не приводить команды выполнения на реальной СХД с реальными идентификаторами.
- [x] **Запустить целевые GREEN, затем обязательную регрессию:**

```bash
tox -e py311 -- 'cinder.tests.unit.(checkpoints|objects|db|api|policies|attachments|volume|group|transfer|backup)'
tox -e functional-py311 -- cinder.tests.functional.test_checkpoint_contract
tox -e api-samples -- checkpoints
tox -e compliance
tox -e pep8
tox -e api-ref
```

Каждая команда: exit 0 и 0 failures; skips и ограничения среды перечислить отдельно. MySQL/PostgreSQL migration/concurrency из Task 2 обязательны; отсутствие СХД допустимо для C1 и явно отмечается. Не повторять широкие suites после успеха без новых изменений/причины.

- [x] **Commit:** `test: cover and document the Cinder checkpoint contract` после зелёных проверок.
- [x] **Собрать результат вне исходного архива:** серия `git format-patch` от локального baseline, README с порядком/миграцией и limitations, SHA-256, фактические команды и результаты. Применить серию в новой чистой копии того же ZIP с `git apply --check` и проверить итоговое дерево против рабочей копии; не выдавать применимость за запуск на СХД. Push/deploy не входят в этот план.

## 4. Самопроверка плана и передача в исполнение

| Требование согласованной spec | Покрытие C1 |
|---|---|
| Capabilities и native-only | Task 1/4/5 |
| Durable operation, idempotency, all-volume ownership | Task 2/4/5/7 |
| Прямой обход Cinder, concurrency | Task 2/3/6 |
| Service/project authorization и секреты | Task 4/5/6 |
| Attachments при сохранении volume UUID | Task 6; физический compute disconnect/restore — будущий N2 |
| Restart, partial failures, cleanup | Task 5/7/8 |
| API/RPC/OVO compatibility | Task 2/4/5/6/8 |
| RAM, Nova quotas/API, libvirt, реальная СХД и время паузы VMware | За рамками C1: N1/N2, D1/D2, V1 |

Пять Review Focus закреплены в конкретных тестах. Имена интерфейсов в задачах соответствуют §1.4. Дополнительные решения C1 (3.72/3.21/1.40, routes, service allowlist, DB schema, phases и guard matrix) предъявляются пользователю вместе с планом до реализации.

**Критерий завершения C1:** код и документация общего контракта проверены локально, неподдержанный driver отклоняется до эффектов, fake multi-volume lifecycle и отказные сценарии проходят, патч воспроизводимо применяется к исходному архиву. Это не готовая функция RAM snapshot для конечного пользователя.

**Рекомендуемое исполнение:** Native — один исполнитель последовательно проходит задачи с общими DB/RPC интерфейсами, затем отдельный reviewer проверяет весь пакет. Альтернатива — subagent-driven с отдельной реализацией и review каждой задачи. Метод выбирает пользователь после просмотра плана.

**Текущий статус:** C1 завершён локально. Tasks 1–8 и единственный fix pass после независимого review выполнены. Финальный набор: 15 664 testsRun, 15 659 passed, 5 штатных skips, 0 failures/errors; SberCloud исключён по решению пользователя. Десять патчей проверяются на исходном ZIP. HEAD `d2d14a5251fa7f61771f83bfbf50f12215dd051d`. Подробности — в [статусе исполнения](2026-10-06-cinder-checkpoint-execution-status.md) и пакете (`deliverables/cinder-checkpoint-contract-c1/README.md`).
