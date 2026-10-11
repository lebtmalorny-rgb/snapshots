# OpenStack VM Snapshots с RAM: анализ соответствия и вариант доработки Nova

**Дата анализа:** 2026-10-06  
**Основная версия:** OpenStack 2025.1 Epoxy / Nova 31.x  
**Сравнение:** OpenStack 2025.2 Flamingo / Nova 32.x, 2026.1 Gazpacho / Nova 33.x, 2026.2 Hibiscus / Nova 34.x  
**Цель документа:** оценить соответствие OpenStack требованиям к snapshot/checkpoint ВМ и описать архитектуру доработки Nova для создания, отката и удаления снимков ВМ с оперативной памятью.

---

## 1. Исходное требование

Требуется наличие технологии создания единовременных моментальных снимков ВМ (Snapshot) с поддержкой Copy-on-Write (CoW), включающих:

- конфигурацию ВМ;
- все виртуальные диски;
- независимость от типа устройства хранения данных;
- возможность отката **существующей** ВМ к snapshot;
- откат без создания новой ВМ;
- сохранение уникальных идентификаторов;
- консолидацию snapshot;
- удаление snapshot;
- возможность создания snapshot **с оперативной памятью ВМ**.

По смыслу это требование близко к VMware VM Snapshot / Hyper-V Checkpoint, то есть к **full-system checkpoint**, включающему диски и runtime state ВМ.

---

## 2. Краткий вывод

Стандартный OpenStack Nova в версиях 2025.1–2026.2 **не предоставляет единый VM snapshot/checkpoint**, который одновременно включает:

- RAM;
- CPU/runtime state;
- конфигурацию ВМ;
- root disk;
- все дополнительные диски;
- возможность последующего in-place rollback той же ВМ;
- сохранение всех идентификаторов;
- управление деревом snapshot;
- snapshot consolidation.

В OpenStack эти функции разделены между несколькими механизмами:

1. **Nova `createImage` / server snapshot** — в первую очередь снимок дискового состояния.
2. **Nova `suspend`** — сохранение текущего runtime state, включая RAM, но без создания именованной точки восстановления.
3. **Cinder snapshots** — снимки отдельных volumes.
4. **Cinder group snapshots** — групповые снимки volumes; write-consistency зависит от storage backend.
5. **libvirt/QEMU** — нижележащий гипервизор уже имеет full-system snapshot и memory-state механизмы, но Nova не предоставляет их пользователю как единый VM checkpoint.

Следовательно:

> **Snapshot с RAM как самостоятельный объект восстановления в штатном Nova 2025.1–2026.2 отсутствует.**

При этом технологическая база для реализации такой функции на libvirt/KVM существует.

---

# Часть I. Что есть в OpenStack сейчас

## 3. Nova Snapshot (`createImage`)

Compute API предоставляет:

```http
POST /servers/{server_id}/action

{
  "createImage": {
    "name": "snapshot-name"
  }
}
```

В документации Nova операция snapshot описана как сохранение состояния **instance root disk**.

Для image-backed и volume-backed ВМ поведение различается.

### 3.1. Image-backed VM

Упрощённо:

```text
VM
├── local/root ephemeral disk   -> snapshot в Glance
├── Cinder data volume #1       -> НЕ входит
└── Cinder data volume #2       -> НЕ входит
```

Официальный Compute API прямо указывает, что для image-backed server snapshots подключённых volumes не создаются.

### 3.2. Volume-backed VM

Для volume-backed server `createImage` ведёт себя иначе:

```text
VM
├── Cinder root volume          -> Cinder snapshot
├── Cinder data volume #1       -> Cinder snapshot
└── Cinder data volume #2       -> Cinder snapshot

                 |
                 v

        Glance snapshot image
       + block_device_mapping
```

Nova создаёт snapshots всех подключённых volumes и записывает ссылки на них в `block_device_mapping` свойства результирующего Glance image.

Однако восстановление из такого образа — это штатно **создание нового server** и новых volumes из snapshot, а не rollback существующего server.

---

## 4. Nova Suspend

Nova имеет:

```bash
openstack server suspend <server>
openstack server resume <server>
```

В документации `suspend` описывается как аналог suspend-to-disk:

- ВМ перестаёт потреблять CPU/RAM;
- live running state сохраняется на диск;
- после `resume` выполнение продолжается с сохранённого состояния.

В актуальном коде libvirt-драйвера Nova это реализовано через:

```python
guest.save_memory_state()
```

а в `nova/virt/libvirt/guest.py`:

```python
def save_memory_state(self):
    self._domain.managedSave(0)
```

То есть Nova уже умеет сохранять runtime state через libvirt `managedSave`.

Но это **не snapshot с RAM**, потому что:

- нет множества именованных RAM checkpoints;
- нет дерева snapshot;
- нельзя продолжить работу, затем выбрать старую RAM-точку и откатиться к ней;
- нет REST API для списка/удаления/отката RAM snapshots;
- memory state не связан как единая транзакционная сущность с snapshot всех дисков.

Схема `suspend`:

```text
RUNNING VM
    |
    | suspend
    v
SUSPENDED
 RAM/runtime сохранён
    |
    | resume
    v
RUNNING VM
```

Требуемая модель:

```text
              +--- Checkpoint A: RAM + disks + config
              |
RUNNING VM ---+------------------------------------> VM продолжает работу
              |
              |                           спустя время
              +--------------------------> revert A
                                           |
                                           v
                                  RAM + disks + config
                                  состояния Checkpoint A
```

Это разные механизмы.

---

## 5. Сравнение релизов OpenStack

По состоянию на 2026-10-06 существенного изменения модели snapshot в Nova не произошло.

| Возможность | 2025.1 Epoxy | 2025.2 Flamingo | 2026.1 Gazpacho | 2026.2 Hibiscus |
|---|---:|---:|---:|---:|
| Nova disk snapshot | Да | Да | Да | Да |
| Snapshot root disk | Да | Да | Да | Да |
| Snapshot Cinder volumes для volume-backed VM | Да | Да | Да | Да |
| Attached volumes входят в snapshot image-backed VM | Нет | Нет | Нет | Нет |
| RAM входит в Nova snapshot | **Нет** | **Нет** | **Нет** | **Нет** |
| Suspend сохраняет RAM/runtime | Да | Да | Да | Да |
| Именованные RAM checkpoints | Нет | Нет | Нет | Нет |
| Несколько RAM restore points | Нет | Нет | Нет | Нет |
| Rollback существующей VM к полному snapshot | Нет | Нет | Нет | Нет |
| Full-system snapshot API | Нет | Нет | Нет | Нет |
| Snapshot consolidation уровня VM | Нет | Нет | Нет | Нет |

