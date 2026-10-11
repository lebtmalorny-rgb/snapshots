# Подготовка адаптеров Dorado и TATLIN без стенда

Продолжение от 2026-10-08: [исследование официального Huawei-драйвера без
оборудования](2026-10-08-dorado-without-hardware.md). В отдельном драйвере
поставщика найден native rollback; приведённые ниже выводы о C1.1 относятся
к нашей зафиксированной локальной версии и остаются в силе.

## Краткий вывод

C1.1 и N2 готовы как локальные пакеты. Следующий этап — D1/D2, то есть
реализация native storage effects для выбранного профиля массива. Пользователь
подтвердил отсутствие СХД. Модель, firmware, pool и transport пока не заданы.

Без оборудования можно спроектировать адаптер и проверить его код на моделях
ответов HTTP. Для этого всё равно нужна документация точной версии native API.
Стенд необходим позже для подтверждения фактических свойств snapshot/revert,
поведения при сбоях и полной приёмки RAM checkpoints. Сейчас положительных
vendor profiles нет; этот документ не является утверждённым планом реализации
или свидетельством поддержки массива.

Практический порядок: начать детализацию D1 с Huawei, поскольку в имеющемся
драйвере уже есть native snapshot primitives. Для D2 сначала получить native
snapshot/revert API либо соответствующий TATLIN driver: архивный путь копирует
весь том. Это выбор порядка работы по локальному коду, а не рекомендация СХД
для закупки.

## Уточнение от 2026-10-11: что означает адаптер массива

Драйверы Huawei и YADRO TATLIN уже входят в исходный `cinder-pvs.zip`:
`cinder/volume/drivers/huawei/` и `cinder/volume/drivers/yadro/tatlin_*.py`.
Следующий этап — доработка этих драйверов под checkpoint-контракт с
использованием существующих операций. Разработка всего драйвера с нуля
не требуется; для TATLIN может потребоваться другая версия драйвера или
native API, поскольку архивная реализация снимка копирует весь том.

В базовый `VolumeDriver` патчами C1/C1.1 добавлены
`get_checkpoint_capabilities`, `checkpoint_create_snapshot`,
`checkpoint_revert_to_snapshot`, `checkpoint_delete_snapshot` и
`checkpoint_query_operation`. Huawei/TATLIN их пока не переопределяют:
capabilities сообщает `unsupported_driver`, четыре операционных метода
выбрасывают `CheckpointNotSupported`.

Адаптер должен связать этот контракт с операциями массива, включая сохранение
идентификаторов, откат выбранного снимка и read-only проверку неизвестного
результата без слепого повтора мутации. Наличие обычных `create_snapshot` и
`delete_snapshot` само по себе не обеспечивает эти гарантии. Сохранение RAM
выполняет Nova/libvirt. Поддержка конкретной СХД остаётся неподтверждённой до
проверки соответствующей реализации на выбранных модели и firmware.

## Зафиксированная база

| Компонент | Commit | Tree |
|---|---|---|
| Cinder C1.1 | `d065a55cbe5cb8cd59489752846a41145d72bd27` | `53888d9cea7fa1b12d35a0b9afd894eeee270843` |
| Nova N2 | `ce3c1c474b8fd04b6223cf8cf3af7079345d14b2` | `4d24555083fea5d547fe2e8ebf6ed39937c0733c` |

Все выводы о драйверах ниже относятся к этой локальной версии Cinder.
SHA-256 просмотренных исходников и перечень отрицательных AST-проверок
сохранены в [source evidence](2026-10-08-checkpoint-vendor-source-evidence.json).
Исторические пакеты C1/C1.1/N1/N2 и их manifests остаются самостоятельными.

## Что уже есть в исходниках

| Граница | Huawei | TATLIN |
|---|---|---|
| Snapshot | Native create, затем activate | Временный volume object, создание backend volume, полное копирование |
| Revert | Собственного `revert_to_snapshot` в vendor classes нет | Собственного `revert_to_snapshot` в vendor classes нет |
| Delete snapshot | Stop, затем delete snapshot | Удаление volume, представляющего snapshot |
| Новый checkpoint-контракт | Vendor overrides отсутствуют; наследуется unsupported | То же |
| Snapshot semantics | CoW/ROW для целевого профиля не подтверждены | Архивная реализация — full copy |
| Reconciliation native operation | Полный протокол с durable step UUID отсутствует | Полный протокол с durable step UUID отсутствует |

