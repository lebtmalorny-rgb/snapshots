# N1: модель и управление Instance Checkpoint в Nova

Дата: 2026-10-07. Статус: **согласована пользователем; N1 ещё не реализован**.

Основа: [общая архитектура](2026-10-06-instance-checkpoints-dorado-tatlin-design.md)
и завершённый C1 (`deliverables/cinder-checkpoint-contract-c1/README.md`).
Это детализация следующего пакета, а не изменение уже поставленной серии C1.

## 1. Краткий вывод и цель

Рекомендуемый следующий пакет — **N1, общий контур Nova**: пользовательский API,
журнал checkpoint/operation в cell DB, дедупликация запросов, резервирование
проектных лимитов между cells, защита ВМ и восстановление состояния журнала.
Он создаёт проверяемую основу для N2: сохранения RAM, libvirt и работы с Cinder.

Конечный пользовательский сценарий уже определён: сохранить работающую ВМ,
продолжить работу, вернуть ту же ВМ к выбранной точке вместе с RAM и всеми
дисками. Сохраняются server/volume/port UUID и MAC; attachment UUID может
измениться. Целевые массивы — Huawei Dorado и YADRO TATLIN.

N1 **не создаёт работающие RAM snapshots сам по себе**. Наличие маршрутов API
или новых таблиц не означает готовности исполнителя. Без подтверждённого N2
capabilities возвращает `supported=false`, изменяющий запрос отклоняется
до резервирования квот и побочных эффектов. Тестовый исполнитель доступен
только в тестах, а не через production config.

Численный бюджет паузы, RAM-store/key management и профили массивов относятся
к N2/D1/D2/V1. N1 не обещает время остановки уровня VMware.

## 2. Почему такой порядок

| Вариант | Преимущество | Ограничение | Решение |
|---|---|---|---|
| N1 → N2 → подключение подтверждённых D-профилей | Можно работать по имеющемуся коду и тестировать API/конкуренцию без СХД | Полезная пользователю RAM-операция появится после интеграции | Рекомендуется |
| Сначала D1/D2 | Рано проверяется пригодность массива | Нет точных firmware/driver/transport и native API профилей | После получения данных; N1 ими не блокируется |
| Сразу один большой патч Nova + libvirt + массивы | Все части видны вместе | Сложнее отделить дефект журнала от RAM/SAN; больше непроверенных предположений | Не выбирать |

Архивы и `sources/` остаются эталоном. Реализация N1 после согласования
выполняется в отдельном checkout `work/nova-checkpoints`, отдельной ветке.
Готовый C1 не изменяется без конкретного совместимого дополнения с тестами.

## 3. Проверенная исходная точка

Приведённые значения прочитаны из локального архива и исходников, а не
получены от работающего OpenStack.

| Объект | Проверенное значение |
|---|---|
| `nova-pvs.zip` SHA-256 | `5e20485c94a525a758141bf1d2dadfd78c88c9166fac704e8e18346f207957a7` |
| Revision в комментарии ZIP | `21072e1cd39a70ac5490624177c41e4fb116da9e` |
| Nova API maximum | `2.105` |
| Compute RPC maximum по service history | `6.4` |
| Nova service version | `68` |
| Conductor compute_task RPC | `1.25`; общий conductor RPC — `3.0` |
| API DB migration head | `cdeec0c85668` |
| Cell/main DB migration head | `2903cd72dc14` |
| Instance OVO | `2.8`; менять его поля для checkpoint не требуется |
| C1 commit | `d2d14a5251fa7f61771f83bfbf50f12215dd051d` |
| Частный Cinder контракт | API `3.72`, Volume RPC `3.21`, contract version `1` |

В Nova отсутствует готовая сущность InstanceCheckpoint. `check_instance_state`
проверяет загруженный OVO. `_instance_update` поддерживает атомарные expected
states, но сам по себе не реализует долгоживущую защиту checkpoint и
взаимное исключение с поздним RPC worker. Штатный `QUOTAS` содержит countable
resources и варианты DbQuotaDriver/UnifiedLimitsDriver; его нельзя считать
готовым атомарным RAM-store reservation между двумя БД.

## 4. Граница N1/N2

