# Checkpoint ВМ с сохранением RAM: Nova и Cinder

Исследования и патчи для предоставленных форков OpenStack Nova/Cinder.
Публикация от **2026-10-11** включает итоговый комплект от 2026-10-10
с уточнением документации о существующих драйверах Huawei и YADRO TATLIN.

## С чего начать

1. [Текущее состояние и границы реализации](deliverables/checkpoint-patches-research-20261010/research/STATUS.md).
2. [Порядок применения патчей](deliverables/checkpoint-patches-research-20261010/APPLY.md).
3. [Подробное руководство по доработке драйверов Huawei/TATLIN](docs/driver-development.md).
4. [Компактный ZIP с патчами и исследованиями](deliverables/checkpoint-patches-research-20261010.zip)
   и [его SHA-256](deliverables/checkpoint-patches-research-20261010.zip.sha256).

## Состав

| Материалы | Содержание |
|---|---|
| [Патчи Nova](deliverables/checkpoint-patches-research-20261010/patches/nova/) | 23 патча: модель/API N1, исполнитель N2 и три интеграционных исправления |
| [Патчи Cinder](deliverables/checkpoint-patches-research-20261010/patches/cinder/) | 17 патчей: контракт C1 и продолжение операций C1.1 |
| [Исследования](deliverables/checkpoint-patches-research-20261010/research/) | Четыре отчёта, приложения с доказательствами из исходников, шаблон профиля СХД и итог проверок |
| [Спецификации](docs/superpowers/specs/) | Архитектура checkpoint, Nova N1/N2, локальная интеграция, RAM и Dorado D1-P |
| [Планы и статус этапов](docs/superpowers/plans/) | Датированные планы реализации и исторический статус выполнения |
| [Руководство разработчика драйверов](docs/driver-development.md) | Методы и структуры Cinder, использование документации Huawei, recovery, локальные и стендовые проверки |

Все четыре исследовательских отчёта:

- [RAM snapshots и расширение Nova](deliverables/checkpoint-patches-research-20261010/research/openstack_vm_snapshot_ram_nova_extension.md).
- [Вывод по Huawei Dorado и YADRO TATLIN](deliverables/checkpoint-patches-research-20261010/research/openstack_ram_checkpoint_dorado_tatlin_conclusion.md).
- [Готовность драйверов к checkpoint-контракту](deliverables/checkpoint-patches-research-20261010/research/2026-10-08-checkpoint-vendor-readiness.md).
- [Исследование Huawei Dorado без оборудования](deliverables/checkpoint-patches-research-20261010/research/2026-10-08-dorado-without-hardware.md).

## Что реализовано

Nova управляет API и жизненным циклом checkpoint, журналами, квотами,
блокировками, сохранением и восстановлением RAM через libvirt. Cinder
предоставляет контракт дисковых операций, защиту ресурсов, продолжение
операций сервисом и восстановление подключений томов.

**Драйверы Huawei и YADRO TATLIN уже присутствуют в исходном `cinder-pvs.zip`.**
Следующая задача — доработать их под checkpoint-контракт, используя имеющиеся
операции и добавляя недостающие возможности отката и проверки результата.
Методы `checkpoint_*` в этих драйверах пока не переопределены; наследуется
базовое поведение `unsupported_driver`. Создавать весь драйвер с нуля
не требуется.

Реализацию можно разрабатывать по документации производителя и проверять
локально до появления оборудования. Испытания на массиве отдельно
подтверждают работу выбранного профиля. Руководство разделяет эти этапы
и отмечает вопросы, по которым ещё нужны сведения из REST Reference.

Сохранение и восстановление настоящей RAM проверены на локальном госте
QEMU/libvirt. Дисковые операции моделировались; совместный откат настоящих
дисков и RAM ещё не подтверждён. Поддержка конкретных Dorado/TATLIN,
KVM и production topology не квалифицирована. Функция выключена по умолчанию.
Датированные планы описывают работу на момент их создания; актуальный итог
приведён в [STATUS.md](deliverables/checkpoint-patches-research-20261010/research/STATUS.md).

## Применение и проверка

Патчи предназначены для исходных `nova-pvs.zip` и `cinder-pvs.zip`;
эти исходные архивы нужны отдельно. Их хеши и точные промежуточные и итоговые
Git trees зафиксированы в
[MANIFEST.json](deliverables/checkpoint-patches-research-20261010/MANIFEST.json).
Порядок: **Cinder C1 → C1.1**, **Nova N1 → N2 → fixes**.
Применение кода и развёртывание сервисов — отдельные этапы, описанные в
[APPLY.md](deliverables/checkpoint-patches-research-20261010/APPLY.md).

Из корня этого репозитория проверить целостность публикации:

```sh
shasum -a 256 -c SHA256SUMS
```

[PUBLICATION.json](PUBLICATION.json) фиксирует происхождение опубликованных
файлов. Вложенный комплект и ZIP сохранены побайтно; собственные manifests
и контрольные суммы комплекта действуют внутри его каталога. Ссылки и пути
к исходному коду, лаборатории и логам в исторических материалах относятся
к локальной рабочей папке; внешние для этой публикации ссылки показаны
текстовыми путями. Эти материалы не заявляют наличие СХД или готовность
внедрения в работающий OpenStack.

При публикации проверяются файлы и патчи; сохранённые результаты тестов
сохраняют свои исходные даты. Новый runtime-прогон не выполнялся.
