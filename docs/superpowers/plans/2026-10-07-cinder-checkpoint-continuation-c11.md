# Cinder Checkpoint Continuation C1.1 Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking. Сохраняется выбранное ранее Native-исполнение; план согласован пользователем, Native-исполнение начато.

**Goal:** Продолжать уже принятую checkpoint operation после истечения пользовательского токена, сохраняя исходного владельца, и устранять дубли attachments при потере ответа.

**Architecture:** Первичный dual-identity reserve атомарно сохраняет ограниченное разрешение Nova service user. Отдельные service routes и scope-aware DB/worker entry points проверяют это разрешение без impersonation. Дочерние attachment intents фиксируют target и результат до/после I/O.

**Tech Stack:** Python 3.11/Linux, Cinder fork, SQLAlchemy/Alembic, oslo.versionedobjects/messaging, Keystone middleware, SQLite/MySQL/PostgreSQL, unittest/stestr.

**Spec:** [Согласованная спецификация N2](../specs/2026-10-07-nova-checkpoint-executor-n2-design.md), разделы 2, 3, 5, 6, 8–11. Пользователь согласовал её ответом «ок»; план и начало исполнения также согласованы пользователем.

## Global Constraints

- База: C1 `d2d14a5251fa7f61771f83bfbf50f12215dd051d`; новая ветка `feature/cinder-checkpoint-continuation`, отдельный checkout `work/cinder-checkpoint-continuation`.
- «Готовые серии C1/N1 не переписываются.» Архивы, `sources/`, готовые deliverables и исходные ветки сохраняются.
- «Cinder API **3.73**, volume RPC **3.22**; native driver contract **1** не меняется без доказанной необходимости.» Cinder object manifest — новый **1.41** поверх текущего 1.40.
- «Исходный actor неизменяем; новый аутентифицированный executor учитывается отдельно.» Не изготовлять context от имени исходного пользователя; не ослаблять legacy `_owned`.
- «Service continuation живёт до завершения операции. После release разрешено только чтение tombstone.» generation=1; новый reserve не разрешается service-only credentials.
- «Контроль scope повторяется в API, DB, RPC worker и attachment mutation paths.» Никаких SQL transactions через RPC/driver I/O.
- Пустой `nova_service_user_ids` запрещает доступ. Legacy 3.72, feature default-off и `max_volumes_per_operation=64` сохраняются.
- Закрытый `storage_client>=1.3` и SberCloud остаются только ранее разрешённым тестовым исключением; production requirements не менять.
- Разработка и проверки локальные. Реальные Keystone/SAN и действующие ВМ — отдельная V1-проверка; план не разрешает инфраструктурные изменения.

## Review Focus

1. Чужой service угадывает UUID или присылает `service_user_id` в headers: ни данных, ни I/O; Task 1/3.
2. Ответ reserve потерян между commit и REST response: lookup возвращает ту же operation; отсутствие записи не даёт service права reserve; Task 2/3.
3. Update attachment мог выполнить backend effect, но не записал ответ: повтор не вызывает initialize_connection ещё раз; Task 4/5.
4. Grant отозван или operation released между API и worker: поздний RPC не мутирует ресурс; Task 5.
5. Два actor одного проекта или изменённый connector повторяют один request key: ownership/body mismatch не превращается в успешный replay; Task 2/4.

---

## 1. Подготовка и команды

Все пути к коду далее относительны новому checkout. Перед Task 1 проверить изоляцию через `superpowers:using-git-worktrees`, исходный SHA и отсутствие чужих изменений. Не создавать ветку из текущего случайного HEAD. Записать base commit/tree в execution ledger.

Среда: отдельный одноразовый Linux container с checkout в `/work`, Python 3.11, исходными OpenStack 2025.1 constraints, SQLite, MySQL 8 и PostgreSQL 16. Сеть тестовая без published DB ports; `OS_TEST_DBAPI_ADMIN_CONNECTION` должен перечислять все три disposable URL. Повторять подготовку по C1 VALIDATION (`deliverables/cinder-checkpoint-contract-c1/VALIDATION.md`), используя публичный requirements artifact из C1; host Python не изменять. Создать `.venv-c11` и установить editable Cinder с `--no-deps` после публичных requirements. Сохранить freeze, версии СУБД и ограничения среды.

Команды в Tasks выполняются внутри container из `/work`:

```sh
export PBR_VERSION=0.0.0
export OS_STDOUT_CAPTURE=1 OS_STDERR_CAPTURE=1 OS_TEST_TIMEOUT=160
export PYTHONDONTWRITEBYTECODE=1 SQLALCHEMY_WARN_20=1
.venv-c11/bin/python -m unittest -v cinder.tests.unit.checkpoints.test_api
```

Последняя команда — baseline C1, expected `OK`. Ошибка зависимостей не считается RED продукта. Для широкого набора сохранить C1 подход с независимыми workers: не сокращать coverage из-за прежних монолитных зависаний. Обнаруживать новые test modules, а не пользоваться неизменным старым списком.