| N1 реализует | N2 реализует поверх N1 |
|---|---|
| Cell routing, policy, schema и безопасные ответы | Фактическую проверку libvirt/QEMU/устройств |
| Checkpoint, operation, effect-step, ссылки на диски/артефакты | RAM save/restore, подготовку и проверку файлов |
| Атомарный admission и guards обычных операций | Отключение/переподключение через os-brick и Cinder attachments |
| Глобальные durable quota holds и их reconciliation | Cinder reserve/execute/query/cleanup/release |
| Контракт executor и RPC, базовый отказ `unsupported` | Production executor и доказательства результатов его шагов |
| Query-only восстановление незавершённого журнала | Query фактического состояния libvirt/store/Cinder |
| Fake lifecycle и отказные тесты на SQL/WSGI/RPC | Стендовые проверки RAM/дисков/паузы |

N1 не сохраняет RAM, не вызывает массивы, не управляет libvirt и не выдаёт
фиктивный `available`. Переход в `available` в тестах допустим только после
всех обязательных подтверждений fake executor. В production такое подтверждение
базовый executor дать не может.

## 5. API и авторизация

Предлагается частная микроверсия Nova **2.106**, без изменения поведения
маршрутов до 2.105. Совпадение номера с upstream не означает совместимость.
Все UUID берутся из server-side данных или проверенной схемы запроса.

| Метод и путь относительно `/servers/{server_uuid}` | Результат |
|---|---|
| `GET /checkpoint-capabilities` | `200`, supported и стабильные reason codes |
| `POST /checkpoints` | `202`, checkpoint UUID и operation UUID |
| `GET /checkpoints` | `200`, список с `limit`/`marker` |
| `GET /checkpoints/{checkpoint_uuid}` | `200`, состояние и безопасная текущая operation |
| `POST /checkpoints/{checkpoint_uuid}/action` | `202` для `revert` |
| `DELETE /checkpoints/{checkpoint_uuid}` | `202`, операция удаления |
| `GET /checkpoint-operations/{operation_uuid}` | `200`, текущий/конечный результат, в том числе после удаления checkpoint |

Create: `{"checkpoint":{"name":"A","request_id":"UUID","include_memory":true}}`.
`name` необязателен, максимум 255 символов; это отображаемое имя, не путь.
`include_memory` необязателен и при отсутствии равен true; false отклоняется.
Все допустимые диски включаются обязательно. Дополнительные поля, quiesce,
выбор дисков и запрос другого target state дают `400`.

Revert: `{"revert":{"request_id":"UUID"}}`. DELETE требует UUID в
`OpenStack-Checkpoint-Request-ID`. Пропущенный/некорректный идентификатор — `400`.
Откат успешной точки заканчивается работающей ВМ; остановленный target state
в этом профиле не поддерживается.

Policy: `os_compute_api:server_checkpoints:{capabilities,index,show,create,revert,delete,operation_show}`.
Read использует project reader, изменение — project member; target содержит
project UUID текущего server. Проверка project/server/resource обязательна
помимо role policy; обычный admin flag не обходит guard или подменяет владельца
существующей операции. Чужой ресурс даёт `404`, недостаточная роль в своей
области — `403`. Нужный cell определяется через InstanceMapping; запретить
cell0, ещё не назначенный cell и изменение данных при недоступной cell.

Дедупликация: уникальный `(project_id, instance_uuid, request_id)`, запись
содержит actor user UUID, действие и canonical payload hash. Один request UUID
нельзя использовать для другого действия/тела. Точный повтор того же actor
возвращает тот же operation UUID без повторного эффекта; другой actor не
приобретает эту operation. Новый авторизованный участник проекта может
создать отдельную revert/delete operation для доступной ему ВМ.

Сначала policy/ownership и поиск точного replay, затем feature/capability
admission. `enabled=false` запрещает новые create, сохраняя чтение, guards,
query и операции над существующими точками при наличии нужного executor.
Повтор DELETE после удаления получает сохранённый operation tombstone.
После удаления самого server этот API возвращает `404`; для операторской
диагностики журнал остаётся в cell, бессрочный публичный API старого server
не вводится.