Версии Nova:

| OpenStack | Nova |
|---|---:|
| 2025.1 Epoxy | 31.x |
| 2025.2 Flamingo | 32.x |
| 2026.1 Gazpacho | 33.x |
| 2026.2 Hibiscus | 34.x |

---

## 6. Соответствие исходному требованию

| Требование | Штатный OpenStack 2025.1–2026.2 | Комментарий |
|---|---|---|
| Создание snapshot ВМ | Частично | Есть Nova `createImage` |
| Snapshot конфигурации VM | Нет как единый объект | Часть параметров присутствует в Nova/Glance metadata, но это не полный restorable VM state |
| Snapshot root disk | Да | Зависит от virt/storage backend |
| Snapshot всех виртуальных дисков | Частично | Volume-backed — существенно лучше; image-backed attached volumes не входят |
| Snapshot независимо от типа storage | Нет | Возможности зависят от backend и драйвера |
| Гарантированный CoW | Нет | CoW — свойство конкретного backend/формата |
| Единовременный snapshot всех дисков | Частично | Cinder group snapshot возможен, но consistency зависит от backend |
| Snapshot с RAM | **Нет** | В Nova snapshot RAM не входит |
| Сохранение RAM текущей VM | Да | Через `suspend` |
| VM продолжает работу после RAM snapshot | Нет | `suspend` останавливает VM |
| Несколько RAM snapshot | Нет | Нет соответствующего объекта/API |
| Откат существующей VM | Нет как единая операция | Есть отдельные механизмы rebuild/Cinder revert |
| Сохранение server UUID при полном rollback | Нет единой гарантии | Возможен в отдельных операциях, но нет full-system rollback |
| Сохранение volume UUID | Частично | Возможно при Cinder revert, но есть ограничения |
| Сохранение всех UUID | Нет | Например, attachment UUID могут быть пересозданы |
| Удаление snapshot | Да | Glance/Cinder |
| Snapshot consolidation | Нет как VM API | Может выполняться внутри конкретного backend |
| Snapshot tree | Нет на уровне Nova VM | libvirt это умеет, Nova не экспонирует |

---

# Часть II. Почему Copy-on-Write нельзя гарантировать средствами одной Nova

## 7. CoW зависит от backend

OpenStack работает с разными типами хранения:

- qcow2;
- raw;
- Ceph RBD;
- LVM;
- NFS;
- iSCSI;
- FC;
- NVMe-oF;
- vendor SAN/NAS;
- другие Cinder drivers.

У разных backends snapshot реализован по-разному.

Примеры:

```text
qcow2           -> backing file + overlay
Ceph RBD        -> RBD snapshot / clone semantics
LVM             -> LVM snapshot
enterprise SAN  -> hardware/native snapshot
NFS             -> зависит от Cinder driver или qcow2-assisted mechanism
```

Поэтому Nova не может универсально гарантировать:

```text
"каждый snapshot физически реализован через один и тот же CoW"
```

Она может гарантировать только семантику операции, если storage backend сообщает требуемую capability.

Для строгого ТЗ правильнее требовать:

> backend должен предоставлять snapshot с CoW/redirect-on-write или эквивалентной технологией без полного копирования данных.

Если требование требует **именно CoW на любом backend**, это невозможно обеспечить только изменением Nova: потребуется capability/контракт на уровне Cinder driver.

---

# Часть III. Cinder snapshots и rollback

## 8. Cinder Group Snapshot

Cinder имеет Generic Volume Groups и group snapshots.

Некоторые storage backends умеют делать **write-consistent group snapshot** нескольких volumes в одной storage-транзакции.

Но документация Cinder явно говорит, что такая возможность зависит от storage driver.

То есть:

```text
volume A ---+
volume B ---+--> backend consistent group snapshot
volume C ---+
```

не эквивалентно:

```text
VM
├── CPU
├── RAM
├── devices
├── config
├── local disks
├── Cinder volumes
└── network state
```

Это только storage-level snapshot.

---

## 9. Cinder revert-to-snapshot

Cinder API поддерживает:

```http
POST /v3/{project_id}/volumes/{volume_id}/action

{
  "revert": {
    "snapshot_id": "<snapshot UUID>"
  }
}
```

Но есть важные ограничения:

- volume должен быть **detached**;
- status volume должен быть `available`;
- документация описывает механизм как revert volume к его **latest snapshot**.

Следствие:

> Для универсального отката произвольного старого VM checkpoint с несколькими Cinder volumes одной доработки Nova недостаточно.

Для полноценного дерева checkpoints потребуется либо:

1. расширять Cinder API/driver contract для rollback к произвольному snapshot;
2. либо ограничить поддержку backend'ами, где такая операция реализуема vendor-native способом;
3. либо удалить более новые snapshots перед rollback старого checkpoint, что меняет семантику дерева и может быть неприемлемо.

---

# Часть IV. Почему доработка Nova технически возможна

## 10. libvirt уже поддерживает full-system snapshots

libvirt различает:

1. **disk snapshot**;
2. **memory/VM state**;
3. **full system snapshot** = disks + VM memory/runtime state.

Документация libvirt указывает, что:

```text
virDomainSnapshotCreateXML()
```

для активного domain без `DISK_ONLY` может создавать **full system snapshot**, содержащий:

- disk state;
- RAM;
- runtime VM state.

Откат выполняется через:

```text
virDomainRevertToSnapshot()
```

libvirt также хранит parent/child отношения snapshots.

Пример XML:

```xml
<domainsnapshot>
  <name>pre-upgrade</name>

  <memory snapshot='external'
          file='/checkpoint-store/vm-uuid/checkpoint-uuid.mem'/>

  <disks>
    ...
  </disks>
</domainsnapshot>
```

Для external disk snapshot:

```text
base.qcow2
    |
    +---- snapshot point
            |
            +---- active-delta.qcow2
```

Это и есть удобная техническая база для функции, отсутствующей в Nova API.

---

## 11. В Nova уже есть часть нужного кода

В текущем `nova/virt/libvirt/guest.py` есть wrapper:

```python
def snapshot(self, conf,
             no_metadata=False,
             disk_only=False,
             reuse_ext=False,
             quiesce=False):

    ...
    self._domain.snapshotCreateXML(device_xml, flags=flags)
```

