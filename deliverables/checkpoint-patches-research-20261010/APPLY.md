# Порядок применения патчей

Серии предназначены для исходников из предоставленных `nova-pvs.zip` и
`cinder-pvs.zip`. Архивы в этот компактный комплект не входят. Их SHA-256,
локальные baseline commits и промежуточные trees записаны в
[MANIFEST.json](MANIFEST.json); SHA-256 также лежат в каталогах серий.
Baseline commits получены локальным импортом ZIP, это не ссылки на upstream.

Порядок в каждом репозитории обязателен:

- Cinder: `c1` → `c11` (17 патчей).
- Nova: `n1` → `n2` → `fixes` (23 патча).

Полный порядок задан файлами `patches/cinder/series` и `patches/nova/series`.
Применять только оставшиеся стадии можно к точно совпадающему промежуточному
tree из MANIFEST. Повторно применять уже установленные стадии не нужно.

## Применение полной серии

Нужны Git и чистый отдельный checkout базовой версии выбранного продукта.
Задайте абсолютные пути. Сначала выполните для Cinder; для Nova повторите
с `product=nova` и путём к базовому checkout Nova.

```sh
bundle=/absolute/path/to/checkpoint-patches-research-20261010
product=cinder
repo=/absolute/path/to/clean-cinder-checkout

(
    set -eu
    cd "$bundle"
    shasum -a 256 -c SHA256SUMS
    cd "$repo"
    test -z "$(git status --porcelain)"
    test "$(git rev-parse 'HEAD^{tree}')" = "$(cat "$bundle/patches/$product/BASE_TREE")"
    git switch -c checkpoint-patches-20261010
    while IFS= read -r patch; do
        git am "$bundle/patches/$product/$patch"
    done < "$bundle/patches/$product/series"
    test "$(git rev-parse 'HEAD^{tree}')" = "$(cat "$bundle/patches/$product/HEAD_TREE")"
    git status --short
)
```

Ошибка останавливает последовательность. Если не совпал baseline tree,
сначала установите причину различий исходников; проверка на данном форке
не доказывает совместимость с другой веткой. При конфликте `git am` следующие
патчи не применяются. `git am --abort` отменяет только текущую незавершённую
операцию; предыдущие успешно применённые коммиты остаются в новой ветке.

Commit SHA после `git am` могут отличаться из-за committer/date; критерием
идентичности файлов является итоговый Git tree.

## Что получается

Это применение изменений к коду. Запуск служб, миграции БД и включение
функции выполняются отдельным этапом. При будущем развёртывании нужно
учесть новые схемы БД, API/RPC и конфигурацию из документации внутри патчей:

- Cinder: `doc/source/admin/checkpoint-contract.rst` и документация continuation.
- Nova: административная документация checkpoint model/executor в `doc/source/admin/`.

Согласованная пара продуктов — финальные Nova и Cinder из MANIFEST.
Драйверы Huawei и YADRO TATLIN уже присутствуют в исходном `cinder-pvs.zip`.
Эти патчи добавляют общий checkpoint-контракт; его методы в существующих
Huawei/TATLIN пока не реализованы. Следующий этап — доработка соответствующего
драйвера, включая откат и проверку исхода операции после потери ответа.
Положительные профили Dorado/TATLIN отсутствуют: обычный snapshot/revert
драйвера не заменяет реализацию checkpoint-контракта.
