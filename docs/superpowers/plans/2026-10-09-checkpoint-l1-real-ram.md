# Checkpoint L1-R Real RAM Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Подключить настоящую RAM diskless Linux-гостя под QEMU/TCG к локальной цепочке Nova/Cinder и проверить её recovery в 13 сценариях без стенда.

**Architecture:** Существующий L1 получает отдельный явно выбранный режим с настоящим libvirt adapter и test-only topology. Штатные Nova executor, Runtime, receipts, staging/store и Cinder HTTP сохраняются; модель хранит только тома/attachments. Supervisor наблюдает RAM через serial независимо от продукта, verifier сверяет raw evidence офлайн.

**Tech Stack:** Python 3.11, unittest, SQLite/WAL, существующие закреплённые Nova/Cinder environments, libvirt-python/native client 11.10.0, libvirt daemon 9.0.0, QEMU 7.2.22 TCG/aarch64, static C initramfs, два disposable Docker containers с общим PID namespace и Linux volume.

**Spec:** [Согласованная спецификация L1-R](../specs/2026-10-09-checkpoint-l1-real-ram-design.md).

Статус: **план подготовлен для review; исполнение не начато**.
Способ исполнения — Native, ранее выбранный пользователем в этой сессии.
Задачи последовательно меняют общие manifest, runtime и evidence contracts,
поэтому реализация в одном контексте уменьшит потери на передаче решений.
После реализации нужен свежий независимый review. Перед началом остаётся
review этого нового плана пользователем.

## Global Constraints

- Исходный L1: `02261dd0d4a28f525535a95cc1e7f68a1d34e906`; отдельная ветка `feature/checkpoint-l1-real-ram`, новый worktree `work/checkpoint-local-integration-real-ram`.
- Nova: `6d55009fae5eb55adb156e13835dd66240488f84`; Cinder: `d065a55cbe5cb8cd59489752846a41145d72bd27`. Их код, pins и production profiles не меняются.
- Python **3.11** внутри Linux; существующий installer L1 и его libvirt/native dependency pins не ослабляются.
- Scope `local_real_ram_simulated_storage`; `ram_runtime=libvirt_qemu_tcg`, `guest_arch=aarch64`, `kvm=false`, `storage_runtime=persistent_test_model`, `topology_admission=test_fixture`.
- Всегда `production_topology_validated=false`, `disk_ram_consistency_validated=false`, `hardware_qualified=false`, `production_qualified=false`.
- Гость: **256 МиБ RAM, 1 vCPU**, persistent domain, без дисков и сетевого интерфейса; проверочный payload **8 МиБ**.
- RAM artifact limit **384 МиБ**, case-local staging/store profile budget **1536 МиБ**, allocation unit **4096 байт**, metadata budget **256 КиБ**.
- Ровно **13** L1-R cases; исходные **24** L1 IDs и model mode по умолчанию сохраняются. Старые отчёты не переименовываются в real-RAM evidence.
- Только принадлежащий запуску UNIX socket libvirt; runtime контейнер с `--init`; общий PID namespace и volume, согласованные UID/GID; без privileged/KVM/физической СХД.
- Мутации guest lifecycle разрешены setup/workflow/cleanup с раздельным audit; recovery не делает bootstrap, повторный native execute или запись RAM.
- Query меняет только audit чтения. Marker, PID, SHUTOFF и save image без штатной receipt не дают completion.
- SQLite, staging/store и binaries находятся на Linux volume; macOS bind mount используется только для read-only inputs или выгрузки после остановки writers.
- Runtime кейсы не skip-аются при неподготовленной среде; fail/incomplete сохраняет diagnostics. Старые результаты не подменяют текущий прогон.
- Нет merge/push/deployment. Дефект продукта фиксируется отдельным RED evidence; harness не обходит его для получения PASS.

## Review Focus

1. UNIX socket или одноимённый domain принадлежат другому запуску: отвергнуть до mutation; Task 2 `test_foreign_socket_or_domain_is_rejected_before_mutation`.
2. После resume пропали domain/config metadata, но остался save image: не создавать guest и не объявлять completion; Tasks 2/3 `test_resume_missing_domain_does_not_bootstrap`, `test_save_file_without_receipt_stays_unknown`.
3. Console выдаёт куски строк, старый challenge, дублирующийся ответ или молчит в PAUSED: не принимать старое состояние и не unpause-ить ради проверки; Task 1 `test_console_requires_one_fresh_complete_response`.
4. Model DB показывает SHUTOFF, а настоящий guest RUNNING либо libvirt недоступен: host connect/disconnect отклоняются; Task 4 `test_host_io_requires_current_native_shutoff`.
5. Прерван cleanup или отчёт/образ переписан с новыми общими hashes: неполный run и несогласованные raw доказательства не PASS; Tasks 2/7 `test_cleanup_timeout_is_incomplete`, `test_rehashed_ram_or_fault_evidence_is_rejected`.

---

## Подготовка исполнения и соглашения