Для каждого Task сохранить RED/GREEN, точные команды, commit и отклонения в `.superpowers/sdd/2026-10-07-cinder-checkpoint-continuation-c11/progress.md`. Перед commit выполнять `git diff --check`; staging — только файлы задачи.

## 2. Файлы и публичная граница между C1.1 и N2

| Файлы | Ответственность |
|---|---|
| `cinder/checkpoints/continuation_contracts.py` | Строгие scope/envelope/request schemas и canonical hashes, без DB |
| `cinder/checkpoints/continuation.py` | Проверка service identity и получение scope из durable grant |
| `cinder/checkpoints/continuation_api.py` | Service API orchestration с явным scope |
| `cinder/checkpoints/attachment_intents.py` | Идемпотентные attachment actions и query |
| `cinder/db/sqlalchemy/checkpoint_continuation.py` | Grant/attachment-intent transactions и ограниченные resource lookups |
| `cinder/objects/checkpoint_continuation.py` | Новые OVO; явные методы, без generic save |
| `cinder/api/v3/checkpoint_continuation.py` | Только новые service routes; legacy controller отдельно |
| `cinder/checkpoints/{api,manager,guards}.py`, `cinder/db/sqlalchemy/checkpoints.py` | Совместимое добавление reserve grant, explicit scope в core transitions |
| `cinder/volume/{api,manager,rpcapi}.py`, attachment DB primitives | Общие низкоуровневые операции с повторной scope-проверкой; не обход legacy guards |

### 2.1. Wire contract C1.1, version 1

Все UUID — canonical lowercase UUID strings; generation только integer 1, bool не принимается. JSON schemas закрыты (`additionalProperties=false`). Hash — SHA-256 canonical JSON с сортировкой keys и volumes. Connector ограничен штатными валидируемыми полями Cinder, неизвестные top-level поля не принимаются.

Первичный `POST /v3/{owner_project}/checkpoint-operations`, microversion 3.73, требует действующую пользовательскую identity + разрешённую Nova service identity. Старый payload 3.72 остаётся допустим; для делегирования добавляются **оба** поля:

```json
{
  "checkpoint_operation": {
    "request_id": "00000000-0000-4000-8000-000000000001",
    "checkpoint_id": "00000000-0000-4000-8000-000000000002",
    "instance_uuid": "00000000-0000-4000-8000-000000000003",
    "kind": "revert",
    "volume_ids": ["00000000-0000-4000-8000-000000000004"],
    "nova_operation_id": "00000000-0000-4000-8000-000000000005",
    "allow_service_continuation": true
  }
}
```

Request ID N2 для Cinder reserve равен Nova operation UUID; Cinder operation UUID выдаётся Cinder. Delegation fields входят в dedup hash. Legacy operation нельзя задним числом делегировать exact replay с другим телом.

Новые пути относительны `/v3`; projectless routes предпочтительны. При наличии `/{project_id}` URL содержит project аутентифицированного **service**, не tenant-владельца ресурсов; несовпадение отвергается. N2 строит endpoint через service catalog/versioned endpoint, не подставляет tenant в service context.

| Метод и путь | Контракт |
|---|---|
| `GET /checkpoint-service-operations/{nova_operation_id}` | `200` существующая делегированная operation; нет коллекционного list |
| `POST /checkpoint-service-operations/{nova_operation_id}/action` | C1 action `execute/reconcile/cleanup/release`, generation=1, release outcome=`commit/abort`; `202` |
| `GET /checkpoint-service-operations/{nova_operation_id}/volumes/{volume_id}/capabilities` | `200`, тот же driver contract 1, только volumes этой operation |
| `POST /checkpoint-service-operations/{nova_operation_id}/attachment-requests` | `202`, зарегистрированный child intent; body ниже |
| `GET /checkpoint-service-operations/{nova_operation_id}/attachment-requests/{child_id}` | `200`, safe result этого intent; не повторяет effect |
| `GET /checkpoint-service-operations/{nova_operation_id}/attachments/{attachment_id}` | `200`, только допустимый по фазе attachment этой ВМ/volume; актуальный connection_info по существующим правилам Cinder |

Service operation response: `{"checkpoint_operation": {...}}`; все поля C1 detail плюс `continuation={version:1,nova_operation_id,owner_project_id,actor_user_id,executor_service_user_id,generation:1}`. Ни bearer tokens, ни backend references. Release tombstone immutable. Только transient scoped attachment read может вернуть connection_info: не включать его в operation/intent journal, DEBUG HTTP dump или общие API Nova.

