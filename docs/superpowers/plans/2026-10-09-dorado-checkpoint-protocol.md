# Dorado Checkpoint Protocol D1-P Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Создать отдельный пакет построения запросов и разбора ответов Dorado, проверяемый на локальном HTTP-сервере без массива и без ложного подтверждения checkpoint.

**Architecture:** Чистые builders создают проверяемый RequestSpec, transport отправляет его одной попыткой, parser возвращает наблюдения об ответе и snapshot. Независимый scripted server фиксирует фактические HTTP-запросы; сетевые сценарии запускаются в дочерних процессах с дедлайном. Терминальные результаты Cinder из этих наблюдений не выводятся.

**Tech Stack:** Python 3.11, стандартная библиотека dataclasses/enum/json/hashlib/http.client/socket/threading/subprocess/unittest; disposable Linux для сетевых проверок; без сторонних runtime dependencies.

**Spec:** [Согласованная спецификация D1-P](../specs/2026-10-09-dorado-checkpoint-protocol-design.md).

Статус: **согласован пользователем; выбран Native; исполнение начато 2026-10-09**.

## Global Constraints

- Самостоятельный repo `work/dorado-checkpoint-protocol/`, ветка `feature/dorado-checkpoint-protocol`; корень workspace вне Git, его не переинициализировать.
- «Python 3.11, без сторонних runtime dependencies»; `requires-python = ">=3.11,<3.12"`.
- Source reference: Huawei `v26.2.0`, commit `bda6d13cd0147648e04da5479d678dc2e0541395`, `Cinder/Epoxy`; это не профиль firmware.
- Cinder `d065a55cbe5cb8cd59489752846a41145d72bd27`, Nova `6d55009fae5eb55adb156e13835dd66240488f84`, L1 `02261dd0d4a28f525535a95cc1e7f68a1d34e906` остаются неизменными.
- «D1-P не импортирует Nova, Cinder, Huawei SDK или драйвер поставщика»; нет Cinder hooks, собственного durable journal или claims.
- Всегда `supported=false`, `production_qualified=false`, `hardware_qualified=false`, report scope `synthetic_vendor_protocol`.
- «Отсутствие автоматического retry относится к одному вызову транспорта»; межвызовная дедупликация не обещается.
- «Read-only методы выполняют только GET»; нет login/relogin, cleanup, отмены rollback, failover и автоматического продолжения substeps.
- ID/name: ASCII `[A-Za-z0-9_-]{1,128}`; description: непустой UTF-8 текст до 1024 bytes без управляющих символов; эти ограничения лабораторные.
- `ROLLBACKSPEED`: точный int 1–4; `operation_id`: UUID, только локальный, в vendor body/headers не передаётся.
- Только HTTP `127.0.0.1` и явный порт; endpoint создаёт runner. Proxy/redirect/URL credentials запрещены; реальная TLS/auth интеграция за границей этапа.
- «Размер запроса и распакованного тела ответа ограничен 1 MiB»; socket timeout 3 секунды, отдельный сетевой case process — 15 секунд.
- Fixtures всегда `origin=synthetic`, с source commit/path/lines и явными assumptions; никаких hardware captures или выдуманных API-гарантий.
- Нет merge/push/deploy. Полноценные D1, C1.2 при необходимости и V1 остаются отдельными этапами.

## Review Focus

1. Caller вручную изменил frozen RequestSpec через новый объект: transport повторно проверяет action/method/path/body и не открывает socket; Task 1/3 `test_forged_request_is_rejected_before_connect`.
2. Сервер выдаёт тело медленно, не достигая socket timeout: родитель прекращает свой case process через 15 секунд и фиксирует failure, а не успешный skip; Task 3 `test_trickle_is_failed_by_case_deadline`.
3. Мутирующий запрос получил 307 или 401: второму адресу и login route ничего не отправлено; Task 3 `test_redirect_and_auth_error_never_resubmit`.
4. Валидный ответ показывает старый rollback progress=100 при другом local UUID: он остаётся наблюдением и не создаёт Cinder status; Task 2 `test_old_rollback_is_only_an_observation`.
5. Summary помечен complete при пропущенном тесте, subtest failure или неполном inventory: verifier отвергает отчёт даже после пересчёта hashes; Task 4 `test_incomplete_or_failed_inventory_cannot_pass`.

---

## Рабочая область и доказательства