До первого изменения применить `using-git-worktrees`, проверить отсутствие
чужой работы и создать указанный новый worktree от точного L1 commit.
Имеющиеся Nova/Cinder/L1 worktrees не переключать и не очищать. Если целевой
путь/ветка уже существуют, проверить их происхождение; не перезаписывать.
Корень workspace вне Git; его не инициализировать.

Пути далее относительно нового worktree. Согласованные входы копируются в
`docs/real-ram-design.md` и `docs/real-ram-plan.md` вместе с исходными SHA256;
существующие `docs/design.md` и `docs/plan.md` описывают прежний L1.
Evidence задачи: `.superpowers/l1r/task-N/{red.log,green.log,commands.json}`
и commit SHA. Исходники/проверяемые fixtures в Git, большие runtime outputs
и локальные environment receipts вне Git. Ошибка подготовки зависимости
не считается содержательным RED.

В командах Linux `L1_ENVIRONMENT` задаётся существующим installer;
`L1R_ENVIRONMENT` указывает новый проверенный `runtime-environment.json`.
Они читаются явно, без fallback на production libvirt. В host CLI используется
`python3` со стандартной библиотекой; все продуктовые тесты — `python3.11`.
Обычные case/startup deadlines остаются 160/60 секунд; gate имеет общий предел
120 секунд, console request — 10 секунд, cleanup собственных ресурсов —
30 секунд. Все overrides — положительные конечные числа; timeout не PASS.

Для Runtime операции применяется внешний deadline существующего supervisor:
зависший вызов прекращается вместе с принадлежащим процессом, его результат
остаётся unknown. После таймаута cleanup сначала доказывает отсутствие
native job/writer или завершает принадлежащий QEMU, и лишь затем освобождает
ресурсы среды. Это аварийная уборка harness, не completion операции Nova.

## Карта файлов

| Область | Файлы и ответственность |
|---|---|
| Среда и ранний gate | `tools/real_ram_env.py`, `tools/real_ram_gate.py`, `environments/real_ram/{client.Dockerfile,runtime.Dockerfile,entrypoint.py}` — owned containers/volume, namespaces, версии, direct libvirt gate |
| Guest и console | `checkpoint_lab/real_ram/{__init__,boot,console}.py`, `guest/real_ram/init.c` — reproducible initramfs и bounded fresh-challenge protocol |
| Manifest и владение | `checkpoint_lab/real_ram/{contracts,manifest,domain}.py` — типы, schema 2, persistent domain lifecycle; dispatch в существующих `manifest.py`, `__main__.py` |
| Native boundary | `checkpoint_lab/real_ram/{native,topology,audit}.py` — настоящий Guest, native calls, test topology и раздельный audit |
| Интеграция | `checkpoint_lab/real_ram/{session,observe,oracle}.py`; узкие изменения `nova/{adapters,bootstrap,entry}.py`, `model/{state,server,client}.py`, `session.py` |
| Сценарии | `checkpoint_lab/real_ram/{cases,scenarios,runner}.py`; точечное расширение `proxy.py` для доказанной потери ответа шага Cinder |
| Офлайн evidence | `checkpoint_lab/real_ram/{evidence,evidence_native,evidence_sql}.py`; dispatch в существующем `evidence.py`; общий SHA/SQL helpers переиспользуются без ослабления legacy checks |
| Тесты | `tests/real_ram/{__init__,fixtures,test_environment,test_console,test_manifest,test_domain,test_native,test_topology,test_model_boundary,test_lifecycle,test_faults,test_matrix,test_evidence}.py` |
| Поставка | `tools/package.py`, `README.md`, `VALIDATION.md`, `REVIEW.md`, `REVIEW-RESOLUTION.md`, `docs/real-ram-*.md` |

### Task 1: Изолированная среда, guest probe и direct libvirt gate

**Files:** Create environment/tools/guest/boot/console files из карты, `tests/real_ram/{__init__,fixtures,test_environment,test_console}.py`; modify `.gitignore` для локальных результатов и `README.md` для gate.

**Interfaces:**
- `boot.build(kernel: Path, init_source: Path, output: Path) -> dict`: kernel/initramfs paths, SHA256 и compiler invocation; output новый.
- `console.GuestProbe(socket_path: Path, *, timeout: float=10)` с `set_point(marker: str, counter: int) -> dict`, `read_point(challenge: str) -> dict`, `step(challenge: str) -> dict`, `close() -> None`.
- `tools.real_ram_env.prepare(root: Path, nova_checkout: Path, cinder_checkout: Path, lab_checkout: Path) -> Path`: возвращает candidate JSON с owned container IDs/volume, socket, UID/GID, images, boot inputs и namespaces. `cleanup(candidate: Path) -> dict` удаляет только подтверждённо owned resources.
- `tools.real_ram_gate.run(candidate: Path, output: Path) -> dict`: создаёт `gate.json` и `runtime-environment.json`; итог `direct_libvirt_gate=passed`, `native_runtime_gate=pending` до Task 3.
- CLI: `python3 -m tools.real_ram_env prepare --root <new-host-output> --nova-checkout <path> --cinder-checkout <path> --lab-checkout <path>`; `cleanup --environment <candidate.json>`.