Attachment body: `{"attachment_request":{"id":UUID,"generation":1,"volume_id":UUID,"action":ACTION,"attachment_id":UUID_OR_NULL,"connector":DICT_OR_NULL,"mountpoint":STR_OR_NULL}}`. Actions=`create|update|complete|delete`. Create: attachment_id=null, connector=null, mountpoint=null, instance берётся из grant; update: ID существующего нового attachment и connector обязательны; complete/delete: ID обязателен, connector/mountpoint=null. Это отдельный route, не произвольный bypass `/attachments`.

Ответ: `{"attachment_request":{"id":UUID,"operation_id":UUID,"generation":1,"volume_id":UUID,"action":ACTION,"status":STATUS,"attachment_id":UUID_OR_NULL,"error_code":CODE_OR_NULL}}`. STATUS=`prepared|dispatching|pending|succeeded|failed_no_effect|unknown`; safe codes используют закрытый набор C1 плюс `attachment_result_unknown`, `request_payload_changed`, `continuation_revoked`. Backend result query здесь не объявляется универсально поддержанным.

HTTP: 400 invalid schema/version; 401 недействительная authentication; 403 неразрешённый service/role; 404 отсутствующий или чужой grant/resource; 409 body/generation/phase conflict; 503 недоступный worker. `404` не означает absence backend effect.

### 2.2. Общие Python-типы

В `continuation_contracts.py` Task 1 определяет immutable dataclasses с `to_primitive()/from_primitive()` и `schema_version=1`:

- `ContinuationScope(grant_id: str, operation_id: str, nova_operation_id: str, owner_project_id: str, actor_user_id: str, executor_service_user_id: str, instance_uuid: str, checkpoint_id: str, kind: str, generation: int, volume_ids: tuple[str, ...], grant_revision: int)`.
- `AttachmentRequest(id: str, generation: int, volume_id: str, action: str, attachment_id: str|None, connector: dict|None, mountpoint: str|None)`.
- `AttachmentResult(id: str, operation_id: str, generation: int, volume_id: str, action: str, status: str, attachment_id: str|None, error_code: str|None)`.
- `validate_reserve_373(payload: dict) -> dict`, `validate_attachment(value: dict) -> AttachmentRequest`, `body_hash(value: dict) -> str`.

Scope из wire/context — только ссылка для повторной DB-проверки, не самостоятельно действующее разрешение. Caller identity остаётся сервисной. Никакого `is_admin=True` как замены grant.

## Task 1: Контракт и проверка service identity

**Files:** Create `cinder/checkpoints/continuation_contracts.py`, `cinder/checkpoints/continuation.py`, `cinder/tests/unit/checkpoints/test_continuation_contracts.py`; Modify `cinder/policies/checkpoints.py`, `cinder/exception.py`.

**Interfaces:** Consumes C1 canonical hashing и `CONF.checkpoint.nova_service_user_ids`. Produces типы §2.2; `continuation.require_executor(context) -> str` возвращает проверенный token subject; policies `checkpoint:continue`, `checkpoint:continuation_read` с default `role:service` плюс mandatory UUID allowlist.

- [x] **Step 1 — RED contract tests.** В unittest setup использовать `cinder.context.RequestContext`, `uuidsentinel`; поля service_only context: user_id=uuids.service, project_id=uuids.service_project, roles=['service'], без исходного actor.

```python
def test_only_authenticated_subject_can_continue(self):
    self.flags(nova_service_user_ids=[uuids.service], group='checkpoint')
    ctx = context.RequestContext(uuids.service, uuids.service_project,
                                 roles=['service'])
    self.assertEqual(uuids.service, continuation.require_executor(ctx))
    ctx.user_id = uuids.attacker
    ctx.service_user_id = uuids.service
    self.assertRaises(exception.CheckpointServiceForbidden,
                      continuation.require_executor, ctx)

def test_boolean_generation_is_not_one(self):
    body = dict(id=uuids.child, generation=True, volume_id=uuids.volume,
                action='create', attachment_id=None, connector=None,
                mountpoint=None)
    self.assertRaises(ValueError, contracts.validate_attachment, body)
```

Также exact test `test_role_without_allowlist_and_allowlist_without_role_denied`: оба случая дают 403; `test_extra_fields_and_action_specific_shape_rejected`: неверные connector/ID формы дают 400.

- [x] **Step 2 — Run RED.** `.venv-c11/bin/python -m unittest -v cinder.tests.unit.checkpoints.test_continuation_contracts`; ожидается отсутствие новых интерфейсов, не зависимостей.
- [x] **Step 3 — Implement.** Реализовать §2.2, mandatory identity check и safe errors. Не считать `service_user_id`, принятый из тела, authenticated subject. Reuse существующий oslo policy механизм с target, не добавлять admin override.
- [x] **Step 4 — GREEN и C1 policy regression.** Та же команда плюс `cinder.tests.unit.policies.test_checkpoints`; expected `OK`, новые tests без skip.
- [x] **Step 5 — Commit:** `feat: define scoped checkpoint continuation contract`.

