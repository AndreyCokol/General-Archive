# План изменений General-Archive: решения, порядок, откат

**Статус:** предложения на согласование; действующие `README.md`, `prompts/*`, `PROJECT.template.md` не изменены.  
**Baseline:** `0c890668cedb884ff8ae441bffe26755aea99c21`. Сначала ознакомиться с [аудитом](2026-11-05-prompt-audit.md), [моделью ролей](2026-11-05-role-model.md), [дополнительным аудитом](2026-11-05-deep-operational-audit.md).

## Вопросы, требующие решения владельца

1. **A1–A4:** согласовать подход *четыре универсальных слота разработчиков* + специализации только в PROJECT.md / optional role profiles. Не следует автоматически менять роли действующих проектов.
2. **Полномочия A5:** может ли A5 самостоятельно утверждать первичную техническую декомпозицию, если scope не меняется, или всегда требуется owner approval? Поле делегирования должно фиксировать ответ.
3. **Ревью и merge:** кто является backup reviewer A5, сколько требуется approvals, есть ли исключения для одиночных маленьких проектов?
4. **Lease:** выбрать атомарный координационный сервис (для полноценного параллелизма) либо последовательную выдачу разрешений A5 (для MVP), не имитировать lock метками.
5. **Видимость репозитория:** сохранить публичным или закрыть; нужен ли LICENSE для внешнего reuse?
6. **Release:** когда допускается выпуск версии архива и как проекты мигрируют со старого закреплённого SHA?

## Рекомендуемые пакеты PR (раздельно)

| Пакет | Приоритет | Файлы для будущего изменения | Содержимое | Приёмка |
|---|---|---|---|---|
| R1 | P1 | `PROJECT.template.md`, `prompts/COMMON.md`, `prompts/A5.md` | APPROVED/DRAFT, authority matrix, review policy, запрет direct-to-main, доступность инструментов | S01, S02, S11, S12, S15, S21, S28, S30 |
| R2 | P1 | `prompts/COMMON.md`, шаблоны Issues/PR, docs protocol | Task state machine, serial arbitration/CAS lease, fencing, handoff и evidence | S03–S10, S16–S17, S25–S27 |
| R3 | P1 | `VERSION`, `CHANGELOG.md`, source-manifest, `README.md`, `prompts/COMMON.md` | Provenance ChemSim/My_shelter, immutable SHA, обновления/rollback, accepted release | S22–S24 |
| R4 | P1 | `.github/workflows/*`, `tools/*`, `tests/*` | Link checks, markdown, prompt-structure lint, негативные scenario fixtures | Сломанная ссылка падает; все P1 сценарии проверены |
| R5 | P2 | `prompts/A1.md`–`A4.md`, `PROJECT.template.md`, optional `profiles/` | Слоты разработчиков и опциональные технологии/модули | S18, S19; backend-only и UI проекты |
| R6 | P2 | `prompts/A5.md`, review policy docs | Делегирование и backup review, SLA эскалации, owner approval для спорных задач | S11, S19, S30 |
| R7 | P2 | `SECURITY.md`, `README.md`, `PROJECT.template.md` | Границы внешних операций, данные/секреты, incident, rollback | S13, S14, S28 |

Приоритеты относятся к **подготовке безопасного универсального шаблона для команд с доступом на запись**, не к текущему подтверждённому production incident.

## Последовательность

**Этап 0 — сохранить baseline.** Записать SHA, исходные ссылки и snapshot/provenance; убедиться, что текущие пользователи шаблонов закрепляют версию. Не удалять оригиналы из ChemSim/My_shelter. Снять список возможных downstream проектов (если они известны).

**Этап 1 — установить права и технический процесс.** Ввести authority/review/PROJECT approvals, запрет прямых коммитов в base; согласовать branch protection. До этого не раздавать пяти агентам неограниченный write-доступ как якобы безопасный параллелизм.

**Этап 2 — определить владение/статусы.** Реализовать либо CAS lease, либо честный single-coordinator режим; сделать contract registry и evidence template. Проверить гонку на синтетическом Issue.

**Этап 3 — проверить качество.** Добавить CI, линтеры и сценарии. Убедиться, что проверки падают на заведомой ошибке, а не просто «зелёные» на пустом наборе.

**Этап 4 — универсализировать роли.** После утверждения модели заменить жёсткую специализацию A1–A4 на проектные назначения; обновить A5 и PROJECT. В проектах, где роли уже распределены, сохранить старые назначения до отдельной миграции.

**Этап 5 — pilot и выпуск.** Один backend-only и один UI/game-like синтетический проект; пять задач, минимум один конфликт lease, один breaking contract, один `NOT_RUN` и независимый ревьюер. Сохранить evidence. Только после этого новый tag/версия и checklist миграции.

## Definition of Done для изменения самих промптов

- Есть согласованная задача, исходный SHA и запись одобрения владельца.
- Diff не отменяет ограничения более высокого приоритета и специальных проектов.
- Все новые требования перечисляют доступные fallbacks при отсутствии tools/permission.
- Подтверждены lint/link checks, scenario regression и независимое ревью по **итоговому SHA**.
- Обновлены lock/version/changelog, compat note, owner decisions.
- Подготовлен rollback на прежний SHA; нет утечек secrets.

## Отдельные точечные правки по файлам

- `README.md`: краткая процедура pin/upgrade, source manifest и warning «proposals != policy».
- `PROJECT.template.md`: `status: DRAFT/APPROVED/SUPERSEDED`, `approved_by`, `approval_source`, `source_commit`, `authority_matrix`, `review_policy`, `task_registry`, `contract_registry`, `lease_mode`, `data_classification`, `release_policy`; обязательность полей/осмысленное N/A.
- `COMMON.md` §1: проверка прав и tool capability, полномочия по операциям; §2: delta-checkpoint; §4: lease/fencing; §5: contract registry; §6–8: approvals, evidence bundle и no-direct-base; §9: формализованный handoff.
- `A1.md`–`A4.md`: общая модель основных разработчиков, не навязывать «ядро/API/данные/UI» каждому проекту; обязанности/проверки уточняются PROJECT и optional profile.
- `A5.md`: координация/архитектура и veto на небезопасную интеграцию, но не самовольное решение владельца; backup reviewer и эскалация при разногласиях.
- Новые шаблоны Issue/PR: task, contract change, review evidence, conflict; `sources/`, `tools/`, `tests/` — после отдельного утверждения.

## Критерий принятия предложений

Владелец может по каждому пакету проставить `ACCEPT / REVISE / REJECT / DEFER` и короткое обоснование. До решения **не считать** ни один из файлов в `proposals/` обязательным для A1–A5. По утверждённым пакетам создавать отдельные PR и верифицировать по плану. Для опасной регрессии — revert PR/возврат закреплённой версии, исправление с новым тестом и повторное утверждение.