То есть Nova уже вызывает libvirt `snapshotCreateXML`.

Однако существующий код assisted volume snapshot вызывает его именно так:

```python
guest.snapshot(
    snapshot,
    no_metadata=True,
    disk_only=True,
    reuse_ext=True,
    quiesce=True,
)
```

Ключевое:

```text
disk_only=True
```

Поэтому RAM/runtime state туда не входит.

Кроме того, текущий класс:

```python
LibvirtConfigGuestSnapshot
```

формирует в XML:

- `<name>`;
- `<disks>`.

Но не имеет полноценного поля для:

```xml
<memory snapshot="external" file="..."/>
```

Это одна из конкретных точек расширения.

---

# Часть V. Предлагаемая доработка Nova

## 12. Терминология

Чтобы не смешивать существующий Nova `createImage` с новой функцией, рекомендуется назвать новую сущность:

```text
Instance Checkpoint
```

а не `snapshot`.

Тогда остаются понятные сущности:

```text
Nova image snapshot  = существующий createImage
Instance checkpoint  = новый RAM + disks + runtime state
Cinder snapshot      = snapshot отдельного volume
```

---

# 13. Предлагаемая архитектура

```text
                    +----------------------+
                    |      Nova API        |
                    | Instance Checkpoints |
                    +----------+-----------+
                               |
                               v
                    +----------------------+
                    | Compute API / RPC    |
                    +----------+-----------+
                               |
                               v
                    +----------------------+
                    | Nova Compute Manager |
                    +----+-------------+---+
                         |             |
                 libvirt |             | Cinder API
                         v             v
                +-------------+   +-------------+
                | KVM / QEMU  |   |   Cinder    |
                | RAM + local |   | volume snaps|
                | disks       |   | / rollback  |
                +------+------+   +------+------+
                       |                 |
                       v                 v
                checkpoint store    storage backend
                RAM + metadata      RBD/SAN/LVM/...
```

---

# 14. Новый REST API

Предлагаемый вариант:

## 14.1. Создание checkpoint

```http
POST /servers/{server_uuid}/checkpoints

{
  "checkpoint": {
    "name": "before-upgrade",
    "include_memory": true,
    "quiesce": true,
    "description": "Before kernel update"
  }
}
```

Ответ:

```json
{
  "checkpoint": {
    "id": "5e0adbc9-...",
    "server_id": "0b1e...",
    "name": "before-upgrade",
    "status": "creating",
    "include_memory": true
  }
}
```

---

## 14.2. Список

```http
GET /servers/{server_uuid}/checkpoints
```

---

## 14.3. Просмотр

```http
GET /servers/{server_uuid}/checkpoints/{checkpoint_uuid}
```

---

## 14.4. Откат

```http
POST /servers/{server_uuid}/checkpoints/{checkpoint_uuid}/action

{
  "revert": {
    "power_state": "running"
  }
}
```

---

## 14.5. Удаление

```http
DELETE /servers/{server_uuid}/checkpoints/{checkpoint_uuid}
```

Если удаление требует merge backing chain, Nova должна дождаться окончания block commit/consolidation.

---

## 14.6. Явная консолидация

Опционально:

```http
POST /servers/{server_uuid}/checkpoints/{checkpoint_uuid}/action

{
  "consolidate": null
}
```

Но с точки зрения API лучше сделать consolidation внутренней частью `DELETE`, если пользовательская семантика этого допускает.

---

# 15. API microversion

Для upstream-совместимого изменения потребуется новый Nova API microversion.

Например концептуально:

```text
2.XX - Instance checkpoint API
```

Новый microversion должен:

- объявить новую сущность;
- определить request/response schemas;
- добавить policy rules;
- определить ошибки `409 Conflict`, если VM находится в несовместимом task state;
- определить capability errors.

Пример ошибки:

```json
{
  "conflictingRequest": {
    "code": 409,
    "message": "Instance checkpoint with memory is not supported by the current compute/storage configuration"
  }
}
```

---

# 16. Состояния Nova

Потребуются новые task states:

```text
CHECKPOINT_CREATING
CHECKPOINT_REVERTING
CHECKPOINT_DELETING
CHECKPOINT_CONSOLIDATING
```

Статус checkpoint:

```text
creating
available
reverting
deleting
error
error_reverting
error_deleting
```

Важно блокировать конкурирующие операции:

- resize;
- live migration;
- evacuate;
- rebuild;
- shelve;
- attach/detach volume;
- attach/detach interface;
- second checkpoint create;
- delete VM.

---

# 17. Nova DB / Versioned Object

Рекомендуется создать объект:

```text
InstanceCheckpoint
```

Пример полей:

```text
uuid
instance_uuid
project_id
user_id
name
description

status
include_memory
created_at
updated_at

host
node
hypervisor_type

parent_uuid
generation

memory_backend
memory_location
memory_size
memory_checksum

domain_xml_location
domain_xml_checksum

nvram_location
vtpm_location

quiesced
crash_consistent

failure_reason
```

Отдельная таблица:

```text
InstanceCheckpointDisk
```

Поля:

```text
uuid
checkpoint_uuid

device_name
device_type

backend_type
source_type

is_root
is_cinder_volume

volume_id
cinder_snapshot_id

local_snapshot_name
local_backing_file
active_overlay

snapshot_generation
status
```

Почему объект нужно хранить в Nova DB:

- checkpoint относится к конкретному Nova instance;
- нужен state machine;
- нужны recovery/cleanup после падения `nova-compute`;
- нужна проверка parent/child;
- нельзя полагаться только на libvirt metadata на одном compute host.

---

# 18. Изменение Compute Driver API

В `nova/virt/driver.py` целесообразно добавить интерфейс уровня гипервизора:

```python
def create_checkpoint(
    self,
    context,
    instance,
    checkpoint,
    block_device_info,
):
    raise NotImplementedError()
```

Также:

```python
def revert_checkpoint(
    self,
    context,
    instance,
    checkpoint,
    block_device_info,
):
    raise NotImplementedError()
```

```python
def delete_checkpoint(
    self,
    context,
    instance,
    checkpoint,
):
    raise NotImplementedError()
```

```python
def get_checkpoint_capabilities(
    self,
    instance,
    block_device_info,
):
    ...
```

Первая реализация:

```text
Libvirt KVM x86_64
```

Для остальных virt drivers можно возвращать:

```text
NotSupported
```

до появления соответствующей реализации.

---

# 19. Расширение LibvirtConfigGuestSnapshot

Сегодня класс Nova содержит в основном:

```python
self.name = None
self.disks = []
```

Предлагаемое расширение:

```python
self.name = None
self.description = None

self.memory_snapshot = None
self.memory_file = None

self.disks = []
```

Формирование XML:

```python
if self.memory_snapshot:
    memory = etree.Element(
        "memory",
        snapshot=self.memory_snapshot,
    )

    if self.memory_file:
        memory.set("file", self.memory_file)

    ss.append(memory)
```

Результат:

```xml
<domainsnapshot>
  <name>checkpoint-uuid</name>

  <memory
      snapshot="external"
      file="/var/lib/nova/checkpoints/.../memory.state"/>

  <disks>
    ...
  </disks>
</domainsnapshot>
```

---

# 20. Расширение `Guest.snapshot()`

Существующий wrapper уже имеет:

```text
NO_METADATA
DISK_ONLY
REUSE_EXT
QUIESCE
```

Стоит добавить:

```text
ATOMIC
LIVE
HALT
```

Концептуально:

```python
def snapshot(
    self,
    conf,
    no_metadata=False,
    disk_only=False,
    reuse_ext=False,
    quiesce=False,
    atomic=False,
    live=False,
    halt=False,
):
    ...
```

и:

```python
flags |= atomic and \
    libvirt.VIR_DOMAIN_SNAPSHOT_CREATE_ATOMIC or 0

flags |= live and \
    libvirt.VIR_DOMAIN_SNAPSHOT_CREATE_LIVE or 0
```

Для full-system local-qcow2 checkpoint предпочтительно использовать:

```text
ATOMIC
```

если конкретная версия libvirt/QEMU и disk layout это поддерживают.

---

# 21. Важный нюанс quiesce

libvirt flag:

```text
VIR_DOMAIN_SNAPSHOT_CREATE_QUIESCE
```

требует `DISK_ONLY`.

Поэтому для full-system snapshot нельзя просто убрать:

```text
disk_only=True
```

и оставить `quiesce=True`.

Для full checkpoint лучше сделать quiesce явно:

```text
QEMU Guest Agent
      |
      v
fsFreeze()
      |
      v
pause / snapshot boundary
      |
      v
create checkpoint
      |
      v
fsThaw()
```

В Nova уже есть:

```python
guest.freeze_filesystems()
guest.thaw_filesystems()
```

Поэтому нужная инфраструктура частично существует.

---

# 22. Два режима реализации

Рекомендуется разделить реализацию на два профиля.

---

## 22.1. Profile A — локальные qcow2 disks

Это наиболее реалистичный первый этап.

Поддерживаются:

```text
libvirt/KVM
local qcow2
без PCI passthrough
без vGPU/mdev на первом этапе
без confidential VM
```

Использовать:

```text
libvirt full-system snapshot
+ external RAM file
+ external qcow2 disk snapshots
```

Плюсы:

- libvirt самостоятельно связывает disk snapshot и memory state;
- snapshot tree уже существует;
- `virDomainRevertToSnapshot()` уже реализует rollback;
- VM UUID можно сохранить;
- нет необходимости создавать новую Nova VM.

Это наиболее близкий путь к VMware snapshot semantics.

---

## 22.2. Profile B — Cinder-backed / mixed storage

Этот режим значительно сложнее.

Пример:

```text
VM
├── local ephemeral disk
├── Cinder root volume
├── Cinder data volume
└── RAM
```

Нельзя просто передать все disks в libvirt full-system snapshot, потому что:

- Cinder должен оставаться владельцем lifecycle volume snapshot;
- разные storage drivers имеют разные snapshot capabilities;
- generic Cinder rollback имеет ограничения;
- snapshot topology libvirt и topology Cinder нужно синхронизировать.

Поэтому нужен **Nova-level checkpoint transaction**.

---

# 23. Алгоритм создания checkpoint для mixed/Cinder VM

Один из безопасных вариантов:

```text
1. Проверить capabilities
2. Заблокировать competing operations
3. Freeze guest filesystem через QGA (если включено)
4. Pause vCPU / остановить guest writes
5. Создать snapshots всех Cinder volumes
6. Создать snapshots local/ephemeral disks
7. Сохранить RAM + CPU/device state
8. Сохранить domain XML/NVRAM/дополнительное состояние
9. Resume VM
10. Thaw guest filesystem
11. Пометить checkpoint AVAILABLE
```

Ключевой принцип:

> Все storage snapshots должны быть зафиксированы, пока VM не выполняет новые записи.

Если storage backend не поддерживает atomic group snapshot, остановка guest writes позволяет получить логически согласованный набор snapshots, хотя операция уже не будет storage-atomic в строгом смысле.

---

# 24. Как сохранить RAM при mixed storage

Есть два архитектурных варианта.

## Вариант 1 — libvirt full-system snapshot

Лучше всего подходит, когда все диски управляются самим libvirt snapshot mechanism.

```text
virDomainSnapshotCreateXML()
```

с external memory state.

Это предпочтительный путь для local qcow2.

---

## Вариант 2 — save-to-file как memory checkpoint

Для Cinder-backed VM можно реализовать отдельный memory-state файл через libvirt save API:

```text
virDomainSaveFlags()
```

или эквивалентный wrapper Nova.

Логика:

```text
VM paused
   |
   +--> Cinder volume snapshots
   |
   +--> local disk snapshot
   |
   +--> save VM runtime -> memory.state
   |
   +--> restore VM immediately
   |
   v
VM continues
```

Memory file при этом сохраняется как объект checkpoint.

При последующем rollback:

```text
stop current VM
restore disk state
restore same domain from memory.state
```

Этот путь требует больше кода и особенно тщательной работы с disk paths/domain XML.

---

# 25. Хранилище RAM checkpoint

RAM snapshot может быть очень большим:

```text
VM RAM = 64 GiB
memory checkpoint ~= десятки GiB
```

Поэтому хранить его просто в:

```text
/var/lib/nova/instances
```

опасно.

Предлагаемая конфигурация:

```ini
[checkpoint]
enabled = true

memory_store = shared_fs
memory_store_path = /mnt/nova-checkpoints

max_checkpoints_per_instance = 4

require_quiesce = false
```

Предпочтительные backends:

```text
CephFS
NFS
другое shared POSIX storage
```