## Task 2: Атомарный grant и replay первичного reserve

**Files:** Create `cinder/db/migrations/versions/83d4e2a671c0_add_checkpoint_continuation.py`, `cinder/db/sqlalchemy/checkpoint_continuation.py`, `cinder/objects/checkpoint_continuation.py`, `cinder/tests/unit/checkpoints/test_continuation_db.py`; Modify `cinder/db/{api.py,sqlalchemy/models.py,sqlalchemy/checkpoints.py}`, `cinder/objects/{checkpoint.py,base.py}`, `cinder/checkpoints/api.py`, `cinder/tests/unit/{db/test_migrations.py,objects/test_objects.py}`.

**Interfaces:** Consumes §2.2. Produces DB facade `checkpoint_continuation_get(context, nova_operation_id: str) -> dict`, `checkpoint_continuation_scope(context, nova_operation_id: str, generation: int, action: str, volume_id: str|None=None) -> dict`; new `API.reserve_with_continuation(context, payload: dict) -> CheckpointOperation`. Grant вставляется внутри существующей reserve writer transaction, не после неё.

Schema: `checkpoint_continuation_grants(id UUID PK, operation_id UUID UNIQUE FK, nova_operation_id UUID, owner_project_id str255, actor_user_id str255, executor_service_user_id str255, generation int, revision int, state str16, payload_hash str64, created_at, updated_at)`, UNIQUE `(executor_service_user_id,nova_operation_id)`. state=`active|released|revoked`; записанный grant не удаляется при release. Resource set/kind/instance/checkpoint читаются из immutable operation и resources; не дублировать изменяемый список в grant.

Сохранить порядок lock C1: instance → volumes по UUID → operation/grant. Lookup может быть read-only, но mutation повторяет locked validation. Уникальная гонка grant при двух requests откатывает проигравшие snapshot quota reservations. Новый operation не может использовать UUID прежнего grant. Grant revocation повышает revision; отключение credentials не освобождает locks.

- [x] **Step 1 — RED.** Setup `ContinuationDBTest` расширяет `test_api.APIHelpers`; `self.prepare_api()` создаёт два volumes и `self.payload`; `self.service_ctx` — context Task 1 с тем же SERVICE_ID, что у исходного dual identity.

```python
def test_reply_loss_keeps_one_owner_and_one_charge(self):
    payload = dict(self.payload, nova_operation_id=uuids.nova_operation,
                   allow_service_continuation=True)
    first = self.api.reserve_with_continuation(self.ctxt, payload)
    second = self.api.reserve_with_continuation(self.ctxt, payload)
    row = db.checkpoint_continuation_get(self.service_ctx, uuids.nova_operation)
    self.assertEqual(first.id, second.id)
    self.assertEqual(first.id, row['operation_id'])
    self.assertEqual(self.ctxt.user_id, row['actor_user_id'])
    self.assertEqual(2, self.usage()['snapshots']['in_use'])
```

Добавить `test_grant_insert_failure_rolls_back_reserve`, `test_changed_actor_or_payload_conflicts`, `test_released_grant_is_read_only`, `test_two_connections_same_nova_uuid_one_winner`: assert одна operation/grant, без проигравших locks/quota. Последний тест выполнять отдельными SQL sessions на MySQL/PostgreSQL, не только mock lock.

- [x] **Step 2 — Run RED.** `.venv-c11/bin/python -m unittest -v cinder.tests.unit.checkpoints.test_continuation_db`.
- [x] **Step 3 — Implement.** Additive migration после `7fa82e10c601`; явно проектные lookups под scope без изменения caller project/user. Новые OVO `CheckpointContinuationGrant`/List 1.0; `CheckpointOperation` 1.1 добавляет nullable `continuation_id`, down-conversion разрешена только для C1 operation без grant. Manifest 1.41. Не модифицировать migration C1.
  `continuation_id` — nullable String(36) в operation без обратного FK, grant.operation_id имеет FK/UNIQUE: это исключает циклическую вставку. Operation, grant и обратная ссылка заполняются внутри одной reserve transaction.
- [x] **Step 4 — GREEN.** Та же команда плюс `cinder.tests.unit.db.test_migrations`, `cinder.tests.unit.objects.test_objects`, `cinder.tests.unit.checkpoints.test_db`, `cinder.tests.unit.checkpoints.test_api`; expected `OK` и реальный проход трёх DB backends.
- [x] **Step 5 — Commit:** `feat: persist checkpoint continuation grants atomically`.

## Task 3: REST 3.73 и ограниченные service actions

**Files:** Create `cinder/api/{v3/checkpoint_continuation.py,schemas/checkpoint_continuation.py}`, `cinder/checkpoints/continuation_api.py`, `cinder/tests/unit/api/v3/test_checkpoint_continuation.py`; Modify `cinder/api/{v3/router.py,v3/checkpoints.py,v3/views/checkpoints.py,microversions.py,openstack/api_version_request.py}`, `cinder/db/sqlalchemy/checkpoint_continuation.py`.

