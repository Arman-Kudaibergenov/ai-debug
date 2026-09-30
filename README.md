# AI_Debug

Расширение 1С:Предприятие 8.3 с HTTP-API для ИИ-диагностики базы и точкой
обращений пользователей в поддержку (чат с автодосье, скриншоты, голосовые
сообщения, ответы внешнего ИИ-воркера).

Изначально основано на [vladimir-kharin/1c_mcp](https://github.com/vladimir-kharin/1c_mcp)
(HTTP-сервис в стиле MCP: tools list/call), далее доработано: диагностика
(движения, обмены, константы, техжурнал), execute_code, точка обращений
пользователей с автодосье и вложениями.

- **Дистрибутивы (.cfe)** — в [релизах](https://github.com/Arman-Kudaibergenov/ai-debug/releases)
- **Инструкция по установке** — [УСТАНОВКА.md](УСТАНОВКА.md)
- Требования: платформа 8.3.24+, конфигурация на БСП, публикация базы на веб-сервере
- Совместимость расширения: 8.3.14, префикс объектов `AI_`, режим «Использовать в основных ролях»

## HTTP-API (`/hs/ai`, Basic-аутентификация)

| Endpoint | Методы | Назначение |
|---|---|---|
| `/hs/ai/health` | GET | Проверка живости: `{"status":"healthy","version":"1.4.0"}` |
| `/hs/ai/tools/list` | GET | Список инструментов (MCP-стиль) |
| `/hs/ai/tools/call` | POST | Вызов инструмента: `{"name":"...","arguments":{...}}` |
| `/hs/ai/resources/list`, `/resources/read?uri=...` | GET | Документация (quickstart, tools, debugging, techlog, investigation) |
| `/hs/ai/support/pending` | GET | Новые обращения (для внешнего ИИ-воркера) |
| `/hs/ai/support/ticket?id=...` | GET | Обращение с историей и досье |
| `/hs/ai/support/attachment?id=...&index=...` | GET | Вложение (скриншот/аудио) |
| `/hs/ai/support/create` | POST | Создать обращение: `{"subject","text","contextLink"}` |
| `/hs/ai/support/reply` | POST | Ответ воркера: `{"id","text"}` → статус «Отвечено» |

## Инструменты (34)

- **Запросы и код**: `execute_query` — произвольный запрос (в т.ч. с параметрами
  через `paramsCode` — код заполнения параметров в безопасном режиме внутри
  всегда отменяемой транзакции); `execute_code` — произвольный серверный BSL
  (eval-выражение, при синтаксической ошибке — exec-блок с результатом через
  переменную `Результат`)
- **Справочники**: `get_catalog_item`, `find_catalog_items`,
  `create_catalog_item`, `update_catalog_item`
- **Документы**: `get_document`, `create_document`, `update_document`,
  `post_document`, `unpost_document`
- **Метаданные и ЖР**: `list_metadata_objects`, `get_metadata_structure`,
  `get_event_log`, `find_references_to_object`, `get_access_rights`,
  версионирование объектов: `get_object_versions`, `get_object_version`,
  `diff_object_versions`
- **Регистры сведений**: `get_register_records`, `write_information_register`,
  `delete_register_record`
- **Диагностика**: `get_document_postings`, `get_exchange_status`,
  `get_constants`, `get_tech_log` (чтение технологического журнала);
  управление сбором ТЖ: `get_logcfg` / `configure_logcfg` / `restore_logcfg`
  (точечный logcfg с бэкапом и возвратом; в read-only заблокированы)
- **Анонимизация ПДн**: `get_anonymization_map` (карта токен → значение
  по области, read-only), `set_anonymization_policy` (мастер-выключатель
  и правила: enable|disable|add|remove|list)
- **Системные**: `delete_object`, `run_unit_tests`, `convert_file`

## Точка обращений пользователей

Раздел «AI: Поддержка» → «Сообщить о проблеме (AI)» — чат. Расширение
автоматически собирает досье: пользователь и роли, версии конфигурации и
расширений, хвост журнала регистрации, контекст объекта (ссылка вставляется
Ctrl+V в большую зону, туда же Ctrl+V скриншота; кнопка «Запись голоса» —
голосовое, расшифровывается автоматически). Ответ ИИ появляется в чате сам
(опрос каждые 15 секунд). Обработку выполняет внешний воркер, опрашивающий
`/hs/ai/support/pending` — LLM из расширения не вызывается.

## Безопасность

- Вся защита API — Basic-аутентификация HTTP-сервиса и права служебной
  учётной записи; `execute_code` выполняет произвольный серверный код.
  Выдавайте доступ только доверенной учётке с минимально нужными правами.
- **Режим «только чтение»**: запись `ReadOnlyMode=true` в регистре
  `AI_Settings` блокирует все пишущие инструменты и `execute_code` —
  рекомендуется для продуктивных баз. Снять режим через API нельзя
  (fail-closed: delete_register_record тоже заблокирован) — только изнутри
  1С (предприятие/конфигуратор или прямое удаление записи в СУБД).
- После установки снять у расширения флаг «Безопасный режим» (иначе не
  работают запись обращений и чтение ТЖ) — см. УСТАНОВКА.md.
- **Обезличивание ПДн** (регистры `AI_AnonymizationPolicy/AI_AnonymizationMap`,
  инструменты `set_anonymization_policy` / `get_anonymization_map`): при
  включённой политике строковые значения в ответах `execute_query` и
  `execute_code` уходят наружу с токенами `[ЛИЦО_N]`, `[ИНН_N]`, `[ТЕЛЕФОН_N]`,
  `[EMAIL_N]` вместо ПДн (детекторы по имени поля и по контексту ИИН/ИНН/БИН/РНН,
  телефоны +7/8, email). Токены стабильны и реверсивны: маппинг хранится в
  `AI_AnonymizationMap`, воркер забирает его через `get_anonymization_map` и
  восстанавливает значения в финальном ответе локально. Режим правила
  `Маскировать` — необратимая замена на `[СКРЫТО]`. Политика выключена по
  умолчанию (нет мастер-записи — поведение как раньше).
- Секреты: ответы `execute_query`, `execute_code`, `get_document`,
  `get_catalog_item` и `get_register_records` проходят необратимую маскировку
  (`AI_Anonymizer`): колонки/ключи/реквизиты с признаками пароля, токена,
  ключа API заменяются на `[SECRET_REMOVED]` до отдачи наружу.

## Источники и благодарности

- [vladimir-kharin/1c_mcp](https://github.com/vladimir-kharin/1c_mcp) — базовая
  архитектура (HTTP-сервис в стиле MCP: tools list/call)
- [SteelMorgan/1c-agent-based-dev-framework](https://github.com/SteelMorgan/1c-agent-based-dev-framework) —
  `tech-log-analysis` (SKILL.md + references/scenarios.md): основа ресурса
  `ai://docs/techlog` (классификация инцидентов, корреляция, шаблон вывода)
- [SteelMorgan/1c-log-checker](https://github.com/SteelMorgan/1c-log-checker) —
  контракт управления logcfg (get/configure/restore с backup_id) →
  инструменты `*_logcfg`
- [SteelMorgan/1c-mcp-tools](https://github.com/SteelMorgan/1c-mcp-tools) —
  идея ROCTUP-границы (необратимая маскировка секретов до отдачи наружу) →
  `AI_Anonymizer`; идея `get_access_rights`
- [vgtitov/bsl-ai-toolkit](https://github.com/vgtitov/bsl-ai-toolkit) —
  `1c-expert/references/investigation-methodology.md` (MIT): основа ресурса
  `ai://docs/investigation`; `tech-journal.md` — референс по logcfg
- [palsergeich/TJ_parce](https://github.com/palsergeich/TJ_parce) —
  `docs/event-inventory.md`: грабли формата ТЖ с боевого корпуса 175 ГБ →
  раздел 7 ресурса `ai://docs/techlog`

## Структура исходников

```
AI_Debug/
├── HTTPServices/ai/          # HTTP-API (health, tools, resources, support)
├── CommonModules/
│   ├── AI_Core               # реестр и маршрутизация инструментов
│   ├── AI_Query, AI_Executor # execute_query, execute_code
│   ├── AI_Catalogs, AI_Documents, AI_Registers
│   ├── AI_Metadata           # метаданные, ЖР, поиск ссылок
│   ├── AI_Diagnostics        # движения, обмены, константы, ТЖ, logcfg
│   ├── AI_Support            # обращения: создание, досье, вложения
│   ├── AI_Security           # read-only режим, валидация параметров
│   ├── AI_Anonymizer         # маскировка секретов (ROCTUP) + анонимизация ПДн
│   └── ...                   # логгер, тесты (ЮТТесты), утилиты
├── Catalogs/AI_Обращения     # обращения пользователей
├── CommonForms/AI_ФормаЧата  # чат обращения
├── InformationRegisters/     # AI_Settings, AI_Anonymization*
└── Roles/AI_Поддержка        # роль «AI: Обращения в поддержку»
```