Преимущество shared filesystem:

- checkpoint доступен после restart compute;
- можно выполнить restore на другом совместимом compute;
- libvirt получает обычный file path.

Object storage (Swift/S3-compatible) можно использовать как archival store, но перед restore memory file придётся staging'овать на compute node.

---

# 26. Шифрование RAM snapshots

Memory snapshot содержит:

- пароли;
- ключи;
- токены;
- session secrets;
- содержимое приложений;
- криптографические ключи.

Поэтому RAM checkpoint должен рассматриваться как **чувствительные данные**.

Минимальные требования:

```text
encryption at rest
project isolation
strict file permissions
Barbican-managed key
secure deletion where possible
audit logging
```

Рекомендуемая схема:

```text
memory.state
    |
    v
encryption key from Barbican
    |
    v
encrypted checkpoint storage
```

---

# 27. Откат checkpoint

Предлагаемый алгоритм:

```text
1. Lock instance
2. Проверить checkpoint
3. Запретить migration/resize/attach/detach
4. Power off / destroy current libvirt domain
   БЕЗ удаления Nova instance
5. Отсоединить Cinder volumes при необходимости
6. Rollback volumes
7. Rollback local disks
8. Восстановить domain XML
9. Восстановить NVRAM/vTPM state, если поддерживается
10. Reconnect Cinder volumes
11. Restore RAM/runtime state
12. Restore VIF/device configuration
13. Start/resume VM
14. Sync Nova power/task state
```

Ключевой момент:

```text
Nova instance НЕ удаляется.
```

Следовательно, можно сохранить:

```text
instance UUID
project ownership
Neutron port UUID
security groups
metadata
```

---

# 28. Какие UUID реально можно сохранить

### Можно сохранить без создания новой VM

```text
Nova server UUID
Neutron port UUID
Cinder volume UUID
floating IP association
security group association
```

при корректно спроектированном rollback.

### Не все внутренние ID гарантированно останутся теми же

При detach/attach могут измениться:

```text
Cinder attachment UUID
host-side device handles
libvirt runtime device identifiers
некоторые PCI addresses
```

Поэтому формулировку ТЗ:

> "с сохранением всех уникальных идентификаторов"

желательно формализовать.

Например:

> должны сохраняться UUID объекта VM, виртуальных дисков/volumes и сетевых портов.

Если требуется сохранить **вообще каждый внутренний attachment/runtime identifier**, может потребоваться изменение не только Nova, но и Cinder.

---

# 29. Проблема arbitrary rollback Cinder

Это одно из главных архитектурных ограничений.

Допустим:

```text
Checkpoint A
     |
Checkpoint B
     |
Checkpoint C
```

Пользователь хочет:

```text
revert -> Checkpoint A
```

Для local qcow2/libvirt это естественная snapshot-tree операция.

Для generic Cinder `revert-to-snapshot` документация описывает откат к latest snapshot.

Поэтому для полной семантики потребуется расширение Cinder:

```text
revert_to_any_snapshot
```

или backend capability:

```text
supports_arbitrary_snapshot_revert = True
```

Nova должна проверять эту capability до создания checkpoint.

---

# 30. Предлагаемая доработка Cinder contract

Для полной реализации требования желательно добавить capability:

```text
supports_consistent_group_snapshot
supports_arbitrary_snapshot_revert
supports_snapshot_cow
supports_inplace_revert
```

И, желательно, новый group-level revert:

```http
POST /v3/{project_id}/groups/{group_id}/action

{
  "revert": {
    "group_snapshot_id": "..."
  }
}
```

Это позволило бы Nova откатывать набор volumes одной логической операцией.

Без этого universal full-VM rollback для произвольного Cinder backend остаётся негарантированным.

---

# 31. Snapshot consolidation

Для qcow2 external snapshots появляется chain:

```text
base.qcow2
   |
snap-A.qcow2
   |
snap-B.qcow2
   |
active.qcow2
```

При удалении snapshot могут потребоваться:

```text
blockCommit
blockRebase
active block commit
```

В `nova/virt/libvirt/guest.py` уже существуют wrappers для:

```python
blockCommit()
blockRebase()
```

Следовательно, Nova может использовать существующие block-job primitives для consolidation.

Логика удаления:

```text
DELETE checkpoint B
        |
        v
проверка parent/child
        |
        v
commit/rebase disk chain
        |
        v
удаление RAM state B
        |
        v
удаление checkpoint metadata
```

Для backend-native Cinder snapshots consolidation обычно должен выполнять сам storage backend/driver.

---

# 32. Изменения в `nova/virt/libvirt/guest.py`

Потребуются wrappers примерно такого уровня:

```python
def create_full_checkpoint(self, snapshot_conf, atomic=True):
    ...
```

```python
def lookup_snapshot(self, name):
    ...
```

```python
def revert_snapshot(self, name, running=True):
    ...
```

```python
def delete_snapshot(self, name, children=False):
    ...
```

```python
def list_snapshots(self):
    ...
```

Это conceptual API; конкретная реализация должна использовать python-libvirt bindings.

---

# 33. Compute Manager

В `nova/compute/manager.py` появятся RPC entrypoints:

```python
@wrap_exception()
@reverts_task_state
@wrap_instance_event(prefix='compute')
def create_instance_checkpoint(...):
    ...
```

```python
def revert_instance_checkpoint(...):
    ...
```

```python
def delete_instance_checkpoint(...):
    ...
```

Compute Manager должен:

- получить `block_device_info`;
- получить network info;
- проверить instance power/task state;
- вызвать virt driver;
- работать с Cinder;
- обновлять checkpoint status;
- корректно восстанавливаться после частичной ошибки.

---

# 34. RPC API

В:

```text
nova/compute/rpcapi.py
```

потребуется новая RPC version и методы:

```text
create_instance_checkpoint()
revert_instance_checkpoint()
delete_instance_checkpoint()
```

При rolling upgrade:

- API/Conductor должны учитывать минимальную версию compute service;
- новая операция должна возвращать `409/NotSupported`, если target compute старой версии.

---

# 35. Conductor и транзакционность

Для простого MVP orchestration можно оставить в `ComputeManager`.

Для production-варианта лучше сделать durable workflow/state machine через Nova DB и conductor task.

Причина — операция многошаговая:

```text
freeze
pause
snapshot volume 1
snapshot volume 2
snapshot local disk
save RAM
resume
thaw
```

Падение процесса между шагами не должно оставлять VM навсегда:

```text
paused
frozen
half-snapshotted
```

Нужен recovery logic.

Пример:

```text
checkpoint.status = creating
checkpoint.phase  = cinder_snapshots_done
```

После рестарта сервис понимает, какие compensating actions нужны.

---

# 36. Failure rollback при создании

Пример:

```text
Cinder volume #1 snapshot -> OK
Cinder volume #2 snapshot -> OK
RAM save                  -> ERROR
```

Нельзя просто вернуть HTTP error.

Нужно:

```text
1. удалить volume snapshots #1/#2
2. удалить local temporary snapshots
3. thaw filesystem
4. resume guest
5. очистить memory temp files
6. checkpoint -> ERROR
```

То есть checkpoint creation — распределённая транзакция с compensating actions.

---

# 37. NVRAM / UEFI

Для UEFI VM может существовать отдельный NVRAM state.

Если требование трактуется как точное состояние VM, checkpoint должен включать и его:

```text
domain XML
RAM
disk state
NVRAM
```

Иначе после rollback firmware variables могут не соответствовать RAM/disk состоянию.

---

# 38. vTPM

vTPM state также требует отдельной обработки.

Начальный production scope разумно ограничить:

```text
vTPM checkpoint = unsupported
```

пока не будет надёжного snapshot/restore внешнего TPM state.

Иначе можно получить несогласованность:

```text
RAM state
disk state
TPM state
```

---

# 39. PCI passthrough / SR-IOV / vGPU

Физические устройства нельзя считать обычным сериализуемым VM device state.

Поэтому для первой версии следует запретить full checkpoint для:

```text
PCI passthrough
SR-IOV direct port
vGPU/mdev
VDPA
других non-migratable devices
```

Пример проверки:

```text
if instance_has_passthrough_device:
    raise CheckpointNotSupported(...)
```

В дальнейшем можно вводить per-device capability.

---

# 40. Confidential VM

Для VM с:

```text
AMD SEV/SEV-SNP
Intel TDX
```

memory state может быть недоступен или иметь дополнительные ограничения безопасности.

Для первой версии:

```text
include_memory=true + confidential VM -> NotSupported
```

---

# 41. Live migration после checkpoint

Если RAM file хранится локально:

```text
compute-01:/var/lib/nova/checkpoints/...
```

а ВМ мигрировала на:

```text
compute-02
```

restore становится проблемой.

Варианты:

### Вариант A

Запретить live migration, пока у VM существуют RAM checkpoints.

### Вариант B

Использовать shared checkpoint storage.

### Вариант C

Мигрировать checkpoint artifacts вместе с VM.

Рекомендуется B:

```text
CephFS/NFS shared checkpoint store
```

---

# 42. CPU compatibility

RAM checkpoint содержит CPU/runtime state.

Restore на другом compute требует совместимой CPU модели/features.

Следовательно, checkpoint metadata должно содержать:

```text
CPU model
CPU features
machine type
QEMU version compatibility data
libvirt compatibility data
```

И scheduler/restore path должен проверять совместимость.

---

# 43. Сохранение конфигурации VM

Checkpoint должен хранить не только libvirt XML, но и Nova-level snapshot конфигурации:

```text
flavor ID
image ref
metadata
system_metadata
block device mapping
network port mapping
NUMA topology
PCI requests
CPU topology
machine type
firmware type
config-drive properties
```

Но при rollback нельзя слепо переписывать Nova DB старыми значениями.

Нужно заранее определить, какие изменения после checkpoint считаются rollback-able.

Например:

```text
resize после checkpoint -> либо запрещён
attach new volume       -> либо запрещён / требует conflict policy
attach interface        -> либо запрещён / требует policy
```

Самый безопасный вариант:

> Пока у VM существует RAM checkpoint, запретить topology-changing operations либо автоматически инвалидировать checkpoints.

---

# 44. Политика инвалидирования

Например:

```text
операция                         checkpoint
------------------------------------------------
metadata update                  сохраняется
security-group update            сохраняется
resize                           INVALID
attach/detach PCI                INVALID
change machine type              INVALID
volume detach                    INVALID
root volume retype               INVALID
live migration                   зависит от shared store
```

В API checkpoint должен иметь:

```text
status = invalid
reason = "instance topology changed after checkpoint creation"
```

---

# 45. Рекомендуемая state machine

```text
                +----------+
                | CREATING |
                +----+-----+
                     |
             success |
                     v
                +----------+
          +---->| AVAILABLE |<----+
          |     +----+-----+     |
          |          |           |
          |   revert |           | rollback failure recovered
          |          v           |
          |     +----------+     |
          +-----+ REVERTING+-----+
                +----------+

                     |
                  delete
                     v
                +----------+
                | DELETING |
                +----+-----+
                     |
                     v
                  DELETED
```

Дополнительные:

```text
ERROR
ERROR_REVERTING
ERROR_DELETING
INVALID
```

---

# 46. Квоты

Нужны новые quota resources:

```text
checkpoints
checkpoint_memory_gb
checkpoint_disk_gb
```

Иначе пользователь может создать, например:

```text
100 VM × 128 GiB RAM × 10 checkpoints
```

что приведёт к огромному расходу storage.

---

# 47. Пример capability API

Полезно предоставить оператору и клиенту возможность узнать поддержку:

```http
GET /servers/{uuid}/checkpoint-capabilities
```

Ответ:

```json
{
  "checkpoint_capabilities": {
    "supported": true,
    "memory": true,
    "disk": true,
    "atomic": true,
    "quiesce": true,
    "arbitrary_revert": false,
    "consolidation": true,
    "migration_safe": true,
    "unsupported_devices": []
  }
}
```

Это лучше, чем обещать универсальную функцию, которая на разных storage работает по-разному.

---

# 48. Рекомендуемый первый MVP

Первый этап:

```text
Libvirt/KVM only
local qcow2 only
single compute host
shared checkpoint memory store желательно
без Cinder volumes либо Cinder volumes snapshot отдельно без arbitrary rollback
без passthrough
без vGPU
без vTPM
без confidential VM
```

Реализовать:

```text
create
list
show
revert
delete
```

с RAM.

Это уже докажет основной технический механизм.

---

# 49. Второй этап

Добавить:

```text
Cinder-backed VM
Cinder group snapshots
mixed local + Cinder disks
same-volume UUID rollback
shared checkpoint storage
migration compatibility checks
```

---

# 50. Третий этап