**Interfaces:** `ContinuationAPI.get(context, nova_operation_id: str) -> dict`; `.action(context, nova_operation_id: str, name: str, generation: int, outcome: str|None=None) -> dict`; `.capabilities(context, nova_operation_id: str, volume_id: str) -> dict`; `continuation.authorize(context, nova_operation_id: str, generation: int, action: str, volume_id: str|None=None) -> ContinuationScope`. Exact routes/response — §2.1. Task 5 реализует вызываемые worker RPC; в этом Task mock RPC только для REST tests.

- [x] **Step 1 — RED.** Test setup использует C1 WSGI router/real policy и подготовленные Task 2 grants. Test helper `request_as(ctx, method, path, body=None, version='3.73') -> webob.Response` ставит доверенный fixture context; это только API-level тест, не доказательство middleware.

```python
def test_foreign_service_cannot_read_grant(self):
    response = self.request_as(self.foreign_service, 'GET', self.service_path)
    self.assertEqual(404, response.status_int)
    self.rpc.execute_checkpoint_step.assert_not_called()

def test_service_cannot_reserve_new_operation(self):
    response = self.request_as(self.service_ctx, 'POST',
                              '/checkpoint-operations', self.reserve_body)
    self.assertEqual(403, response.status_int)
    self.assertEqual(1, self.count_operations())
```

Setup определяет `service_path='/checkpoint-service-operations/'+nova_operation_id`, `reserve_body` §2.1; `count_operations()` считает через DB fixture. Дополнить tests: старый 3.72 не видит новые routes/fields; неизвестный UUID/чужой volume дают 404; changed generation даёт 409; mutable action после release даёт 409, GET возвращает tombstone; service-project URL не выбирает owner project. В 3.73 reserve body без delegation сохраняет legacy C1 путь.

- [x] **Step 2 — Run RED.** `.venv-c11/bin/python -m unittest -v cinder.tests.unit.api.v3.test_checkpoint_continuation`.
- [x] **Step 3 — Implement.** Зарегистрировать `CHECKPOINT_CONTINUATION='3.73'`, max version/history, закрытые schemas и views. Scope loads используют owner из grant. Отсутствующий grant не создаёт operation и не снимает locks. Errors не содержат connector, raw exceptions, токены.
- [x] **Step 4 — GREEN.** Новые tests + `cinder.tests.unit.api.v3.test_checkpoints`, `cinder.tests.unit.policies.test_checkpoints`; expected `OK`.
- [x] **Step 5 — Commit:** `feat: expose restricted checkpoint continuation API`.

## Task 4: Durable attachment requests

**Files:** Create `cinder/checkpoints/attachment_intents.py`, `cinder/tests/unit/checkpoints/test_attachment_intents.py`, migration `cinder/db/migrations/versions/95b6d7f082e1_add_checkpoint_attachment_intents.py`; Modify continuation API/DB/schema/OVO, `cinder/db/sqlalchemy/{models,api}.py`, `cinder/volume/api.py`.

**Interfaces:** `AttachmentIntents.submit(context, scope: ContinuationScope, request: AttachmentRequest) -> AttachmentResult`; `.get(context, scope, child_id: str) -> AttachmentResult`; `.get_attachment(context, scope, attachment_id: str) -> dict`. DB facade: `checkpoint_attachment_intent_prepare(context, scope: dict, request: dict) -> dict`, `checkpoint_attachment_intent_claim(context, child_id: str, executor_id: str) -> dict|None`, `checkpoint_attachment_intent_finish(context, child_id: str, result: dict) -> dict`. Driver connection_info только через scoped current read, не в result.

Table `checkpoint_attachment_intents`: id UUID PK, operation_id FK, generation, volume_id, action, request_hash, target_attachment_id nullable, result_attachment_id nullable, connector_json nullable, mountpoint nullable, executor_id nullable, status, safe error_code, timestamps. UNIQUE `(operation_id,generation,volume_id,action)` в первой версии: один attach cycle на revert; альтернативный child UUID для уже принятого action — conflict. Connector не содержит credentials, allowed fields валидируются; сохранять только то, что необходимо для исполнения этой операции.

Добавить `CheckpointAttachmentIntent`/List OVO 1.0 в ещё не выпущенный manifest 1.41. `AttachmentIntents.submit` подготавливает update/delete для worker Task 5; Task 4 тестирует DB identity/dedup, не притворяется готовым backend executor. Connector schema разрешает используемые fork поля os-brick: host, ip, initiator, wwpns/wwnns, multipath, platform, os_type, nqn, uuid, system uuid и штатные необязательные capabilities; точные spelling/types брать из текущего Cinder connector validation и фиксировать regression fixture, management passwords/CHAP data туда не добавлять.