- [ ] **Step 1: Создать рабочую изоляцию и сохранить входы.** Проверить базовый SHA и чистоту; copy spec/plan с hashes; для контейнеров подготовить автономные Git clones exact pins без alternates, как требует существующий installer. Старый остановленный L1 container не изменять.
- [ ] **Step 2: Добавить RED environment/console tests.** В `test_environment` проверить: неправильный PID namespace, UID, native client version, небезопасный socket и существующий output отклонены; foreign container не остановлен cleanup. `test_console` использует socketpair и fragmented synthetic peer, не QEMU, с независимыми wire fixtures:

```python
def test_console_requires_one_fresh_complete_response(self):
    self.assertEqual(POINT_A, self.read_fragmented_reply(CHALLENGE_A))
    self.assertRejected(stale_challenge=CHALLENGE_B)
    self.assertRejected(duplicate_complete_reply=True)
    self.assertRejected(eof_mid_line=True)
    self.assertRejected(no_reply=True)
    self.assertEqual([], self.native_resume_calls)
```

Fixtures определяются в `tests/real_ram/fixtures.py`: 64-hex marker, 32-hex challenge,
counter, `bytes=8388608`, checksum, `valid=1`. Reader допускает только одну
полную строку нужного challenge, ограничивает вход 64 КиБ, отвергает malformed
поля и ошибку byte-check; digest payload вычисляется host-стороной независимо.
- [ ] **Step 3: Запустить RED.** `python3.11 -m unittest tests.real_ram.test_environment tests.real_ram.test_console -v`; ожидается отсутствие нового API/нарушение сформулированных assertions, не ошибка OS dependency.
- [ ] **Step 4: Реализовать подготовку и probe.** Runtime image основывается на проверенном `nova-ram-spike:20261009` с проверкой ID `sha256:b401f226777150df23ca3fab57f21a262316df0f4618e2a4c5355f264bbf2d25`; фиксируются client/daemon версии отдельно. Client image — отдельный производный Rocky образ, installer L1 неизменён; exact libvirt 11.10.0 проверяется до установки venv. Общий Linux volume смонтирован одинаково, runtime запускается первым с `--init`, client — с `--pid=container:<owned-runtime-id>`. Оба без privileged и внешней сети во время run; создание образов/зависимостей — отдельная подготовка. Docker вызывается argv-массивами с timeout, credentials/env хоста не копируются.
- [ ] **Step 5: Реализовать guest wire protocol и direct gate.** Static C PID 1 использует только RAM и serial; marker не включён в boot inputs. В отличие от spike, повторный SET точки A/B допустим во время RUNNING; он имеет fresh request ID и не производится recovery. Gate из client interpreter через owned UNIX socket выполняет persistent define/start, metadata LIVE/CONFIG, save через открытый directory FD и `/proc/<client-pid>/fd/.../data`, SHUTOFF/XML, restore PAUSED, state/XML, resume, guest proof и cold control. Raw libvirt results, PID/start, XML и boot hashes сохраняются. Вывод `direct_libvirt_gate=passed` разрешён только после bounded cleanup.
- [ ] **Step 6: Проверить GREEN и настоящий gate.** Повторить unittest. В свежей среде: `python3.11 -m tools.real_ram_gate --environment <candidate.json> --output /l1r/gate-1`; exit 0, все direct checks true, native Runtime пока pending. Несовместимость client/daemon или pinned path останавливает дальнейшую реализацию: сохранить failure и пересмотреть размещение среды согласно spec §4, не менять pins или путь записи.
- [ ] **Step 7: Commit.** `git add tools/real_ram_env.py tools/real_ram_gate.py environments/real_ram guest checkpoint_lab/real_ram tests/real_ram .gitignore README.md docs/real-ram-*`; `git commit -m "test: add isolated libvirt RAM environment gate"`.

### Task 2: Закрытый режим и владение persistent domain

**Files:** Create `real_ram/{contracts,manifest,domain,cases}.py`, `tests/real_ram/{test_manifest,test_domain}.py`; modify `checkpoint_lab/manifest.py`, `checkpoint_lab/__main__.py`.

**Interfaces:**
- `RuntimeEnvironment.from_file(path: Path) -> RuntimeEnvironment`: frozen record закрытого schema 1; socket/owner token, UID/GID, namespace identity, boot inputs/hashes, image/version inventory, direct gate proof. `to_dict() -> dict`.
- `RealRamManifest`: immutable schema 2, те же `run_id, case_id, root, nova, cinder, ids`, что RunManifest, плюс `runtime: RuntimeEnvironment`, qualification fields из Global Constraints. `from_dict(value: dict) -> RealRamManifest`, `to_dict() -> dict`.
- `real_ram.manifest.create(root: Path, case_id: str, runtime: RuntimeEnvironment) -> RealRamManifest`; общий `read_manifest(path)` dispatch по schema 1/2. Legacy `RunManifest.from_dict` по-прежнему отклоняет schema 2.
- `OwnedDomain(manifest: RealRamManifest)`: `bootstrap() -> dict`, `reopen() -> libvirt.virDomain`, `observe() -> dict`, `cleanup(*, deadline: float) -> dict`. `observe` возвращает UUID/state/persistent/job/XML/PID identity, без RAM и completion status.
- `real_ram.cases.all_cases() -> tuple[Case, ...]`, `required_checks(case_id: str) -> frozenset[str]`: точные 13 IDs из spec §5.