Ответы исключают host/node, RAM/XML, абсолютные пути, connection_info,
backend_reference, токены и сырые исключения. Ошибки: `400` схема, `403`
policy, `404` область/ресурс, `409` конфликт/неподдержанный профиль, `413`
лимит, `503` недоступная cell/служба. `202` означает durable принятие,
а не завершённый checkpoint.

## 6. Данные и транзакционные границы

Checkpoint и workflow принадлежат cell DB текущей ВМ. API DB хранит только
глобальный учёт лимитов и квотные tickets с привязкой к cell/instance/operation.
Между этими БД нет общей SQL transaction; обещать её атомарность нельзя.

Логические таблицы cell DB:

| Таблица | Существенные поля и инварианты |
|---|---|
| `instance_checkpoints` | UUID, instance/project/creator, name, status, host/node, format_version, canonical device fingerprint, active operation UUID, charged quota ticket UUID; UUID не переиспользуется |
| `instance_checkpoint_disks` | checkpoint UUID, BDM UUID, исходный volume UUID, device/serial/boot index, snapshot UUID, backend profile, Cinder operation UUID; уникальный volume внутри checkpoint |
| `instance_checkpoint_artifacts` | UUID, checkpoint, kind, logical object ID, expected upper bound, actual bytes/checksum, state; не произвольный пользовательский путь |
| `instance_checkpoint_operations` | UUID, instance/project/actor, request UUID/hash, kind, generation=1, executor identity, state, phase, исходное vm/power state, безопасная ошибка; уникальность request key, terminal tombstones |
| `instance_checkpoint_steps` | UUID, operation, immutable phase/payload hash, executor, status, результаты; claim фиксируется до эффекта, уникальность шага операции |
| `instance_checkpoint_guards` | instance UUID PK, active checkpoint operation либо ordinary worker claim, persistent checkpoint count, guard token; CAS под блокировкой той же Instance |

Новые OVO версионируются отдельно и сохраняются через явные DB transitions;
неограниченные generic save для operation/step запрещены. Изменять обычный
Instance OVO только ради хранения checkpoint UUID не требуется. Миграции
добавляют новые таблицы; существующие записи Instance/BDM не переименовываются.

Порядок local locks: Instance → instance guard → checkpoint → operation →
steps/disks/artifacts в порядке UUID. Admission проверяет актуальные task/vm
state, host/node, BDM/network/share fingerprint и отсутствие текущего worker.
Запись operation, links и checkpoint task_state выполняется одной cell
transaction. Нельзя держать SQL transaction через RPC, Cinder или libvirt.

`task_state` — отображение операции для существующих клиентов, а не единственный
источник владения. Только текущая operation/generation может изменить свои
фазы и освободить guard. Публичного force-unlock и автоматического lease
expiration нет. Архивация/purge не удаляет активные ссылки и dedup tombstones.

## 7. Лимиты и учёт между cells

Предлагается отдельный минимальный checkpoint quota ledger в API DB:
проектная строка usage под row lock и tickets на создаваемые checkpoints.
Он не меняет обычные квоты instances/cores/ram, не маскируется под уже готовую
поддержку Keystone Unified Limits и не добавляет новый публичный quota API.

Ограничения N1 задаются одинаковыми настройками всех API/conductor:
`max_checkpoints_per_instance=8`, `max_checkpoints_per_project=100`,
`max_artifact_bytes_per_project=0`. Ноль bytes означает, что новые create
не разрешены до явного назначения положительного бюджета. Отрицательные
значения и unlimited в N1 не поддерживаются. Изменение лимитов вниз не
удаляет существующие данные; новые reservations запрещены сверх нового
предела. Согласованность конфигурации всех писателей — условие эксплуатации.

Bytes учитывают все RAM/runtime/config artifacts и staging; Cinder volumes
сюда не входят. До записи файла executor N2 обязан получить верхнюю границу
объёма и квотное разрешение на неё. Любое увеличение резервируется до записи;
при отсутствии достоверной границы create отклоняется. Свободное место
проверяется отдельно и не заменяет quota. Учёт actual bytes допускает
уменьшение после подтверждённой публикации и удаления временных копий.

Протокол admission:

1. После read-only preflight атомарно зарезервировать ticket в API DB;
   его UUID связан с request/operation и учитывается во всех cells проекта.