Create резервирует volume и создаёт единственный attachment, result ID и success в одной DB transaction с intent. Backend I/O отсутствует до update. Delete/update требуют separate durable dispatch и отдельного worker; unknown update/delete не повторяется. C1 ограничения disconnect/reconnect сохраняются. Release не проходит при nonterminal attachment intents или незавершённом complete.

- [x] **Step 1 — RED.** Setup строит C1 create+release и revert reservation по `test_attachment_scope` helpers; stage переводится только подтверждёнными Cinder steps. `self.create_request` — §2.2, `self.scope` — Task 3 authorize.

```python
def test_create_reply_loss_reuses_attachment(self):
    first = self.intents.submit(self.service_ctx, self.scope, self.create_request)
    replay = self.intents.submit(self.service_ctx, self.scope, self.create_request)
    self.assertEqual('succeeded', first.status)
    self.assertEqual(first.attachment_id, replay.attachment_id)
    self.assertEqual(1, self.count_volume_attachments(first.volume_id))

def test_changed_connector_does_not_create_second_intent(self):
    accepted = self.intents.submit(self.service_ctx, self.scope, self.update_request)
    changed = dataclasses.replace(self.update_request,
                                  connector={'host': 'other-compute'})
    self.assertRaises(exception.CheckpointConflict, self.intents.submit,
                      self.service_ctx, self.scope, changed)
    observed = self.intents.get(self.service_ctx, self.scope, accepted.id)
    self.assertEqual(accepted.id, observed.id)
    self.assertEqual(1, self.count_attachment_intents('update'))
```

Дополнить assertions: новый UUID того же action не создаёт attachment; disconnect принимает только original IDs; reconnect только после всех disk successes; create concurrency на двух DB sessions даёт один ID; complete before valid connection отвергается; query never calls driver; unknown delete держит lock.

- [x] **Step 2 — Run RED.** `.venv-c11/bin/python -m unittest -v cinder.tests.unit.checkpoints.test_attachment_intents`.
- [x] **Step 3 — Implement.** Reuse штатные mutation primitives после проверенного scope; вынести минимальные DB primitives, если существующий API смешивает policy/I/O. Не строить context с owner identity. Scope-aware resource getters в новом DB модуле жёстко фильтруют operation→volume→attachment и не расширяют generic tenant queries.
- [x] **Step 4 — GREEN.** Новые tests + `cinder.tests.unit.checkpoints.test_attachment_scope`, migration/model sync и OVO tests, MySQL/PostgreSQL create race; expected `OK`.
- [x] **Step 5 — Commit:** `feat: journal checkpoint attachment requests`.

## Task 5: RPC 3.22, worker проверки и неопределённые эффекты

**Files:** Create `cinder/checkpoints/continuation_manager.py`, `cinder/tests/unit/checkpoints/test_continuation_manager.py`; Modify `cinder/volume/{rpcapi,manager}.py`, `cinder/checkpoints/{manager,guards,continuation_api,attachment_intents}.py`, `cinder/context.py`, `cinder/db/sqlalchemy/checkpoint_continuation.py`, `cinder/tests/unit/test_rpc.py`.

**Interfaces:** Новые worker methods:
`get_checkpoint_continuation_capabilities(context, nova_operation_id: str, volume_id: str) -> dict`,
`execute_checkpoint_continuation(context, nova_operation_id: str, generation: int) -> None`,
`reconcile_checkpoint_continuation(context, nova_operation_id: str, generation: int) -> None`,
`execute_checkpoint_attachment(context, nova_operation_id: str, child_id: str, generation: int) -> None`,
`query_checkpoint_attachment(context, nova_operation_id: str, child_id: str, generation: int) -> dict`.
RPC `VolumeAPI` зеркалит сигнатуры с завершающим `host: str`; casts для execute, calls для query/capabilities. Capability route Task 3 вызывает новый scoped RPC, а не legacy C1 handler с неподходящей service-only identity. Worker re-authorize по durable grant, identity и текущей revision. RequestContext сохраняет только ссылку `checkpoint_continuation_id`; wire scope не даёт полномочий без проверки.

Core C1 DB transitions выделяют internal functions с явно переданным owner tuple; legacy wrappers получают tuple через прежний `_owned`, continuation wrappers — через проверенный grant. Не менять поведение старых API. Resource OVO не должны незаметно вызывать tenant-scoped save с service project: scoped writes проходят только explicit DB transitions. Quota deltas используют сохранённый owner_project_id.

Worker перепроверяет grant state/revision и локальный allowlist; `revoked` либо удалённый из allowlist subject запрещён. Проверка отзыва Keystone credentials проходит через штатную token validation на новых REST-запросах; не обещать мгновенной отмены уже принятого RPC или backend I/O при выключении пользователя в Keystone. Для исключения позднего эффекта нужен подтверждённый fence, а не только факт истечения/отзыва токена.

