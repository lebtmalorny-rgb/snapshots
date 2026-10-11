# C1: состояние реализации

Дата: 2026-10-07. [План](2026-10-06-cinder-checkpoint-contract.md) и непрерывное Native-исполнение согласованы.

## Выполнено

Рабочая копия: `work/cinder-checkpoints/`, ветка `feature/cinder-checkpoint-contract`.
Baseline `9c75625` — локальный импорт предоставленного ZIP, а не история upstream.
Архивы и `sources/` не изменяются; Nova и реальные СХД не используются.

Tasks 1–8 реализованы: native driver contract, durable journal/OVO/migration,
атомарная защита, сервисный API с квотами, RPC, scoped attachments и recovery/cleanup.
Функция выключена по умолчанию; существующие production drivers возвращают unsupported.

| Этап | Последняя проверка |
|---|---|
| Driver contract | 63 теста |
| Journal, migration, OVO | 80 тестов, включая MySQL/PostgreSQL |
| Guards | 551 API/attachment, 334 services, 116 guard/admin проверок |
| Service API | 183 теста |
| RPC/native/query | 163 теста |
| Scoped attachments | 180 профильных и 575 регрессионных тестов |
| Recovery/cleanup/quotas | 109 тестов, включая квоты на трёх СУБД |

Числа отражают отдельные прогоны с пересечениями; складывать их нельзя.

## Проверки и исправления review завершены

- Пять сквозных сценариев прошли через WSGI, SQL и настоящую сериализацию RPC с двумя fake backend.
- Sample test: 1 passed; compliance: 112 passed.
- API reference собран с `sphinx-build -W`; doc8: 0 ошибок.
- Configuration generator и проверка executable flags прошли.
- Полный flake8 содержит те же 10 замечаний, что и чистый архив; новые файлы их не добавили.
- Широкий набор: 342 модуля, 15 664 testsRun — 15 659 passed, 5 штатных skips, 0 failures/errors.
- Первый прогон выявил и помог исправить legacy attach/detach и тестовые mocks; три force-detach фикстуры обновляют Volume перед HTTP-запросом из-за изменения updated_at.
- Монолитные повторы зависали/завершались exit 139; причина интерпретаторного сбоя не доказана. Итоговая проверка использует восемь отдельных процессов, до четырёх одновременно. Набор тестов сохранён полностью; после исправлений review заново пройдены все восемь workers.
- Сквозные сценарии, samples и scoped attachments повторно прошли (20 тестов); дополнительно RPC/service 40 passed.
- Итоговая серия содержит десять патчей; HEAD `d2d14a5251fa7f61771f83bfbf50f12215dd051d`, tree `18b12459d77cab89657611f4ea1daeb3168fda10`. Доказательство применения к чистому ZIP находится в пакете.
- Независимый reviewer нашёл две гонки: stale retype/резервирование checkpoint и устаревший size/type при расчёте квот. Оба findings воспроизведены RED, исправлены и проверены GREEN; повторного review не было.
- Финальные дополнительные тесты: WSGI/scoped/sample/RPC/service 60 passed, compliance 112 passed.
- Пакет патчей и отчёты (`deliverables/cinder-checkpoint-contract-c1/README.md`) готов для локального применения; remote и инфраструктура не изменялись.

## Тестовая среда и ограничение

Первоначальные отказы Docker/DNS устранены для Linux test container. Используются
Python 3.11.13, constraints OpenStack 2025.1, MySQL 8.0.46 и PostgreSQL 16.15.
В host-среде зависимости не устанавливаются.

Пользователь разрешил пропустить закрытый `storage_client>=1.3`. Из тестовых
зависимостей исключён только этот пакет, из широкого прогона — SberCloud.
Tracked `requirements.txt` и `tox.ini` не менялись. Это не полный исходный
dependency baseline с проверкой SberCloud.

Журнал исполнения (`deliverables/cinder-checkpoint-contract-c1/evidence/progress.md`) содержит команды, результаты и решения.

**Граница результата:** C1 — общий контракт Cinder, не готовые RAM snapshots.
N1/N2 (Nova/libvirt/RAM), D1/D2 (Dorado/TATLIN adapters) и V1 (испытания на СХД)
остаются следующими отдельными пакетами.