Основание: Huawei common.py (`../../work/cinder-checkpoint-continuation/cinder/volume/drivers/huawei/common.py`)
`835–877`, TATLIN common.py (`../../work/cinder-checkpoint-continuation/cinder/volume/drivers/yadro/tatlin_common.py`)
`284–344, 438–477`, VolumeDriver (`../../work/cinder-checkpoint-continuation/cinder/volume/driver.py`)
`2265–2296, 2342–2352`.
Huawei driver объявляет version `2.0.9`, TATLIN FC — `1.0`, iSCSI — `1.1`.
Эти строки не устанавливают модель или firmware действующей СХД.

Полное копирование через обычный Cinder fallback не удовлетворяет
`native_snapshot` с `snapshot_semantics=cow|row`. Наличие ClonePair API в
Huawei также само по себе не доказывает безопасный in-place revert:
сохранение исходного LUN, WWN, остальных snapshots и экспортов требует
отдельного контракта. Подмена revert клонированием в этом проекте не выбрана.

## Разрывы, которые надо решить до реализации

### Повтор HTTP-запросов после неопределённого исхода

Huawei `RestClient.do_call()` преобразует transport exception в код ошибки;
`call()` при connection/auth error выполняет login и снова отправляет исходный
запрос. Ограничения только на GET у этого пути нет.
Исходник (`../../work/cinder-checkpoint-continuation/cinder/volume/drivers/huawei/rest_client.py`),
строки `64–110, 159–184`.

У TATLIN `_access_api()` обёрнут в retry по `Exception`, с настраиваемым числом
повторов; HTTP method не исключает mutating calls из этой обёртки.
Исходник (`../../work/cinder-checkpoint-continuation/cinder/volume/drivers/yadro/tatlin_client.py`),
строки `29, 477–510`. Фактическое число повторов зависит от конфигурации.

Для checkpoint effects нужен отдельный режим транспорта: неопределённый исход
мутации сохраняется как unknown, без автоматического повторения. Повторные
наблюдения должны использовать read-only query. Исключение возможно только
при доказанном серверном idempotency contract, привязанном к тому же step UUID
и payload; в просмотренных методах такой контракт не установлен. Изменять
retry-поведение всех обычных операций драйвера для этого не требуется.

### Составные native effects

Huawei snapshot проходит `create → activate`; при исключении activate обычный
метод пробует delete. Этот путь нельзя целиком считать одним атомарным native
эффектом. Аналогично delete состоит из `stop → delete`.

В C1.1 execute может claim только prepared step, а query не выполняет мутации.
Ответ succeeded для snapshot переводит его в available. Поэтому сбой после
create, но до activate требует явного решения: что доказывает готовность
snapshot и кто имеет право выполнить недостающий mutating substep.
Просто найти объект по имени недостаточно; запуск activate внутри query
нарушит существующий контракт.

Если целевой vendor API предоставляет одну атомарную/серверную операцию с
долговечным ID и терминальным readback, использовать её можно после проверки
её семантики. Если API действительно многошаговый, потребуется отдельный
проект durable substeps/continuation. Это пока развилка дизайна, а не решение
добавлять C1.2. До её разрешения соответствующий профиль остаётся unsupported.
Основание: Huawei create (`../../work/cinder-checkpoint-continuation/cinder/volume/drivers/huawei/common.py`)
`851–865`, C1.1 dispatch/query (`../../work/cinder-checkpoint-continuation/cinder/checkpoints/continuation_manager.py`)
`88–142`, step finish (`../../work/cinder-checkpoint-continuation/cinder/db/sqlalchemy/checkpoints.py`)
`451–485`.

### Восстановление связи с native operation