- [x] **Step 1 — RED.** Setup предоставляет два worker hosts и mock driver, настоящую сериализацию нового RPC context и grant Task 2.

```python
def test_revocation_between_api_and_worker_prevents_effect(self):
    primitive = self.prepare_attachment_dispatch()
    self.revoke_grant()
    self.assertRaises(exception.CheckpointServiceForbidden,
                      self.worker.execute_checkpoint_attachment,
                      self.service_ctx, self.nova_operation_id,
                      primitive['id'], 1)
    self.driver.initialize_connection.assert_not_called()

def test_unknown_update_is_never_redispatched(self):
    self.driver.initialize_connection.side_effect = TimeoutError()
    self.execute_update()
    self.execute_update()
    result = self.query_update()
    self.assertEqual('unknown', result['status'])
    self.assertEqual(1, self.driver.initialize_connection.call_count)
    self.assertTrue(self.volume_lock_exists())
```

Task-local fixture helper methods точно оборачивают новые worker signatures и `AttachmentIntents.get`; revoke меняет grant state/revision через locked DB fixture. Проверить worker host mismatch, 3.21 cap, object cap 1.40, forged scope, lost release reply, dispatch после release и identity revoke. C1 internal reconciliation остаётся read-only; continuation entry не восстанавливает actor roles из DB.

- [x] **Step 2 — Run RED.** `.venv-c11/bin/python -m unittest -v cinder.tests.unit.checkpoints.test_continuation_manager`.
- [x] **Step 3 — Implement.** RPC cap 3.22 + object cap 1.41 до reserve/dispatch. Backend exceptions после claim → unknown. У стандартного initialize/terminate connection нет общего query-by-request-id: без точного доказательства сохранять unknown, не объявлять функцию автоматически идемпотентной. Read-only queries могут подтвердить DB-only create/complete; неизвестный backend update/delete требует D/V1 доказательства либо оператора.
- [x] **Step 4 — GREEN.** Новые tests + C1 `test_manager`, `test_recovery`, `test_attachment_scope`, RPC/service tests; expected `OK`. Сверить обычные attachment RPC версии и legacy calls.
- [x] **Step 5 — Commit:** `feat: enforce checkpoint continuation at worker boundaries`.

## Task 6: Сквозные проверки, документация и поставка

**Files:** Create `cinder/tests/functional/test_checkpoint_continuation.py`, `cinder/tests/functional/checkpoint_continuation_fixture.py`, `cinder/tests/unit/api/middleware/test_checkpoint_continuation_auth.py`, `cinder/tests/functional/api_sample_tests/test_checkpoint_continuation.py`, `doc/source/admin/checkpoint-continuation.rst`, `api-ref/source/checkpoint-continuation.inc`; Modify соответствующие doc/API indexes и API version history.

**Interfaces:** `ContinuationFunctionalBase` расширяет `CheckpointFunctionalBase` C1; helper `request_with_token(token: str, method: str, path: str, body: dict|None=None) -> webob.Response` проходит `keystonemiddleware.auth_token.AuthProtocol` и `CinderKeystoneContext`. Token-validation endpoint в local tests заменён контролируемым responder; тест не инжектирует готовый trusted cinder context после middleware. Приватные связи и UUID совпадают с §2.1.

Test-only `checkpoint_continuation_fixture.serve_fixture(bind_host: str, port: int, ready_path: str) -> None` поднимает тот же WSGI/SQL/fake-driver fixture для Nova contract tests. CLI `python -m cinder.tests.functional.checkpoint_continuation_fixture --host 127.0.0.1 --port 0 --ready PATH`: записывает выбранный endpoint, fixture UUIDs и фиксированные тестовые token markers; завершение по SIGTERM с cleanup. Не включать launcher в production service entrypoints. Он работает из C1.1 venv, Nova обращается по HTTP из своего venv.

- [x] **Step 1 — RED functional tests.** Определить `test_expired_user_restart_service_completes_same_operation`, `test_service_only_missing_grant_cannot_reserve`, `test_forged_identity_headers_overwritten_by_middleware`, `test_attachment_response_loss_full_revert`, `test_create_delete_two_backends_keep_other_checkpoint`.

```python
def test_expired_user_restart_service_completes_same_operation(self):
    accepted = self.reserve_with_dual_tokens()
    self.token_server.expire_user_token()
    self.restart_api_and_workers()
    response = self.request_with_token(
        self.valid_service_token, 'GET',
        '/checkpoint-service-operations/' + self.nova_operation_id)
    op = response.json['checkpoint_operation']
    self.assertEqual(200, response.status_int)
    self.assertEqual(accepted['id'], op['id'])
    self.assertEqual(self.actor_id, op['continuation']['actor_user_id'])
    self.assertNotIn(self.user_token, self.serialized_journals())
```

