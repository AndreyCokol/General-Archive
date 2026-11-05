# Предложение: операционный протокол задач, lease, handoff и контрактов

**Статус:** draft proposal для `prompts/COMMON.md`, `PROJECT.template.md` и шаблонов GitHub. Не является автоматической блокировкой в существующем репозитории. Связанные замечания: OP-02, OP-03, OP-07, OP-08, OP-12.

## A. Жизненный цикл задачи — три независимых измерения

1. `workflow_state`: `BACKLOG → READY → CLAIMED → IN_PROGRESS → IN_REVIEW → IN_INTEGRATION → DONE`. Допускаются `BLOCKED` (с `blocked_from_state`, причиной, owner решения) и `CANCELLED`. После неудачного ревью возвращать в `IN_PROGRESS`, а не объявлять DONE.
2. `verification`: для **каждого** gate `PASS | FAIL | NOT_RUN | N/A` с объяснением N/A. `FAIL` и блокирующий `NOT_RUN` запрещают DONE.
3. `priority` и `milestone`: отдельно от workflow; P1 относится к конкретному согласованному выпуску, не абстрактному «когда-нибудь MVP».

Любой переход фиксирует `task_id, actor, at, from, to, base_sha, evidence_url`. `PARTIAL` лучше использовать как долю выполненных критериев, а не конкурирующее итоговое состояние. Закрытый Issue **не всегда** эквивалентен DONE.

## B. Гарантии владения и lease

GitHub labels, assignees и comments **не дают атомарного compare-and-swap**. Поэтому возможны два режима:

**Режим 1 (предпочтительный):** сериализующий claim-coordinator или сервис с транзакцией/CAS и fencing token.  
**Режим 2 (без инфраструктуры):** все claim/renew/takeover **последовательно утверждает A5**, который выдаёт однозначный grant в Issues или реестре; агенты **не делают самостоятельных claim по меткам**. Такой режим даёт операционную дисциплину, но не техническую гарантию от потери/дублирования — это явно указать.

### Grant: обязательные поля

```yaml
task_id: TASK-123
agent_id: A2
scope_paths: ["src/api/"]
contract_ids: ["api.v1"]
base_sha: "<sha>"
lease_id: "<unique-grant-id>"
lease_generation: 3
issued_by: A5
issued_at: "<ISO-8601>"
expires_at: "<ISO-8601>"
state: CLAIMED
```

- Агент до изменения сверяет `scope_paths`, свой grant и актуальную ветку. До claim не менять пересекающиеся исходники.
- Продление возможно только если grant активен, generation совпадает и не истёк.
- При потере связи: не продолжать публикацию изменений, зависящих от неподтверждённого grant.
- При takeover: проверить конфликтующие ветки и PR, дождаться отзыва/expiry старого grant, выдать generation+1. Старый агент должен **прекратить запись** (fencing).
- Независимые задачи с непересекающимися файлами могут идти параллельно. Общие файлы (конфигурации, монорепо root, схемы) имеют отдельного владельца/очередь интеграции.
- Проверять generation **перед публикацией результата**, а не лишь в начале сессии.

### Контрпример и проверка

A2 и A3 одновременно читают свободный Issue. Два комментария «беру» не подтверждают эксклюзивность. Ожидаемое: только один grant с уникальным generation; второй получает conflict, не редактирует пересекающиеся файлы и сообщает A5. При недоступности координатора ни один агент не пишет в перекрывающийся scope.

## C. Contract registry

Предлагаемая обязательная запись:

```yaml
contract_id: api.v1
path: contracts/api.schema.json
provider: A2
consumers: [A1, A3, A4]
owner_review_required: true
consumer_reviews_required: true
schema_sha: "<sha>"
compatibility: backward-compatible
validation: "command supplied by PROJECT.md"
breaking_change_plan: "<PR or N/A>"
status: approved
```

Правило: если меняется публичная схема, сначала зарегистрировать proposal `expected → actual`, список потребителей и план миграции. Меняет код **владелец пути/контракта**, A5 проверяет межмодульный эффект, а потребители подтверждают совместимость через contract-test или объяснимую ручную проверку. Интегратор не копирует и не переписывает чужие модели молча.

## D. Handoff / evidence bundle

```yaml
task: TASK-123
agent: A2
branch: agent/A2/TASK-123-api
base_sha: "<sha>"
result_sha: "<sha>"
pr: "<URL-or-none>"
lease_generation: 3
acceptance:
  - criterion: AC-1
    outcome: PASS
    evidence: "<artifact-url>"
checks:
  - command: "pytest -q"
    environment: "example only, replace with actual"
    result: NOT_RUN
    reason: "example document; not executed"
contracts_changed: []
risks: []
reviewer: null
overall_state: IN_REVIEW
```

Это **образец**, не сообщение о выполненных тестах. Обязательны: реальные команды, версии/окружение, актуальный SHA, доказательство PASS/FAIL/NOT_RUN, reviewer, список затронутых контрактов, нерешённые проблемы. Моки и локальные симуляции отдельно от реальной E2E проверки. Если инструмента нет — `NOT_RUN`, полезный патч/артефакт и минимальный следующий шаг; не симулировать успешный запуск.

## E. Checkpoint / продолжение работы

Хранить `last_verified_sha, task_id, issue_url, pr_url, lease_generation, open_blockers, tests_run, unmerged_changes, next_action`. В запросе «Продолжай» начать с `git diff`/сверки изменений от этой точки; **повторить полный аудит**, если поменялись: базовая ветка, контракт, назначение/разрешения, CI, владелец, scope, критическая зависимость или обнаружен security incident.

## F. Эскалация и честная готовность

В споре владельцев → заморозить **только пересекающуюся запись**, документировать ожидаемое/фактическое и согласовать A5 + владельцы контрактов; при изменении scope или разногласии A5 и владельца модуля — решение пользователя-владельца проекта либо его документированного делегата. Остальные независимые работы продолжаются.

`DONE` невозможно при невыполненной обязательной проверке, отсутствии требуемого review или только mock-интеграции. Merge по команде ответственного не заменяет проверки. Не назначать самому себе reviewer на свой PR.