Driver hooks получают durable Cinder **step UUID**. `backend_reference`
сохраняется после ответа драйвера, но существующий query hook принимает
`volume, snapshot, step UUID, action`, без отдельного аргумента native job ID.
Следовательно, требуется доказуемый способ восстановить эту связь после
потери первого ответа или restart: серверный lookup по request key либо
защищённое durable mapping с определённым порядком записи и отправки.

Нельзя считать отсутствие объекта доказательством failed_no_effect: запрос
может ещё исполняться или не быть видимым. Для delete нужна проверка именно
принадлежащего этой операции объекта, а не общий HTTP 404 при произвольном
сбое авторизации/доступа. Для revert требуется подтверждение исходной цели,
выбранного snapshot и завершения конкретной операции.

## Пакет входных данных для будущего профиля

Подготовлен [шаблон данных](2026-10-08-checkpoint-vendor-profile.template.json).
Это анкета исследования, не конфигурация Cinder и не разрешающий profile.

Нужны модель/firmware/API revision, pool и его свойства, FC или iSCSI,
точный driver source revision, лицензии native snapshots, ограничения revert,
документация job/query/idempotency и срока хранения истории операций.
Management credentials и токены в анкету не включаются.

Для каждого API понадобятся описание request/response и обезличенные примеры:
success, pending, explicit rejection, response loss, absent/foreign object,
expired job history, controller failover. Примеры, написанные по спецификации,
помечаются synthetic. Captures с устройства помечаются отдельно; одно не
превращается в другое после успешного unit test.

## Матрица локальных проверок будущего адаптера

| Сценарий | Обязательный результат |
|---|---|
| Неизвестный model/firmware/pool/transport | supported=false до native write |
| Capability не подтверждает хотя бы одно обязательное поле C1 | Отказ admission |
| Exact повтор step UUID/payload | Нет второго внешнего эффекта |
| Тот же key с другим volume/snapshot/action | Отказ до I/O |
| Transport timeout или потеря HTTP reply | unknown; мутация автоматически не повторяется |
| Query при каждом переходе | Ноль mutating запросов |
| Сбой create → activate / stop → delete | Частичный native результат не выдаётся за success/no-effect |
| Native object есть, но его owner/parent/WWN не совпадает | Нет присвоения или удаления чужого объекта |
| Сохранился durable completion, потерялся Nova/Cinder RPC reply | Query подтверждает исходный эффект без нового dispatch |
| Native operation/job исчез из истории | unknown без снятия guards и квот |
| Частичный revert двух томов | Нет reconnect/unpause смешанного набора |
| Drift профиля после reserve | Исполнение отклоняется, прежняя защита сохраняется |
| Перезапуск Cinder до/после ответа массива | Только доказуемое восстановление исходной операции |
| Обновление обычного драйвера | Existing snapshot/volume/attachment regression без изменения legacy semantics |

Здесь описаны тесты, которые предстоит реализовать после фиксации vendor API.
Существующие fake-driver tests C1.1/N2 не засчитываются как эти vendor tests.

## Стендовая квалификация

Будущий тестовый профиль должен подтвердить A → B → revert A → revert B →
повторный revert A, независимое удаление A/B, прежние volume/LUN/WWN identity
и отсутствие повреждения других points. Дополнительно проверяются реальные
lost replies, restart/controller failover, завершённость native writes,
сохранение operation history и отсутствие I/O после доказанного fencing.

V1 отдельно проверяет RAM-only marker, маркеры всех дисков, server/volume/port
UUID, актуальные attachments/network bindings, паузу guest и peak storage.
Hardware acceptance использует явно выделенные disposable объекты. Этот
документ не запускает такие действия.

## Результат текущей подготовки

Сверены production sources C1.1, точки расширения, transport retries и
неопределённость составных native effects. Подготовлены исходная матрица,
анкета профиля и проверяемая привязка к файлам. Product code не менялся.

После появления документации выбранного API можно подготовить детальную
спецификацию одного адаптера и реализовать его локальные protocol tests даже
до получения оборудования. Положительный profile и включение на СХД требуют
последующей стендовой квалификации. Выдуманные endpoints и автозаполнение
capabilities значением true в этот путь не входят.