Для полного соответствия строгому ТЗ:

```text
Cinder arbitrary snapshot rollback
storage capability negotiation
group-level rollback
backend snapshot CoW capability
consolidation semantics
HA/recovery
NVRAM/vTPM where possible
```

То есть полное соответствие требует архитектурной работы **Nova + Cinder + storage driver**, а не только патча `nova/virt/libvirt/driver.py`.

---

# 51. Наиболее простой proof-of-concept

Для лаборатории можно реализовать POC только в libvirt driver.

Условная последовательность:

```python
def create_checkpoint(instance, checkpoint_id):
    guest = self._host.get_guest(instance)

    path = (
        "/mnt/checkpoints/"
        f"{instance.uuid}/"
        f"{checkpoint_id}.mem"
    )

    snap = LibvirtConfigGuestSnapshot()
    snap.name = checkpoint_id
    snap.memory_snapshot = "external"
    snap.memory_file = path

    # добавить local disks как external snapshots

    guest.snapshot(
        snap,
        disk_only=False,
        atomic=True,
    )
```

Далее:

```python
def revert_checkpoint(instance, checkpoint_id):
    guest = self._host.get_guest(instance)
    guest.revert_snapshot(checkpoint_id, running=True)
```

И:

```python
def delete_checkpoint(instance, checkpoint_id):
    guest = self._host.get_guest(instance)
    guest.delete_snapshot(checkpoint_id)
```

Это **не production implementation**, а минимальный POC для проверки возможности full-system snapshot на конкретном libvirt/QEMU/storage stack.

---

# 52. Почему нельзя просто переделать существующий `createImage`

Не рекомендуется добавлять:

```json
{
  "createImage": {
    "include_memory": true
  }
}
```

Причины:

1. `createImage` создаёт Glance image.
2. RAM checkpoint не является обычным image artifact.
3. У него другая lifecycle-модель.
4. Требуется `revert`.
5. Требуется snapshot tree.
6. Требуется parent/child metadata.
7. Требуется consolidation.
8. Требуются storage references.
9. Требуются крупные memory artifacts.

Поэтому новая сущность `InstanceCheckpoint` архитектурно чище.

---

# 53. Почему Glance плохо подходит для RAM state

Теоретически RAM можно загрузить как отдельный Glance artifact/image.

Но это создаёт проблемы:

- размер;
- latency;
- restore staging;
- image semantics не соответствуют runtime-state;
- lifecycle snapshot tree трудно моделировать;
- нужен специальный access control;
- RAM содержит чувствительные данные.

Поэтому рекомендуется отдельный checkpoint store.

---

# 54. Поведение удаления snapshot

Требование:

> удаление моментальных снимков

может быть реализовано полностью.

Для local qcow2:

```text
leaf snapshot:
    remove overlay / metadata

middle snapshot:
    blockCommit / blockRebase
    wait block job
    update parent-child relation
    remove memory file
```

Для Cinder:

```text
delete Cinder snapshot
```

если он не нужен дочерним checkpoints.

---

# 55. Консолидация

Требование:

> консолидация моментальных снимков

в новой реализации можно определить как:

> сокращение backing chain после удаления checkpoint без изменения текущего guest-visible disk state.

Для qcow2:

```text
block commit / rebase
```

Для storage backend:

```text
driver-specific snapshot deletion/merge
```

Важно: это должно быть частью checkpoint abstraction, а не прямым QEMU API для пользователя.

---

# 56. Application consistency

Даже RAM snapshot не гарантирует application-level consistency для внешних систем.

Например:

```text
VM database
   |
   +--- внешняя DB / distributed cluster
   +--- network storage
   +--- remote API
```

После rollback VM локальное состояние может оказаться старше состояния внешних сервисов.

Нужно разделять:

```text
crash-consistent
filesystem-consistent
application-consistent
```

Предлагаемый API:

```json
{
  "checkpoint": {
    "consistency": "filesystem"
  }
}
```

Для application consistency потребуется hook/guest agent integration.

---

# 57. Рекомендуемые ограничения первой production-версии

Разрешать checkpoint с RAM только если:

```text
hypervisor = KVM/QEMU
libvirt capability = supported
guest has no passthrough devices
guest is not confidential
machine type supported
checkpoint storage available
no migration/resize in progress
all attached volumes support selected snapshot mode
```

При несоответствии вернуть понятную ошибку.

---

# 58. Тестирование

Нужны несколько уровней.

## Unit tests

```text
API schema
policy
DB object
RPC versioning
task state
libvirt XML generation
error cleanup
```

## Functional tests

```text
create checkpoint
VM continues running
modify RAM/disk
revert
verify process state
verify disk state
verify UUID
delete checkpoint
```

## Destructive tests

```text
kill nova-compute during snapshot
kill libvirtd
Cinder timeout
storage snapshot failure
memory store full
blockCommit failure
host reboot during checkpoint
```

## Tempest

Новый Tempest scenario:

```text
boot VM
start process/counter
write disk marker
checkpoint with memory
change marker and process state
revert
verify old process state
verify old disk state
verify server UUID unchanged
```

---

# 59. Проверка именно RAM state

Хороший тест:

В VM запустить:

```python
counter = 12345
```

не записывая значение на диск.

Создать checkpoint.

После этого изменить:

```python
counter = 99999
```

Выполнить rollback.

После rollback значение должно снова быть:

```text
12345
```

и процесс должен продолжить выполнение с соответствующего instruction/runtime state.

Это доказывает, что был восстановлен RAM/CPU state, а не только disk snapshot.

---

# 60. Проверка сохранения идентификаторов

До snapshot:

```text
server UUID = A
root volume UUID = B
data volume UUID = C
port UUID = D
```

После revert:

```text
server UUID = A
root volume UUID = B
data volume UUID = C
port UUID = D
```

Отдельно проверить Cinder attachment ID и решить, является ли его сохранение требованием.

---

# 61. Критерии приёмки доработки

Минимальный набор:

- [ ] Snapshot включает RAM.
- [ ] Snapshot включает CPU/runtime state.
- [ ] VM после snapshot продолжает работу.
- [ ] Можно создать минимум два checkpoints.
- [ ] Можно выбрать checkpoint и выполнить revert.
- [ ] Revert не создаёт новый Nova server.
- [ ] Server UUID не меняется.
- [ ] Disk state соответствует snapshot.
- [ ] RAM state соответствует snapshot.
- [ ] Можно удалить snapshot.
- [ ] Удаление middle snapshot корректно консолидирует disk chain.
- [ ] Ошибка snapshot не оставляет VM frozen/paused.
- [ ] После restart `nova-compute` checkpoint metadata не теряется.
- [ ] RAM artifacts защищены.
- [ ] Storage capabilities проверяются до начала операции.