До исполнения применить `using-git-worktrees` для проверки изоляции. Здесь
нужен новый standalone repo, а не worktree внутри Nova/Cinder. Если целевой путь
появился, сначала прочитать его состояние; ничего не очищать и не перезаписывать.
Инициализация разрешается только после review этого плана и выбора исполнения.

Команды ниже запускаются из корня будущего repo. Сначала read-only проверить
Python 3.11, Linux, loopback bind и создание дочернего процесса. Если Linux
пока нет, Tasks 1/2 допускают portable-прогон; сетевые gates остаются pending.
Не менять инфраструктуру и не объявлять старую Docker-среду готовой без проверки.

На задачу сохранять `.superpowers/d1p/task-N/{red.log,green.log,commands.json}`
и commit SHA; scratch исключить из Git. Ошибка установки или неверный interpreter
не считаются содержательным RED. Copy approved spec/plan в `docs/design.md` и
`docs/plan.md` с исходными hashes; локальные ссылки переписать на копии входов,
сохранив originals в workspace.

## Карта файлов и общие интерфейсы

Все пути ниже относительно нового repo.

| Файлы | Назначение |
|---|---|
| `pyproject.toml`, `.gitignore`, `dorado_protocol/__init__.py` | Python package, metadata и ограничения qualification |
| `dorado_protocol/types.py`, `requests.py` | Неизменяемые типы, канонический JSON, семь builders, повторная валидация |
| `dorado_protocol/responses.py` | Строгий JSON и наблюдения без Cinder terminal states |
| `dorado_protocol/transport.py` | Loopback endpoint, framing, один HTTP request |
| `tests/test_requests.py`, `test_responses.py`, `test_transport.py` | Независимые ожидания и отрицательные проверки |
| `tests/http_server.py`, `network_cases.py`, `process_runner.py` | Raw HTTP scripted server и изолированные case processes |
| `tests/fixtures/index.json`, `tests/fixtures/responses/` | Синтетические raw bodies и их provenance |
| `tools/run_checks.py`, `report.py`, `package.py`, `tests/test_report.py` | Inventory, результаты, hashes, verifier и локальная поставка |
| `README.md`, `VALIDATION.md`, `REVIEW.md`, `REVIEW-RESOLUTION.md`, `docs/` | Запуск, реальные результаты, review и согласованные входы |

`tests/` и `tools/` — Python packages с `__init__.py`. CLI относится к комплекту
исходников: `python3.11 -m tools.run_checks`. Библиотека `dorado_protocol` не
импортирует tests/tools. Python-модули поставщика не копируются как implementation.

Task 1 определяет следующие frozen dataclasses и общий InputError:

```text
Identity(snapshot_id: str | None, name: str, parent_id: str, wwn: str | None = None)
RequestSpec(operation_id: str, action: str, method: str, relative_path: str,
            body: bytes | None, expected: Identity)
LabEndpoint(url: str, prefix: str)
WireReply(http_status: int | None, headers: tuple[tuple[str, str], ...],
          body: bytes, network_attempted: bool, diagnostic_code: str | None)
SubmissionObservation(kind: str, diagnostic_code: str | None,
                      network_attempted: bool, http_status: int | None,
                      body_sha256: str, body_size: int)
SnapshotObservation(kind: str, identity: Identity | None,
                    health: str | None, running: str | None,
                    rollback_rate: int | None, rollback_endtime: str | None,
                    missing_fields: tuple[str, ...])
ProtocolResult(submission: SubmissionObservation,
               snapshot: SnapshotObservation | None)
InputError(ValueError)
```

Submission kinds строго из spec §7. Snapshot kinds: `observed`,
`not_observed`, `ambiguous_identity`, `incomplete_listing`, `unrecognized_state`.
В этих типах нет `status`, `succeeded`, `failed_no_effect` или `pending`.
Текст exception не используется как diagnostic; коды ограничены enum в types.py.

Обязательные diagnostic codes: `invalid_request`, `invalid_endpoint`,
`invalid_token`, `transport_failure`, `invalid_framing`, `body_too_large`,
`unsupported_encoding`, `invalid_json`, `invalid_envelope`, `invalid_data`,
`identity_mismatch`, `invalid_snapshot_fields`, `http_error`, `vendor_error`.
Они не несут утверждения об отсутствии backend effect.

### Task 1: Типы и проверяемые builders

**Files:** Create `pyproject.toml`, `.gitignore`, `dorado_protocol/{__init__,types,requests}.py`, `tests/{__init__,test_requests}.py`, `docs/` input copies.