Fixture methods: reserve создаёт §2.1 payload и записывает IDs/token fixtures; restart пересоздаёт API/worker objects при сохранении SQL; serialized_journals сериализует grant/operation/step/attachment-intent rows. Продолжить execute/reconcile/release того же op и assert одна backend dispatch на step и immutable actor.

- [x] **Step 2 — Run RED.** `.venv-c11/bin/python -m unittest -v cinder.tests.functional.test_checkpoint_continuation cinder.tests.unit.api.middleware.test_checkpoint_continuation_auth`.
- [x] **Step 3 — Implement тестовые fixtures и docs.** Описать scoped credentials, immutable actor, невозможность service-only initial reserve, unknown initialize_connection, новую микроверсию, правила release/revoke и отсутствие public grant-management API. Глобальный отзыв service identity выполняется штатной инфраструктурой, не новым произвольным force endpoint.
- [x] **Step 4 — GREEN/regression.** Новые tests, вся checkpoint/API/attachment suite, API samples, compliance, migrations/OVO, полный Cinder unit набор с единственным разрешённым SberCloud исключением; docs/API-ref `sphinx-build -W`, genconfig/genpolicy, executable checks. Запуск широкого набора — C1 worker harness с обнаружением новых modules. Сравнить skips и flake8 с точным C1 baseline; new failures/diagnostics=0.

  Подготовить scratch `run-regression.py` из C1 evidence, изменить `RUN` на текущий scratch, а discovery на `sorted(Path('cinder/tests/unit').rglob('test_*.py'))`: старый C1 `PARTS` покрывал широкий, но не полный набор. Сохранить все остальные worker/error/inventory проверки, единственное исключение `/sbercloud/`. Запустить тот же полный inventory и на точной базе C1; исходные failures сообщать отдельно, не сокращать inventory.

```sh
.venv-c11/bin/python .superpowers/sdd/2026-10-07-cinder-checkpoint-continuation-c11/run-regression.py
.venv-c11/bin/python -m unittest discover -s cinder/tests/compliance -v
.venv-c11/bin/python -m unittest -v cinder.tests.functional.test_checkpoint_contract cinder.tests.functional.test_checkpoint_continuation cinder.tests.functional.api_sample_tests.test_checkpoint_continuation
.venv-c11/bin/sphinx-build -W -b html doc/source doc/build/html
.venv-c11/bin/sphinx-build -W -b html api-ref/source api-ref/build/html
.venv-c11/bin/oslo-config-generator --config-file=tools/config/cinder-config-generator.conf
.venv-c11/bin/oslopolicy-sample-generator --config-file=tools/config/cinder-policy-generator.conf
.venv-c11/bin/python tools/check_exec.py cinder doc/source releasenotes/notes
git diff --check
```

  Expected suites exit 0, либо явно воспроизведённые только baseline failures с неизменным inventory; новых regressions=0. Генераторы и docs — exit 0. Документационные зависимости установить из исходного `doc/requirements.txt` под теми же constraints в disposable среде.
- [x] **Step 5 — Commit:** `test: verify checkpoint continuation lifecycle and authentication`.
- [x] **Step 6 — Native final review.** Один свежий reviewer всей серии base..HEAD; один цикл исправлений с regression по изменённым областям. Не выдавать авторскую проверку исправлений за второе независимое review. Сохранить findings/resolutions, deferred limitations и test evidence.
- [x] **Step 7 — Package.** `deliverables/cinder-checkpoint-continuation-c11/`: patches/series, BASE/HEAD commit/tree, source SHA, README, DESIGN/PLAN, VALIDATION, REVIEW/RESOLUTION, RULINGS, evidence и SHA256SUMS. Проверить C1→C1.1 series на чистом источнике, совпадение конечного tree. Обновить dependency pin N2 на фактический C1.1 HEAD только после этой проверки. Не push/merge/deploy.

## 3. Граница приёмки и покрытие спецификации

| Spec | Tasks |
|---|---|
| §5.1 identity/token/actor | 1, 2, 3, 5, 6 |
| §5.2 reserve/grant/replay/revoke | 2, 3, 5 |
| §5.2 scoped attachments/потеря ответа | 4, 5, 6 |
| §5.3 C1/N2 wire semantics | 1, 3, 4; N2 Task 5 |
| §8 unknown/late RPC/partial results | 4, 5, 6 |
| §9 compatibility/default-off/additive migrations | 2, 3, 5, 6 |
| §10 тесты и evidence | 6 |

Локальное завершение пакета доказывает авторизацию и state machine в приведённой тестовой среде. Реальные Keystone/RabbitMQ, D-profile и production recovery после неопределённого initialize/terminate connection — отдельная V1-приёмка. Пока они не пройдены, поддержка соответствующего профиля не включается. Отсутствие этих данных не заменяется fake success.

План сохранён вне Git, как остальные workspace plans. План согласован. Выполняются Native Task 1–6, после них N2; текущее состояние — в execution ledger.