2. Ticket неизменно привязан к cell UUID из InstanceMapping. Под local locks
   cell повторно сверяет mapping/fingerprint и принимает operation
   с ticket UUID либо записывает окончательный отказ/отмену этого admission.
3. Подтвердить ticket; зарезервированный объём до и после подтверждения
   учитывается одинаково, поэтому потерянный ответ не создаёт свободную квоту.
4. Только после durable admission всех частей разрешить dispatch.

Нельзя освобождать ticket по timeout, отсутствию ответа или однократному
`operation not found`: запоздавший admission ещё может прийти. Перед refund
cell должна атомарно записать cancellation tombstone под ключом operation
либо подтвердить терминальную очистку уже принятой операции. Поздний admission
по отменённому UUID отклоняется. Недоступная cell удерживает ticket.

После delete/abort refund выполняется ровно один раз по подтверждённому
удалению всех artifacts и дисковых ссылок. Quota reconcile безопасно повторяем,
может оставлять консервативно завышенный usage, но не выдавать занятую квоту.
Проектная сериализация проверяется минимум двумя независимыми cell DB.

## 8. Защита от обычных операций

| Путь | Во время checkpoint operation | Между операциями при живых checkpoints |
|---|---|---|
| Delete/force delete/soft delete, rebuild, resize/confirm/revert resize | Запрет | Запрет |
| Cold/live/cross-cell migrate, evacuate, shelve/offload, rescue/unrescue | Запрет | Запрет |
| Attach/detach/swap volume, interface, share; runtime SSH key/config-drive change | Запрет | Запрет |
| Start/stop/reboot | Запрет | Разрешены при прежней топологии и обычной отдельной сериализации |
| Pause/unpause/suspend/resume | Запрет | Разрешены как обычные операции; checkpoint create/revert проверяет свои допустимые states |
| Обычный image snapshot/backup и crash dump | Запрет | Штатный путь допустим только без изменения защищённой топологии; Cinder сохраняет собственные ограничения |
| Admin reset state и unlock | Не снимают checkpoint ownership | Не обходят постоянную защиту топологии |
| Read, console, обычная информационная metadata/tags | Не создают side effects checkpoint | Разрешены; старые значения не восстанавливаются |
| Изменение guest-дисков | Останавливается исполнителем N2 по фазам | Разрешено |

`force`/admin не являются обходом. Guard повторяется перед worker I/O;
проверка только в API недостаточна. Ordinary worker получает durable claim
под той же Instance/guard row lock, даже если пришёл запоздавший RPC со старым
OVO. Пока claim занят, checkpoint admission запрещён. Завершение другого
worker не может снять claim; внутренние вложенные вызовы используют свой
непередаваемый пользователем token. Claim не удерживает SQL transaction.
Это не обещание exactly-once для всех старых RPC Nova.

При падении обычного worker claim остаётся до выяснения его результата.
Автоматический startup reset task_state, power-state sync, cleanup, shutdown,
resume/reboot recovery и внешние events не должны оживить ВМ при неизвестном
checkpoint outcome. Для активной checkpoint operation эти пути записывают
наблюдения/ошибки, но не выполняют конфликтующую коррекцию состояния.

BDM, VIF/port identity, shares, flavor/image/device identity защищаются также
в DB mutation paths. Поля статуса обновляются только в допустимом scope.
Совместимость Neutron control-plane за пределами Nova не гарантируется:
удалённый/изменённый порт обнаруживается повторным N2 preflight до disk revert;
security groups используются актуальные, не восстанавливаются из manifest.

## 9. Workflow, RPC и recovery

Checkpoint states: `creating`, `available`, `deleting`, `error`, `deleted`.
Operation states: `reserved`, `running`, `succeeded`, `failed`,
`recovery_required`, `released`. Step states: `prepared`, `dispatching`,
`pending`, `succeeded`, `failed_no_effect`, `unknown`.

N1 проверяет допустимый граф фаз и обязательные доказательства; N2 реализует
эффекты следующих фаз:

| Операция | Основной порядок фаз |
|---|---|
| Create | preflight → Cinder reserve → RAM/runtime save и остановка записей → native disk snapshots → подтверждение/publish artifacts → restore текущего runtime → Cinder release → available |
| Revert | preflight сохранённой точки → Cinder reserve → остановка текущей ВМ → физический disconnect и scoped detach → native disk revert всех томов → scoped reconnect → restore сохранённого runtime → Cinder release |
| Delete | Cinder reserve → подтверждённое удаление snapshots → удаление RAM/config artifacts → Cinder release → quota refund и terminal tombstone |

Откат дисков без проверенных artifacts/совпадения устройств запрещён.
Частичный revert/неизвестный эффект оставляет ВМ остановленной и guard занятым.
Неудачный create имеет отдельные cleanup steps; возврат текущей ВМ в работу
и очистка ресурсов допустимы только после выяснения исхода каждого эффекта.

Каждый эффект имеет immutable step UUID. Claim `prepared → dispatching`
фиксируется до вызова executor. Повторный execute шага не повторяет эффект:
для dispatching/pending/unknown разрешён query. `failed_no_effect` означает
терминальное доказательство невозможности позднего эффекта; timeout/404
таковым не является. Поколение 1 неизменно в N1; takeover не реализуется.

Предлагаемые частные версии: Compute RPC **6.5**, service version **69**,
compute_task RPC **1.26**. Общий conductor RPC остаётся 3.0; OVO remoting
использует штатный механизм и отдельные versioned checkpoint objects.
Новый compute_task endpoint принимает идентификатор durable operation,
не произвольный workflow из тела API. Compute methods: `checkpoint_probe`,
`checkpoint_execute`, `checkpoint_query`; ответы имеют строгую schema.

До quota admission read-only probe должен пройти и через новый conductor
endpoint, и через целевой compute. Отсутствующий endpoint, неподдержанная
RPC schema или недоступный сервис дают отказ; локальный `can_send_version`
не считается доказательством наличия удалённого исполнителя.

Conductor координирует и пишет журнал через cell DB/OVO. Compute использует
OVO/conductor remoting; прямой доступ nova-compute к cell SQL не добавляется.
Executor закреплён за host/node из admission, чужой host/generation/step
отклоняется до I/O. Минимальная service/RPC version дополняется положительным
probe живого executor: номер версии сам по себе не означает готовность N2.

После рестарта conductor и compute сначала читают существующий журнал.
Conductor periodic reconciliation не запускает разрушительный шаг повторно;
prepared intent можно доставить до первого claim, claim-нутый — только query.
Никакой timeout не освобождает ownership. Ошибки освобождения квот/guard
исправляются идемпотентными local transitions после terminal proof.
`enabled=false` не выключает это поведение. Старые писатели не допускаются
к эксплуатации checkpoint; RPC gate не доказывает обновление API fleet.

## 10. Граница авторизации с Cinder C1

C1 уже требует пользовательскую и разрешённую сервисную identity; владелец
операции — project/user/service user. N1 хранит ссылки и владельца, но не
сохраняет bearer token, пароль или поддельный Keystone context.

Для N2 остаётся обязательная отдельная проработка фонового доступа к C1
после истечения user token. Само сохранение user UUID не даёт Nova права
получить токен от его имени. Нельзя молча заменить actor сервисным пользователем
или расширить C1 ownership ради удобства retry. N1 recovery своего журнала
этого доступа не требует; production N2 recovery нельзя объявить готовым
до согласования и проверки механизма делегирования либо отдельного строго
ограниченного service-recovery контракта. Неопределённость удерживает защиту.

## 11. Конфигурация и развёртывание

`[checkpoint] enabled=false`, `reconcile_interval=60`; лимиты — раздел 7.
Production executor по умолчанию отсутствует/unsupported независимо от flag.
RAM-store пути и ключи N1 не конфигурирует; это часть N2.

Сначала migrations API DB и каждой рабочей cell на копиях БД, затем полное
обновление API/conductor/compute и иных писателей. До включения нужны N2,
проверенный D-профиль и V1. Downgrade запрещён при живых checkpoints,
обычных worker claims или незавершённых operations/quota tickets.
Старые API до 2.105 сохраняют контракт, но обязаны уважать новые guards.

## 12. Критерии приёмки N1

- Policy/response schemas: чужой project/server, reader/member, admin без
  обхода, отсутствие путей/секретов; старые микроверсии не видят новых routes.