**Interfaces:**
- Consumes: approved spec §5 и raw-source evidence, без runtime imports.
- Produces: типы выше; `canonical_json(value: dict) -> bytes`; `validate_request(request: RequestSpec) -> None` (raises InputError).
- Produces: `create(operation_id: str, expected: Identity, description: str) -> RequestSpec`; `rollback(operation_id: str, expected: Identity, speed: int) -> RequestSpec`; `activate`, `stop`, `delete`, `get_by_id`, `get_by_name` — каждый `(operation_id: str, expected: Identity) -> RequestSpec`.

- [ ] **Step 1: Подготовить будущий repo и входы.** Создать только новый согласованный путь/ветку, сохранить копии spec/plan/provenance и ignore scratch/results. В build-system оставить setuptools; runtime dependencies пусты, тесты запускаются из source без обязательной установки пакета.
- [ ] **Step 2: Добавить независимые RED-тесты builders.** Зафиксировать literal method/path/body для всех семи операций, не получать ожидаемые значения через production builder. Например:

```python
def test_rollback_wire_shape(self):
    req = rollback(OP_ID, IDENTITY, 3)
    self.assertEqual(('PUT', '/snapshot/rollback'),
                     (req.method, req.relative_path))
    self.assertEqual(b'{"ID":"snap-1","ROLLBACKSPEED":3}', req.body)
    self.assertNotIn(OP_ID.encode(), req.body)

def test_bool_speed_is_not_integer_speed(self):
    with self.assertRaises(InputError):
        rollback(OP_ID, IDENTITY, True)
```

Здесь tests определяют `OP_ID='00000000-0000-4000-8000-000000000001'` и
`IDENTITY=Identity('snap-1','point-A','lun-1','wwn-1')`. Для create ID=None;
для get_by_name ID допускается None; остальные операции требуют native ID.
GET/DELETE body=None; create содержит три поля spec, activate — один элемент.

`test_identifier_boundaries` принимает длины 1/128 и отвергает 0/129,
Unicode, slash, percent escapes, query delimiters, CR/LF. `test_description_utf8_limit`
считает bytes и отвергает C0/C1 controls (U+0000–001F, U+007F–009F), пустую строку
и >1024 bytes; lone surrogate также даёт InputError. `test_non_uuid_operation_rejected` и `test_action_payload_binding`
проверяют UUID, неверные action/method/path, лишние/недостающие body keys,
bool speed, неверную expected identity и неканонический body.

- [ ] **Step 3: Запустить RED.** `python3.11 -m unittest tests.test_requests -v`; ожидается ошибка отсутствующего нового API, а не environment failure.
- [ ] **Step 4: Реализовать builders и validate_request.** Канонизация: UTF-8, sorted keys, separators `(',', ':')`, ensure_ascii=False, allow_nan=False. Validator повторно строит ожидаемый запрос из разрешённых полей, сверяет bytes и identity; не доверяет frozen dataclass как признаку валидности. Description и speed валидируются до сериализации.
- [ ] **Step 5: Проверить GREEN.** Та же команда: все случаи пройдены; отдельно замена каждого binding-поля вызывает InputError. `git diff --check` без ошибок.
- [ ] **Step 6: Commit.** `git add pyproject.toml .gitignore dorado_protocol tests docs` затем `git commit -m "feat: define validated Dorado protocol requests"`.

### Task 2: Строгий parser и происхождение fixtures

**Files:** Create `dorado_protocol/responses.py`, `tests/test_responses.py`, `tests/fixtures/index.json`, `tests/fixtures/responses/`; modify types.py только для согласованных диагностик.

**Interfaces:**
- Consumes: RequestSpec, WireReply, ProtocolResult и остальные типы Task 1.
- Produces: `interpret(request: RequestSpec, reply: WireReply) -> ProtocolResult`; parser не вызывает сеть и не мутирует inputs.

- [ ] **Step 1: Создать fixtures и RED parser tests.** Index schema_version=1: каждый файл имеет `origin=synthetic`, file hash, pinned source path/line span, `purpose`, `invented_values`, `assumptions`. Полные ID/name/parent/WWN/health/state в строках snapshot — синтетическая форма, не vendor capture. Повреждённые JSON хранятся как raw bytes, чтобы duplicate keys не исчезли при чтении metadata.