- [ ] **Step 1: Написать RED contract/ownership tests.** Schema 2 не принимается legacy-reader; closed fields, точные flags/types, UUID, immutable boot inputs, case list проверяются. Изменение path через symlink, чужой socket/domain и несовпадение owner token отклоняются до libvirt mutation.

```python
def test_resume_missing_domain_does_not_bootstrap(self):
    self.delete_owned_domain_outside_adapter()
    with self.assertRaises(LabError):
        self.owner.reopen()
    self.assertEqual([], self.recorded_calls("defineXML", "create", "restoreFlags"))

def test_cleanup_timeout_is_incomplete(self):
    result = self.cleanup_with_unresponsive_owned_domain()
    self.assertFalse(result["complete"])
    self.assertEqual([], self.foreign_resource_mutations)
    self.assertTrue(self.evidence_directory.exists())
```

- [ ] **Step 2: Запустить RED.** `python3.11 -m unittest tests.real_ram.test_manifest tests.real_ram.test_domain -v`.
- [ ] **Step 3: Реализовать mode/ownership.** CLI получает `--runtime {model,libvirt-tcg}` (default model) и `--runtime-config` для preflight/run; отсутствие config в real mode — ошибка, наличие real config в model mode — ошибка. `--case` валидируется после выбора inventory; unknown/duplicate IDs отклоняются. Owned domain name derived из run UUID, UUID=instance UUID; metadata ownership использует Nova namespace, совместимый с штатным checkpoint marker. Bootstrap только один раз, resume только lookup + проверки. Cleanup фиксирует окончания QEMU и daemon-owned resources; при неполном cleanup PASS невозможен.
- [ ] **Step 4: Проверить GREEN и совместимость.** Повторить новую команду и `python3.11 -m unittest tests.unit.test_manifest tests.unit.test_cli tests.unit.test_case_inventory -v`. Default CLI и schema 1 не изменены; два разных real runs не могут управлять одним domain/socket.
- [ ] **Step 5: Commit.** `git add checkpoint_lab/real_ram checkpoint_lab/manifest.py checkpoint_lab/__main__.py tests/real_ram`; `git commit -m "feat: define isolated real RAM manifest and domain ownership"`.

### Task 3: Настоящий Guest, test topology и gate штатного Runtime

**Files:** Create `real_ram/{native,topology,audit}.py`, `tests/real_ram/{test_native,test_topology}.py`; modify `tools/real_ram_gate.py`.

**Interfaces:**
- `NativeAudit(manifest)`: `append(kind: str, key: EffectKey, fields: dict) -> None`, fsync до отправки mutation; записи `dispatch/return/error/observe`, последовательность и process identity. Setup/cleanup — отдельный scope audit.
- `NativeAudit.dispatches_for(key: EffectKey) -> tuple[dict, ...]`: read-only выборка по child; не дедупликация. После process restart sequence продолжается из сохранённого журнала, неполная последняя запись не принимается как завершённый native return.
- `NativeHost(manifest, owner: OwnedDomain, audit: NativeAudit, faults=None)`: `get_guest(instance)`, `checkpoint_saved_xml(path: str) -> str`, `checkpoint_restore(path: str, xml: str) -> None`. `get_guest` возвращает wrapper над настоящим `nova.virt.libvirt.guest.Guest` с `uuid`, `get_xml_desc`, `checkpoint_state`, `checkpoint_block_job`, `checkpoint_set_marker`, `checkpoint_save`, `checkpoint_stop`, `resume`; методы сохраняют существующие сигнатуры ModelRuntime/Guest.
- `FixtureTopology(host: NativeHost, manifest)`: `inspect(context, instance, db_manifest: dict, purpose: str, saved_binding=None) -> dict`, `current_xml(context, instance, manifest: dict, binding) -> str`, `restore_xml(saved_xml: str, current: dict, binding) -> str`.
- Native gate report после Task 3: `native_runtime_gate=passed`, с source pins, точными лимитами и evidence настоящих Runtime.save/restore_paused/unpause/query.

- [ ] **Step 1: Написать RED adapter/topology tests.** Проверить одно выполнение native call, его exception без retry, актуальный lookup после restore, metadata/UUID binding, paused restore flags. При forged XML identity или изменённых kernel/initramfs inspect/restore отклоняются. Production `Topology` отвергает реальный TCG XML; FixtureTopology возвращает fingerprint из фактической identity, а не постоянную строку.

```python
def test_save_file_without_receipt_stays_unknown(self):
    self.save_with_native_success_then_drop_before_receipt()
    before = self.audit.dispatches_for(self.save_child)
    result = self.production_runtime.query(self.ctx, self.instance, self.save_child)
    self.assertEqual("unknown", result.status)
    self.assertEqual(before, self.audit.dispatches_for(self.save_child))
    self.assertTrue(self.guard_is_owned())
```