- Две реальные cell DB: правильная маршрутизация, недоступная cell,
  один project quota limit на конкурентные запросы разных cells.
- Request replay после timeout/delete; иное тело/actor; terminal cancellation
  против запоздавшего admission; нет двойного reserve/refund.
- Реальные SQL races на MySQL/PostgreSQL и логика на SQLite:
  checkpoint/ordinary worker имеют одного победителя; старый completion
  не снимает чужой guard; deadlock retry не повторяет backend effect.
- Migration/model sync для API DB и cell DB; версии OVO/RPC/service,
  старые endpoints и неподдержанный executor дают отказ до admission.
- Fake A/B/revert A/delete A сохраняет B и идентификаторы; для каждого
  перехода тест потери ответа до/после durable claim и подтверждения.
- Restart/periodic/power sync не запускает guest после частичного revert;
  выключение флага не снимает защиту; tombstones не исчезают при purge.
- Регрессии ordinary lifecycle, attach/detach, shares, runtime SSH key,
  admin/reset, scheduler/conductor/compute и quota paths.
- Серия патчей воспроизводимо применяется к чистому `nova-pvs.zip`,
  контрольные суммы и итоговый Git tree совпадают с проверенной веткой.

Тестовый fake executor не является доказательством libvirt/RAM/SAN.
Фактические команды, counts, skips и недоступные зависимости фиксируются
при исполнении, а не объявляются заранее. Никакие исходные skips нельзя
превращать в утверждение о полном прохождении непротестированного набора.

## 13. Привязка к локальному коду

| Область | Источник |
|---|---|
| Routes, cell-aware subresource | routes.py (`sources/nova-pvs/nova/api/openstack/compute/routes.py`), server_shares.py (`sources/nova-pvs/nova/api/openstack/compute/server_shares.py`) |
| API versions, policy | api_version_request.py (`sources/nova-pvs/nova/api/openstack/api_version_request.py`), policies/base.py (`sources/nova-pvs/nova/policies/base.py`) |
| Instance admission, runtime key | compute/api.py (`sources/nova-pvs/nova/compute/api.py`) |
| SQL CAS, BDM и Instance | db/main/api.py (`sources/nova-pvs/nova/db/main/api.py`), models.py (`sources/nova-pvs/nova/db/main/models.py`), objects/instance.py (`sources/nova-pvs/nova/objects/instance.py`) |
| Cell mapping и global DB | db/api/models.py (`sources/nova-pvs/nova/db/api/models.py`), instance_mapping.py (`sources/nova-pvs/nova/objects/instance_mapping.py`) |
| Quota paths | quota.py (`sources/nova-pvs/nova/quota.py`), limit/local.py (`sources/nova-pvs/nova/limit/local.py`) |
| RPC и service compatibility | compute/rpcapi.py (`sources/nova-pvs/nova/compute/rpcapi.py`), conductor/rpcapi.py (`sources/nova-pvs/nova/conductor/rpcapi.py`), service.py (`sources/nova-pvs/nova/objects/service.py`) |
| Restart и фоновые действия | compute/manager.py (`sources/nova-pvs/nova/compute/manager.py`), conductor/manager.py (`sources/nova-pvs/nova/conductor/manager.py`) |
| Cinder auth/client | volume/cinder.py (`sources/nova-pvs/nova/volume/cinder.py`), C1 API (`work/cinder-checkpoints/cinder/checkpoints/api.py`) |

## 14. Состояние и следующий шаг

В этом проходе проверены локальные точки расширения, версии и migration heads;
создана спецификация. Production-код, зависимости и поставленный C1 не менялись.
Корень рабочей папки не является Git repo; документ сохранён отдельным MD.

После согласования этого документа — подробный план N1 с RED/GREEN шагами,
точными DB/RPC interfaces, тестовой средой и способом исполнения. Предлагается
продолжить ранее выбранный Native: один исполнитель, последовательные задачи,
одно независимое review всего N1 в конце. Код начинается после рассмотрения
конкретного плана; реальная инфраструктура этим не затрагивается.

План реализации подготовлен: [N1 Implementation Plan](../plans/2026-10-07-nova-checkpoint-model-n1.md). Он рассматривается отдельно перед началом кода; выбранный Native-подход сохраняется.