Базовые fixtures: snapshot active, rolling back, old rollback complete,
create reply, empty name lookup, multiple name lookup, code-only ack, vendor
error. Negative cases изменяют копию или отдельный raw fixture; original не
перезаписывается. `test_fixture_provenance_is_complete` проверяет отсутствие
пропусков и несовпавших hashes.

```python
def test_old_rollback_is_only_an_observation(self):
    result = interpret(get_by_id(OP_ID, IDENTITY), OLD_ROLLBACK_REPLY)
    self.assertEqual('api_acknowledged', result.submission.kind)
    self.assertEqual(100, result.snapshot.rollback_rate)
    self.assertFalse(hasattr(result, 'status'))
    self.assertFalse(hasattr(result.snapshot, 'succeeded'))

def test_missing_endtime_is_explicit(self):
    result = interpret(get_by_id(OP_ID, IDENTITY), NO_ENDTIME_REPLY)
    self.assertIsNone(result.snapshot.rollback_endtime)
    self.assertIn('ROLLBACKENDTIME', result.snapshot.missing_fields)
```

Test constants — WireReply с HTTP 200, complete synthetic bytes,
network_attempted=True, diagnostic=None. Snapshot active fixture содержит
HEALTHSTATUS='1', RUNNINGSTATUS='43'; rolling_back='44'. OLD_ROLLBACK_REPLY
имеет ROLLBACKRATE='100' и произвольную непустую synthetic endtime, не `'-1'`.
Вариант с другим OP_ID даёт тот же набор наблюдений, без присвоения завершения.

`test_json_rejections` покрывает duplicate keys на обоих уровнях, invalid UTF-8,
NaN/Infinity, trailing content, non-object root и nesting, превышающий возможности
decoder: последний даёт unreadable_reply, а не uncaught RecursionError. `test_envelope_types` проверяет
отсутствие error/code, bool/float/string code, nonzero code с ложным data.
`test_identity_mismatch` меняет каждое ожидаемое ID/name/parent/WWN и проверяет
unreadable_reply + identity_mismatch + snapshot=None.

`test_action_specific_data_shape` требует object для create/get_by_id и list
для get_by_name, допускает отсутствие data только для четырёх code-only ack.
Create проверяет ID/name/parent; state fields обязательны для GET observation,
а отсутствие их в create ack не создаёт SnapshotObservation.

`test_lookup_cardinality` проверяет пусто→not_observed, 2..99→ambiguous_identity,
100→incomplete_listing, >100→unreadable_reply/invalid_data; одна запись должна
совпасть по identity. `test_snapshot_fields` проверяет отсутствующие и неверно
типизированные health/state, unknown values, rate -1/101/bool/int вместо string,
пустой/non-string endtime. Отсутствующие optional rollback fields перечисляются
в missing_fields; неверно присутствующие дают unreadable_reply.

- [ ] **Step 2: Запустить RED.** `python3.11 -m unittest tests.test_responses -v`; новое поведение отсутствует, логи сохранены.
- [ ] **Step 3: Реализовать interpret.** Сначала request validity и transport diagnostics; потом HTTP status; далее JSON/envelope и action-specific data. local failure→local_rejected, transport failure→transport_unknown, framing/size/encoding failure→unreadable_reply, non-2xx→http_error, nonzero code→api_error. При успехе возвращать api_acknowledged и, для GET, отдельное observation. Нормализация never coerces bool/string/int между типами. Ненормальный health или неизвестный running→unrecognized_state.
- [ ] **Step 4: GREEN и purity.** `python3.11 -m unittest tests.test_requests tests.test_responses -v`; результаты совпадают с literal expectations, повторный вызов не меняет inputs. Добавить `test_parser_has_no_network_or_terminal_mapping`: сетевые entrypoints под mock fail-on-call; статический публичный API не экспортирует Cinder hooks/status mapping.
- [ ] **Step 5: Commit.** `git add dorado_protocol tests/fixtures tests/test_responses.py` затем `git commit -m "feat: parse Dorado replies as bounded observations"`.

### Task 3: Одна HTTP попытка и настоящие отказные сценарии

**Files:** Create `dorado_protocol/transport.py`, `tests/{http_server,network_cases,process_runner,test_transport}.py`.