Этот тест использует настоящие journal/staging fixtures, не mock `Runtime.query`.
Отдельно receipt success + неверный marker/current XML не признаётся completion.
- [ ] **Step 2: Запустить RED.** В Nova interpreter: `python -m unittest tests.real_ram.test_native tests.real_ram.test_topology -v`; PYTHONPATH содержит точный Nova checkout и lab, private SDK sentinel установлен fixtures.
- [ ] **Step 3: Реализовать adapter и FixtureTopology.** Save вызывает Guest.saveFlags-путь без сериализации RAM в JSON и без ModelState.execute. Restore читает фактический saved XML и использует PAUSED. Wrapper добавляет только audit/fault hooks, не outcome/dedup. Inspect проверяет owned persistent ARM/TCG domain, state/job/managed save, memory/vCPU/boot hashes и отсутствие guest disks/NICs; строит runtime manifest с реальным `xml_sha256`, `memory_bytes=268435456`, `ports=[]`, пустыми config/key fingerprints и отдельным test-profile ID/hash. BDM/VIF остаются в DB fingerprint и не выдаются за guest devices. Для XML используется штатный `guest_identity/with_marker/restore_xml`; production admission не вызывается как положительное разрешение.
- [ ] **Step 4: Завершить gate spec §4.5.** Добавить реальный Runtime цикл с настоящими SQL child intents/receipts и StagingStore из exact Nova pin, новым FixtureTopology и лимитами 384/1536 МиБ. Тестовый fixture в `tests/real_ram/fixtures.py` переиспользует миграции/bootstrap L1 и шаблон сборки `nova.tests.fixtures.checkpoints_v2.CheckpointV2TestCase.make_driver`; synthetic Backend RAM callbacks заменены NativeHost, Runtime/journal не подменены. Это ограниченный Runtime gate, не Cinder HTTP acceptance. Использовать pinned writer/reader пути без промежуточного копирования. Запустить `python3.11 -m tools.real_ram_gate --environment <candidate.json> --output /l1r/gate-2 --with-nova-runtime`; exit 0, direct и native gates passed, RAM proof есть. При product failure сохранить RED и не обходить проверку.
- [ ] **Step 5: Проверить GREEN и commit.** Повторить unit/integration command; `git diff --check`; `git add checkpoint_lab/real_ram tools/real_ram_gate.py tests/real_ram`; `git commit -m "feat: connect native libvirt RAM to checkpoint Runtime"`.

### Task 4: Сквозной lifecycle и раздельные источники наблюдения

**Files:** Create `real_ram/{session,observe,oracle,scenarios,runner}.py`, `tests/real_ram/{test_model_boundary,test_lifecycle}.py`; modify `nova/{adapters,bootstrap,entry}.py`, `model/{state,server,client}.py`, `session.py`, `__main__.py`.

**Interfaces:**
- `build_driver(..., *, runtime_config=None)` выбирает NativeHost/FixtureTopology только по проверенному real manifest; default прежний.
- `ModelState.open(path, *, create: bool, native_observer=None)`; observer `() -> dict` даёт актуальные UUID/state/XML настоящего domain, только read-only. `guest_write_disks(disks: dict[str, bytes]) -> None` и соответствующий ModelClient/server route доступны только real mode при фактическом RUNNING.
- `RealRamSession(FaultNovaCase)`: `guest_write(marker: str) -> None`, `observe_point() -> dict`, `close() -> None`; ожидаемые случайные A/B points создаются supervisor до мутаций и сохраняются отдельно от runtime recovery.
- `real_ram.observe.observe_case(manifest, endpoints: dict, owner: OwnedDomain, probe: GuestProbe) -> dict`: Nova/Cinder SQL, `storage_model`, `ram_guest` и `native_domain`; для PAUSED/SHUTOFF ram_guest содержит `available=false` с причиной, не ожидаемую метку.
- `real_ram.oracle.assert_point(observed: dict, expected: dict) -> None`; `real_ram.runner.run_suite(options: dict) -> dict` повторно использует bounded supervisor и общий environment inventory.

- [ ] **Step 1: Написать RED модели и lifecycle.** New mode отвергает synthetic save/stop/restore/unpause и запись model RAM. Старые model mode методы сохраняют поведение. Для host I/O read-only native observer обязателен и срабатывает до изменения host_connections:

```python
def test_host_io_requires_current_native_shutoff(self):
    self.set_legacy_model_state("SHUTOFF")
    for native_state in ("RUNNING", "PAUSED", "UNAVAILABLE"):
        with self.subTest(native_state=native_state):
            self.assertHostMutationRejected(native_state)
            self.assertEqual(self.before_connections, self.read_connections())
```