Для расширенного Cinder режима:

- [ ] UUID volumes сохраняются.
- [ ] Все attached volumes входят в checkpoint.
- [ ] Multi-volume state согласован.
- [ ] Поддержан rollback к выбранному checkpoint, а не только latest.
- [ ] Storage driver декларирует CoW/equivalent capability.

---

# 62. Оценка соответствия после предлагаемой доработки

## Только Nova + libvirt/KVM + local qcow2

| Требование | После доработки |
|---|---|
| Snapshot VM | Да |
| Snapshot RAM | Да |
| Runtime/CPU state | Да |
| Конфигурация | Да |
| Local disks | Да |
| CoW | Да для qcow2 |
| Rollback той же VM | Да |
| Server UUID | Да |
| Delete | Да |
| Consolidation | Да |
| Любой storage backend | **Нет** |
| Все Cinder volumes | **Не полностью** |

## Nova + Cinder extension + capable storage drivers

| Требование | Потенциально |
|---|---|
| Snapshot VM | Да |
| RAM | Да |
| Все disks | Да |
| Multi-volume consistency | Да при capability |
| Rollback той же VM | Да |
| Server UUID | Да |
| Volume UUID | Да при in-place revert |
| Delete | Да |
| Consolidation | Да |
| CoW | Да, если backend capability это гарантирует |
| Независимость от storage | Только при наличии стандартизированного driver contract |

---

# 63. Итоговая рекомендация

Если цель — **закрыть конкретное требование ТЗ**, реализацию лучше разбить на два уровня.

### Уровень 1 — Nova/libvirt VM Checkpoint

Добавить в Nova полноценный:

```text
Instance Checkpoint
```

с:

```text
RAM
CPU/runtime
domain XML
local disk snapshots
revert
delete
consolidation
```

Это технически реализуемо на базе существующих libvirt primitives.

### Уровень 2 — Storage-integrated Checkpoint

Добавить coordination с Cinder:

```text
volume snapshots
group snapshots
rollback
capability negotiation
```

Для строгого требования:

> «вне зависимости от типа устройства хранения данных»

понадобится не только Nova, но и стандартизированный Cinder driver contract.

---

# 64. Формулировка для технического заключения

> **В штатных версиях OpenStack 2025.1 Epoxy, 2025.2 Flamingo, 2026.1 Gazpacho и 2026.2 Hibiscus отсутствует единый механизм full-system snapshot виртуальной машины, включающий оперативную память и обеспечивающий последующий откат существующей VM к выбранной точке восстановления. Nova `createImage` и Nova `suspend` являются раздельными механизмами.**
>
> **Реализация требуемой функции возможна путём расширения Nova Compute API, Compute Manager и libvirt driver с использованием full-system snapshot/save-state механизмов libvirt/QEMU. Для локальных qcow2-дисков такая реализация может обеспечивать RAM checkpoint, in-place rollback, snapshot tree, удаление и consolidation. Для Cinder-backed VM и требования независимости от типа хранения необходима дополнительная интеграция с Cinder и формализация capabilities storage driver, включая consistent group snapshot, in-place revert к произвольному snapshot и CoW/equivalent snapshot semantics.**

---

# 65. Основные источники

1. OpenStack Nova 2025.1 Feature Support Matrix  
   https://docs.openstack.org/nova/2025.1/user/support-matrix.html

2. OpenStack Nova 2025.2 Feature Support Matrix  
   https://docs.openstack.org/nova/2025.2/user/support-matrix.html

3. OpenStack Nova 2026.1 Feature Support Matrix  
   https://docs.openstack.org/nova/2026.1/user/support-matrix.html

4. OpenStack Nova 2026.2 Feature Support Matrix  
   https://docs.openstack.org/nova/2026.2/user/support-matrix.html

5. OpenStack Compute API — `createImage`  
   https://docs.openstack.org/api-ref/compute/

6. OpenStack Cinder API v3 — volume revert to snapshot  
   https://docs.openstack.org/api-ref/block-storage/v3/

7. Cinder driver interface — consistency/group snapshots  
   https://docs.openstack.org/cinder/latest/contributor/drivers.html

8. Cinder Generic Volume Groups  
   https://docs.openstack.org/cinder/latest/admin/groups.html

9. libvirt Snapshot XML format  
   https://libvirt.org/formatsnapshot.html

10. libvirt Domain Snapshot API  
    https://libvirt.org/html/libvirt-libvirt-domain-snapshot.html

11. Nova current source — `nova/virt/libvirt/guest.py`  
    https://github.com/openstack/nova/blob/master/nova/virt/libvirt/guest.py

12. Nova current source — `nova/virt/libvirt/config.py`  
    https://github.com/openstack/nova/blob/master/nova/virt/libvirt/config.py

13. Nova current source — `nova/virt/libvirt/driver.py`  
    https://github.com/openstack/nova/blob/master/nova/virt/libvirt/driver.py

14. Nova current source — Compute Driver interface  
    https://github.com/openstack/nova/blob/master/nova/virt/driver.py

15. OpenStack 2025.1 Epoxy releases  
    https://releases.openstack.org/epoxy/index.html

16. OpenStack 2025.2 Flamingo releases  
    https://releases.openstack.org/flamingo/index.html

17. OpenStack 2026.1 Gazpacho releases  
    https://releases.openstack.org/gazpacho/index.html

18. OpenStack 2026.2 Hibiscus releases  
    https://releases.openstack.org/hibiscus/index.html

---

## Примечание

Раздел о доработке является **проектным предложением**, а не описанием уже существующей upstream-функции Nova.

Наиболее важное архитектурное различие:

```text
libvirt уже умеет full-system snapshot
                |
                v
Nova не предоставляет его как
полноценный Instance Checkpoint API
                |
                v
доработка в первую очередь должна
добавить orchestration + lifecycle
```

А для Cinder-backed VM:

```text
Nova checkpoint
       |
       +--- libvirt RAM/runtime
       |
       +--- Cinder volume snapshots
       |
       +--- Nova state/config
       |
       +--- единая state machine / rollback
```

Именно этот orchestration layer в штатном OpenStack на данный момент отсутствует.