**Interfaces:**
- Consumes: Task 1 validate_request/types; Task 2 interpret/fixtures.
- Produces: `send_once(endpoint: LabEndpoint, request: RequestSpec, *, token: str) -> WireReply` и `validate_endpoint(endpoint: LabEndpoint) -> None`.
- Produces test support: `run_network_case(case_id: str, output: Path, *, deadline: float = 15.0) -> dict`; поддерживаемые case IDs фиксируются в `network_cases.CASES` и не принимают внешних адресов.
- Produces test support: `ScriptedServer(script: tuple[dict, ...], audit_path: Path)` как context manager с `.endpoint: LabEndpoint`; сервер запускается thread внутри case process и пишет durable audit на каждый полностью принятый запрос.

- [ ] **Step 1: Создать RED тесты без транспортного mock.** ScriptedServer использует raw socket для неверных headers/chunks; не нормализует их через HTTP framework. Endpoint содержит prefix `/lab-device`, выбранный лабораторией. Сервер читает целиком ожидаемый запрос, записывает method/path/body hash, flush, затем исполняет fault. Для частичного request сохраняет отдельную incomplete запись.

Обязательные cases: good seven actions, complete chunked body, disconnect
после приёма, read timeout, HTTP 301/307/308, 401/403/500, vendor error, short
Content-Length, missing zero chunk, duplicate CL, TE+CL, unknown transfer/content
encoding, >1 MiB response, absent framing, trickle. Все действия и размеры
задаются фиксированным script; ожидания не вычисляются клиентским builder.

```python
def test_lost_reply_does_not_retry(self):
    case = run_network_case('lost_reply', self.output)
    self.assertEqual(1, case['complete_requests'])
    self.assertEqual('transport_unknown', case['submission_kind'])
    self.assertEqual(0, case['login_requests'])

def test_explicit_second_call_is_a_second_attempt(self):
    case = run_network_case('two_explicit_calls', self.output)
    self.assertEqual(2, case['complete_requests'])
```

`test_redirect_and_auth_error_never_resubmit` проверяет audit основного server,
второй локальный sentinel server из Location и отсутствие login routes;
результат http_error. `test_forged_request_is_rejected_before_connect` патчит
connect на fail-on-call и передаёт все подмены Task 1.

`test_endpoint_and_prefix_validation` отклоняет external/localhost/IPv6,
credentials, неявный порт, query/fragment/path в URL, bool port через некорректный
объект и prefix с `..`, percent escapes или CR/LF. Prefix пуст или начинается
с `/`, состоит из тех же ASCII ID segments, без trailing slash, max 256 chars.
Token — непустой ASCII до 1024 chars без C0/DEL; он проверяется до connect.

`test_proxy_env_is_ignored` ставит HTTP_PROXY/HTTPS_PROXY/ALL_PROXY на второй
sentinel server; адресат остаётся первым. `test_wire_limits` покрывает 1 MiB и
1 MiB+1 bytes, включая chunked. `test_trickle_is_failed_by_case_deadline`
проверяет failure, завершение собственного child и сохранение audit; внутренняя
test assertion проходит, хотя контролируемый дочерний сценарий закончился
ожидаемым timeout. Это не PASS для зависшего transport и не Linux skip.

- [ ] **Step 2: Запустить RED на Linux.** `python3.11 -m unittest tests.test_transport -v`; сетевые проверки выполняются в собственных child process. При отсутствии Linux/loopback — environment blocker, не GREEN.
- [ ] **Step 3: Реализовать validation и send_once.** Только `HTTPConnection('127.0.0.1', port, timeout=3)`; новая connection, один request, close в finally, debuglevel=0, без urllib/proxy/retry/login. Перед соединением повторно validate RequestSpec и endpoint/token; invalid input возвращает WireReply diagnostic с network_attempted=False. Признак становится True непосредственно перед сетевой попыткой, а не после успешной отправки.
- [ ] **Step 4: Реализовать чтение framing.** Проверить исходный список headers до тела: один десятичный неотрицательный CL или ровно TE=chunked; оба/дубликаты/absence отвергать. CL >1 MiB отвергать; для CL сверять фактически прочитанные bytes, для chunked требовать финальный chunk. Чтение максимум 1 MiB+1; compressed response не распаковывать, вернуть unsupported_encoding. Partial/malformed body никогда не отдавать parser как успешный reply. Exception type преобразовать в allowlisted diagnostic, не сохранять repr/token.
- [ ] **Step 5: Реализовать владение case process.** Использовать `subprocess.Popen` с конкретным interpreter и `start_new_session=True`; передавать только case ID/output. Parent ждёт 15 секунд, затем TERM и ограниченный grace 1 секунду, затем KILL своего child и wait. Общая верхняя граница cleanup — ещё 2 секунды. Не искать процессы по имени и не использовать PID из evidence. Server thread принадлежит child, чужих процессов нет. Fixed test data не приходит из production endpoints.
- [ ] **Step 6: GREEN и audit assertions.** `python3.11 -m unittest tests.test_requests tests.test_responses tests.test_transport -v`; каждый network case проходит собственный deadline wrapper. Проверить отсутствие второго запроса, отсутствие токена в logs и нулевые proxy/redirect/login counts. После tests нет живых owned child processes. `git diff --check` без ошибок.
- [ ] **Step 7: Commit.** `git add dorado_protocol/transport.py tests` затем `git commit -m "feat: send one Dorado HTTP attempt with fault coverage"`.