Lifecycle test вызывает настоящие Nova API create A/B, revert A/B/A и delete B/A.
Для каждого RUNNING endpoint сравнивает fresh guest response и два независимых
model volume markers; проверяет instance/volume/port UUID, artifact bytes,
guards и освобождение quotas. Ни expected RAM, ни model RAM не передаются recovery.
- [ ] **Step 2: Запустить RED.** `python3.11 -m unittest tests.real_ram.test_model_boundary tests.real_ram.test_lifecycle -v` в подготовленном client container с обоими environment paths.
- [ ] **Step 3: Реализовать wire-up и model policy.** Bootstrap real flavor=256/1, limits согласованы в admission/staging/store/quota observers. Model meta фиксирует runtime mode; real mode domain row хранит пустую RAM и явное внешнее состояние, не синтетический KVM XML. `domain()` для real execution читает native observer; `ModelState.observe_storage() -> dict` читает только SQL и не требует libvirt, чтобы offline verifier работал без среды. Real-mode session запускает model server через проверенный `environment['nova_python']` с binding и owned read-only connection; default model interpreter прежний. Manifest/evidence modules не импортируют libvirt при offline чтении. Native effects запрещены до model mutation; host_connect/disconnect продолжают моделировать I/O, проверяя реальный SHUTOFF. Live libvirt data не записывается как completion receipt модели.
- [ ] **Step 4: Реализовать session/наблюдение и L1R-01.** Case preflight требует оба gate=passed из Tasks 1/3 и сверяет их pins/среду; pending direct-only receipt не допускает интеграционный run. Bootstrap domain/probe до группы Nova; restart группы переоткрывает существующий domain. A/B SET допустим только supervisor; pause/stop observation не вызывает PROBE/resume. Вся checkpoint RAM проходит штатный store; evidence archive сохраняет копии уже зафиксированных образов до штатного delete продукта, не влияет на workflow.
- [ ] **Step 5: Проверить GREEN.** Повторить tests; CLI `python3.11 -m checkpoint_lab run --runtime libvirt-tcg --runtime-config "$L1R_ENVIRONMENT" --nova-checkout <nova> --cinder-checkout <cinder> --nova-python <nova-python> --cinder-python <cinder-python> --output /l1r/results/lifecycle-1 --case L1R-01` — case pass, `suite_complete=false`. До Task 7 отчёт явно development/incomplete verification, не финальная приёмка.
- [ ] **Step 6: Commit.** `git add checkpoint_lab tests/real_ram`; `git commit -m "feat: run real RAM lifecycle through local Nova and Cinder"`.

### Task 5: Потеря процесса до и после durable receipt

**Files:** Modify `real_ram/{native,audit,scenarios,cases,oracle}.py`; create `tests/real_ram/test_faults.py`; reuse `faults.py` и process supervisor без изменения их semantics.

**Interfaces:**
- `RealRamSession.crash_runtime(action: str, *, after_receipt: bool) -> dict`: action только save/restore_paused/unpause; возвращает bound ACK, before/after SQL/native observations и ordered process transitions.
- Consumes `NativeAudit.dispatches_for` из Task 3; native audit не подменяет штатную receipt.

- [ ] **Step 1: Написать RED для шести случаев.** Для каждой action проверить два окна: native return до receipt — unknown/recovery_required, guard/quota сохранены, dispatch count=1; receipt до observation — recovery завершается без повторения faulted key. UUID и launch IDs до/после различаются корректно, остальные child actions допускаются только по штатному workflow. Добавить missing ACK и foreign child/resource negative controls.
- [ ] **Step 2: Запустить RED.** `python3.11 -m unittest tests.real_ram.test_faults -v`; реальные process tests не mock-ают exit/restart и не skip-аются.
- [ ] **Step 3: Реализовать точные barriers.** Первый hook после успешного native call и проверяемого return, до возврата wrapper штатному Runtime; второй — после настоящего RuntimeRecord.finish, до observation. Fsync dispatch/return event предшествует ACK. Supervisor ждёт ACK, SIGKILL/exit, запускает новый interpreter на тех же БД, затем повторяет recovery минимум трижды для unknown cases. No bootstrap/SET/native replay между ними. Актуальные состояния unknown: save→SHUTOFF, restore_paused→PAUSED, unpause→RUNNING.
- [ ] **Step 4: Проверить GREEN.** Все шесть tests проходят; сравнить native audit до/после repeated query и сверить raw journal/guards/quotas, а не только report flags. Для PAUSED/SHUTOFF нет подставленного RAM proof; для RUNNING — fresh console request.
- [ ] **Step 5: Commit.** `git add checkpoint_lab/real_ram tests/real_ram/test_faults.py`; `git commit -m "test: verify real RAM recovery around durable receipts"`.

### Task 6: Потерянные Cinder ответы, partial revert, writer и cold control

**Files:** Modify `real_ram/{scenarios,cases,oracle}.py`, `checkpoint_lab/proxy.py`; create `tests/real_ram/test_matrix.py`; modify `tests/unit/test_proxy.py` при расширении общего proxy.

**Interfaces:**
- `RealRamSession.lost_cinder_reply(action: str) -> dict`, actions snapshot/revert/delete_snapshot.
- `RealRamSession.partial_revert() -> dict`, `live_writer() -> dict`, `cold_boot_control() -> dict`.
- Existing `FaultProxy.drop_once` расширяется keyword `result_action` только при необходимости: response binding к ожидаемому operation/step/resource записывается в trace; никакого изменения/повтора upstream request.