### Task 4: Inventory, отчёт, review и воспроизводимый пакет

**Files:** Create `tools/{__init__,run_checks,report,package}.py`, `tests/test_report.py`, README.md/VALIDATION.md/REVIEW.md/REVIEW-RESOLUTION.md; update provenance/doc copies as needed.

**Interfaces:**
- Consumes: tests Tasks 1–3, fixture metadata, pinned input hashes и test-support results.
- Produces: `discover_inventory() -> tuple[str, ...]`, `run(selected: tuple[str, ...], output: Path) -> int`, `verify_report(output: Path) -> dict`.
- Produces: `package(repo: Path, report: Path, destination: Path, *, unpacked_report: Path | None = None) -> None`; source из чистого committed repo, существующий destination не перезаписывать.
- CLI: `python3.11 -m tools.run_checks --output PATH [--case FULL_TEST_ID ...]`; `python3.11 -m tools.report --verify PATH`; `python3.11 -m tools.package --repo . --report PATH --output PATH [--unpacked-report PATH]`.

- [ ] **Step 1: Добавить RED report/package tests.** `test_inventory_is_fixed_before_execution` сравнивает discovered IDs и записанный inventory; test cases имеют уникальные IDs. `test_incomplete_or_failed_inventory_cannot_pass` подменяет completed/passed, skip, unexpected failure и subtest result, пересчитывает hashes и требует rejection. `test_subset_is_not_complete` подтверждает suite_complete=False при успешном subset. `test_existing_or_symlink_output_is_rejected` сохраняет исходный каталог нетронутым. `test_package_uses_committed_source` сравнивает архив с git archive и rejects dirty repo. `test_package_rejects_untested_program_bytes` меняет, добавляет и удаляет по одному исполняемому файлу после прогона; пакет отвергается, даже если новый commit чистый.
- [ ] **Step 2: Запустить RED.** `python3.11 -m unittest tests.test_report -v`; отсутствующие новые modules/поведение, не случайные ошибки окружения.
- [ ] **Step 3: Реализовать runner и verifier.** Записать sorted full inventory до первого теста, затем события start/result/skip/subtest failure. Parent test ID с любым failed subtest считается failed; interrupted started ID остаётся incomplete. Verifier пересчитывает hashes и множества из events, проверяет source/fixture manifests и false qualification flags. Hash manifest покрывает каждый файл кроме себя. Выводы о fault берутся из сохранённых parent assertions и server audit; проверка hashes не называется доказательством hardware или внешней подписью.

Run output: `inventory.json`, `events.jsonl`, `results.json`, `cases/`,
`SHA256SUMS`. Сохранять Python/platform, package commit/tree, input hashes и
выбранный case prefix. Отдельный `tested-source.json` перечисляет точные hashes
всех файлов `dorado_protocol/`, `tests/` (включая fixtures), `tools/` и
`pyproject.toml`, без cache/pyc. Он снимается до тестов и сравнивается после них;
дрейф во время прогона проваливает run. Commit/tree записываются как metadata,
а не как замена хешам фактически исполненных файлов. До candidate commit
допускается dirty source с честной меткой `source_clean=false`.

Упаковщик требует чистый финальный commit и точное совпадение полного набора
программных файлов/хешей с успешным report. Изменение только README/review/docs
не требует нового прогона программного suite; новый, удалённый или изменённый
файл из tested-source требует нового полного report. `results.json` содержит expected/selected/completed/
passed/failed/skipped IDs и suite_complete. Полнота требует Linux+Python 3.11,
все expected IDs passed, отсутствуют skips/failures/incomplete. Exit 0 — выбранные
тесты прошли; 1 — failure/incomplete/invalid report; 2 — input/preflight error.
Успешный subset имеет exit 0 и suite_complete=False. Тесты report используют
изолированные minimal fixtures, а не рекурсивный запуск полного runner.