- [ ] **Step 1: Написать RED оставшихся шести cases.** L1R-04: proxy увидел HTTP 200/202 для выбранного Cinder operation/step, закрыл downstream, recovery не повторяет mutation. L1R-05: один volume revert выполнен, другой unknown; настоящий guest SHUTOFF, нет restore/unpause и host reconnect. L1R-06: cleanup не удаляет workspace при подтверждённом живом writer; после завершения/остановки владельца работает штатная проверка. L1R-07: только после успешного lifecycle/освобождения guards cold boot даёт unset/0 и не прежний payload.
- [ ] **Step 2: Запустить RED.** `python3.11 -m unittest tests.real_ram.test_matrix tests.unit.test_proxy -v`. Проверка inventory требует точного ordered списка 13 IDs, все обязательные case checks перечислены до выполнения.
- [ ] **Step 3: Реализовать сценарии.** Response-loss ставится на доказуемую HTTP границу Cinder; это не обещание потери libvirt wire reply. Phase/action/resource match нельзя выводить из случайного sleep. Live-writer hook соответствует настоящему открытому staging writer после native save. Cold boot — отдельная owned setup/control action вне recovery, с новым PID/start identity и без SET старой метки. Сохранять все transition observations до cleanup.
- [ ] **Step 4: Проверить GREEN и CLI inventory.** Повторить tests. Селектор с L1 ID при real mode, L1R ID при model mode и дублирующим ID даёт input error до bootstrap; subset никогда не `suite_complete=true`.
- [ ] **Step 5: Commit.** `git add checkpoint_lab/real_ram checkpoint_lab/proxy.py tests/real_ram/test_matrix.py tests/unit/test_proxy.py`; `git commit -m "test: complete the real RAM fault matrix"`.

### Task 7: Проверяемый офлайн отчёт и corruption controls

**Files:** Create `real_ram/{evidence,evidence_native,evidence_sql}.py`, `tests/real_ram/test_evidence.py`; modify `checkpoint_lab/evidence.py`, `checkpoint_lab/evidence_sql.py` только для выделения общих read-only helpers, `real_ram/runner.py` и `runner.py` source inventory.

**Interfaces:**
- `real_ram.evidence.write_report(root: Path, selected_ids: tuple[str,...], outcomes: list[dict], environment: dict) -> dict`; schema 2 + exact L1-R scope/flags.
- `real_ram.evidence.verify_report(root: Path) -> dict`; без Docker/libvirt/Nova service startup, без native mutations.
- `evidence_native.validate_native(directory: Path, manifest: RealRamManifest, case: Case) -> None` и `evidence_sql.validate_real_ram_databases(directory: Path, manifest: RealRamManifest, after: dict, barriers: tuple=()) -> None`.
- Общий CLI `verify-report` dispatch только по exact schema/scope; неизвестная комбинация отклоняется, legacy validator не ослабляется.

- [ ] **Step 1: Написать RED raw-evidence tests.** Взять реальные сохранённые development cases, создать отдельные копии, изменить по одному семантическому факту и пересчитать общий SHA manifest. Проверки:

```python
def test_rehashed_ram_or_fault_evidence_is_rejected(self):
    for edit in ("scope", "kvm", "guest_checksum", "challenge", "uuid",
                 "ack_order", "missing_dispatch", "duplicate_mutation",
                 "ram_artifact", "cleanup_missing", "sql_json_disagreement"):
        with self.subTest(edit=edit):
            copied = self.copy_and_corrupt(edit)
            write_hashes(copied)
            with self.assertRaises((LabError, AssertionError)):
                verify_report(copied)
```

Дополнительно: truncated JSONL, duplicate sequence, omitted case и существующий
foreign trace под правильным именем, завышенный size, missing WAL. Подмена
artifact проверяется относительно штатных SQL/staging metadata/checksums,
а не только newly-rehashed SHA256SUMS. Полностью согласованная фальсификация
всех источников не объявляется решённой криптографической аттестацией.
- [ ] **Step 2: Запустить RED.** `python3.11 -m unittest tests.real_ram.test_evidence -v`; контрольная неизменённая copied evidence должна проходить, иначе отрицательные tests неинформативны.
- [ ] **Step 3: Реализовать semantic verification.** Проверять actual XML/boot hashes, source/client/daemon inventories, PID/start и native-call order, unique bound challenges/guest payload, expected vs observed A/B, dispatch counts по child, receipts/unknown guard/quotas, proxy upstream/drop и restart ordering. Raw SQL/WAL читать на приватных копиях; storage-only модель сравнивается с `storage_model`, RAM берётся только из serial evidence. Manifest schema 2 читается без требования существования старых execution paths. Boot C source/compiler/hashes включить в lab source inventory, не ограничиваться Python файлами. Бинарные файлы хэшировать потоково; JSON budgets не применяются к RAM blobs.
- [ ] **Step 4: Проверить GREEN и portability.** Выполнить все corruption controls; скопировать отчёт в другой каталог и `python3.11 -m checkpoint_lab verify-report --output <copied-report>` при недоступном runtime socket и отсутствии Docker. Перед/после сравнить hashes всех файлов; verifier ничего в evidence не изменил.
- [ ] **Step 5: Commit.** `git add checkpoint_lab tests/real_ram/test_evidence.py`; `git commit -m "feat: verify real RAM evidence independently of runtime"`.