- [ ] **Step 4: GREEN отчёта и единый run guide.** `python3.11 -m unittest tests.test_report -v`; README содержит подготовку Linux/Python, один полный CLI запуск, subset, verification, ограничения single-call и absence hardware qualification. Full suite не требует установки private SDK, libvirt или OpenStack. Уточнить, что wheel библиотеки не содержит test CLI, полный комплект распространяется source archive.
- [ ] **Step 5: Провести полный Linux gate.** `python3.11 -m tools.run_checks --output results/final-1`, затем `python3.11 -m tools.report --verify results/final-1`; expected exit 0, suite_complete=True, zero skipped/failed/incomplete, false qualification flags. Отдельно `python3.11 -m compileall -q dorado_protocol tests tools` и `git diff --check`. Сверить исходные HEAD/status Nova/Cinder/L1, без перезапуска их test suites.
- [ ] **Step 6: Commit кандидата.** `git add tools tests README.md VALIDATION.md docs dorado_protocol` затем `git commit -m "test: qualify and document the synthetic Dorado protocol kit"`. VALIDATION содержит реальные counts и ссылки на логи, а не ожидаемые цифры.
- [ ] **Step 7: Независимый review всего изменения.** Для Native — один свежий reviewer по полному diff/spec/plan/Review Focus и evidence. Для subagent-driven — итоговый whole-branch review после reviews задач. Обоснованные находки исправить с RED/GREEN; после изменённого поведения повторить затронутые проверки и финальный CLI gate. Сохранить review и решения; не объявлять vendor support по результату review.
- [ ] **Step 8: Финальный commit и кандидат поставки.** Commit проверенных review fixes/документов, затем `python3.11 -m tools.package --repo . --report results/final-1 --output results/package-candidate`. Если fixes потребовали нового полного report, передать его фактический путь. Проверить tested-source против финального commit, hashes, точное соответствие source archive этому commit и ссылки документации. Перенесённый report должен пройти verifier без исходного absolute root.
- [ ] **Step 9: Проверить распаковку и сформировать окончательную поставку.** В новом disposable каталоге распаковать source candidate, запустить полный CLI на Python 3.11/Linux и verifier; сохранить отдельный unpacked report, не переписывая первый. Source archive не имеет `.git`: runner записывает commit/tree/source_clean=null и source_origin=`unversioned_source`, не ищет Git в родительских каталогах. Пакет связывает этот run с финальным commit через полное совпадение tested-source, а не через заявленный SHA. После zero skips/failures и проверки отсутствия токенов вызвать package повторно с НОВЫМ destination `../../deliverables/dorado-checkpoint-protocol-d1p`, указав `--unpacked-report` с полученным путём. Он включает оба reports до построения SHA256SUMS; source archive должен побайтно совпасть с candidate. Проверить финальные hashes и ссылки без повторного запуска неизменного кода. Отдельно сформулировать D1-P complete, D1/V1 not qualified.

## Сверка плана со спецификацией

| Требования spec | Исполнение |
|---|---|
| §1–4 scope, independent package, source pins | Global Constraints, workspace, Task 1/4 |
| §5 builders, limits, action binding | Task 1, повторная проверка Task 3 |
| §6 single attempt, timeout, framing, auth/redirect | Task 3 |
| §7 strict JSON, data shapes, identity, old rollback | Task 2 |
| §8 no terminal mapping, no continuation | Types/Task 2 public API audit, Task 3 read-only audit |
| §9 synthetic provenance, independent HTTP proof, reports | Tasks 2–4 |
| §10 Linux gate, review, unchanged products | Task 4 Steps 5–9 |

Self-review плана: имена межзадачных интерфейсов сведены в один контракт;
все пять Review Focus имеют named tests; выбранные значения перенесены из spec.
Новые implementation details (diagnostic enums, fixture prefix, report layout)
не добавляют runtime capability и не закрывают vendor API gaps.

## Передача к исполнению

Пользователь подтвердил Native; создание отдельного repo и исполнение разрешены. Root `docs/` не является Git repo;
текущий план сохранён как файл. Его утверждённая копия будет включена в первый
commit самостоятельного пакета при исполнении.