### Task 8: Полная приёмка, review и воспроизводимая поставка

**Files:** Modify `tools/package.py`, `README.md`, `VALIDATION.md`, `REVIEW.md`, `REVIEW-RESOLUTION.md`, `docs/real-ram-*.md`; tests `tests/unit/test_package.py` и metadata checks при необходимости.

**Interfaces:** Существующий `tools.package.create_package` сохраняет clean-commit/source pin правила; real-RAM evidence проходит новый verifier, legacy evidence — старый. Поставка `deliverables/checkpoint-local-integration-real-ram` создаётся в новом пути.

- [ ] **Step 1: Проверить все изменённые контракты.** `L1_ENVIRONMENT=<file> L1R_ENVIRONMENT=<file> python3.11 -m unittest discover -s tests -v`; `python3.11 -m compileall -q checkpoint_lab tests tools`; `git diff --check`. Отсутствие real environment не превращает новые integration tests в skip. Если package behavior меняется, сначала его RED/положительный unit control, затем GREEN.
- [ ] **Step 2: Запустить полный L1-R.** CLI из Task 4 без `--case`, output `/l1r/results/acceptance-1`; ровно 13 passed IDs, exit 0, `suite_complete=true`, exact scope/flags, complete cleanup. Проверить report отдельно. После новых изменений/найденных дефектов повторяются затронутые проверки и полный финальный acceptance; без новых оснований не множить одинаковые прогоны.
- [ ] **Step 3: Пройти legacy regression.** Полный обычный CLI `run --runtime model` на новом output даёт исходные 24/24 и прежний scope; проверить legacy report. Прежний L1 package также должен проходить legacy offline verification. Новые tests не заменяют эту матрицу, старые исторические results не считаются свежими.
- [ ] **Step 4: Зафиксировать результат и запросить финальный независимый review.** В Native использовать `requesting-code-review` с полной diff и evidence, а не только последним commit; reviewer проверяет plan/spec и false-success риски. Устранить существенные замечания с RED/GREEN; необязательные/отложенные замечания явно перечислить. Нет изменения Nova/Cinder pins без отдельного согласования.
- [ ] **Step 5: Обновить документацию по факту.** README даёт команды подготовки/cleanup и offline verify, VALIDATION — actual versions, image IDs, counts, ограничения, preparation failures и qualification flags. Review/resolution ссылаются на конкретные изменения и свежие результаты. Проверить чистоту и commit всех deliverable sources: `git add README.md VALIDATION.md REVIEW.md REVIEW-RESOLUTION.md tools/package.py tests/unit/test_package.py docs`; `git commit -m "docs: record real RAM integration acceptance"`.
- [ ] **Step 6: Упаковать и проверить из распакованной копии.** Существующий `python3.11 -m tools.package --repo . --output <new-package-dir> --nova-checkout <nova> --evidence l1r=<acceptance-1> --evidence legacy=<legacy-run> --evidence development=<task-evidence>`; проверить package/source hashes и совпадение commit/tree. В fresh unpacked source запустить offline verification полного L1-R report и smoke `L1R-01`, `L1R-02.save`, `L1R-03.restore_paused`; эти subset smoke не обозначаются 13/13. Проверить trace фактического unpacked import root.
- [ ] **Step 7: Завершить owned cleanup и handoff.** Сохранить evidence до удаления containers/volume, проверить absence owned domains/QEMU и outcome cleanup, clean worktree, unchanged product HEAD/tree. Отдать пакет, source commit, 13/13 и 24/24 результаты с ограничениями. Не выполнять merge/push/deploy.

## Проверка покрытия плана

| Требование спецификации | Задача |
|---|---|
| Точные pins, среда/версии и pinned path gate | 1, 3 |
| Явный scope, 13 IDs, immutable mode manifest | 2, 7 |
| Persistent domain, ownership и resume без bootstrap | 2, 3 |
| Настоящие Guest/Runtime, metadata/paused restore, unchanged production admission | 3 |
| Реальная RAM, 8 МиБ и fresh challenge, A/B/A | 1, 4 |
| Storage-only model, актуальное native state для host I/O | 4 |
| Unknown vs durable receipt, отсутствие replay | 3, 5 |
| HTTP lost replies, partial revert, live writer, cold control | 6 |
| SQL/WAL/trace/boot/binary evidence, negative controls, portable verification | 7 |
| 13/13, legacy 24/24, независимый review, чистая поставка | 8 |

Самопроверка: scope не расширен до production topology/СХД; gate разделён на
direct libvirt (Task 1) и настоящий Nova Runtime (Task 3), оба до сквозной
матрицы. Контракты типов/методов согласованы между задачами; все пять Review
Focus условий имеют явные tests. В плане нет выполненных checkbox или
заявлений о готовой интеграции. После review плана выполняется выбранный
пользователем способ исполнения, задачи по порядку.
