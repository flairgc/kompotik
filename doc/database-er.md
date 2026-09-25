# ER-диаграмма базы Kompotik

Статус: визуализация [проекта схемы БД](database.md), не реализованная схема. Дата: 25 сентября 2026 года.

Ниже — общий вид и отдельные диаграммы по областям. Разделение позволяет читать связи без одной огромной схемы. Таблицы, повторённые в разных диаграммах, обозначают одну и ту же сущность. Показаны ключи и основные предметные поля; полный перечень полей и ограничений находится в [описании БД](database.md).

Диаграммы записаны в Mermaid и отображаются в Markdown-просмотрщиках с его поддержкой.

## Обозначения

- `PK` — первичный ключ; несколько PK-полей образуют один составной ключ.
- `FK` — внешний ключ; несколько полей могут образовывать один составной FK.
- `UK` — уникальное поле. Составные и условные ограничения уникальности описаны текстом под схемой.
- `||` — ровно один; `o|` / `|o` — ноль или один; `o{` / `}o` — ноль или много; `|{` / `}|` — один или много.
- Сплошная линия — идентифицирующая связь: ключ родителя входит в PK дочерней записи. Пунктир — остальные FK-связи. Пунктир не означает отсутствие ограничения БД.
- Комментарий `nullable` означает необязательное поле. Обязательность связей показана также символами у концов линий.

## 1. Общий вид предметной области

```mermaid
erDiagram
    users["users"]
    folders["folders"]
    folderCards["folder_cards"]
    languages["languages"]
    lexicalEntries["lexical_entries"]
    senses["senses"]
    cards["cards"]
    cardRevisions["card_revisions"]
    userCardProgress["user_card_progress"]
    lessonResults["lesson_results"]
    lessonResultCards["lesson_result_cards"]
    imports["imports"]
    importCandidates["import_candidates"]

    users ||..o{ folders : owns
    folders |o..o{ folders : parent
    folders ||--o{ folderCards : contains
    cards ||--o{ folderCards : included_in
    languages ||..o{ lexicalEntries : language
    lexicalEntries ||..o{ senses : meanings
    senses ||..o{ cards : represented_by
    cards ||--o{ cardRevisions : versions
    users ||--o{ userCardProgress : learns
    cards ||--o{ userCardProgress : progress
    users ||..o{ lessonResults : completes
    lessonResults ||--|{ lessonResultCards : reports
    cards ||--o{ lessonResultCards : practiced
    users ||..o{ imports : submits
    imports ||..o{ importCandidates : produces
    senses |o..o{ importCandidates : selected_sense
```

Связь папок и карточек — многие ко многим через `folder_cards`. Прогресс относится к паре пользователь/карточка, а не к папке. Один смысл может иметь общую и несколько приватных карточек. Общая схема намеренно опускает медиа, авторизацию и технические таблицы — они раскрыты ниже.

## 2. Пользователи, устройства и настройки

```mermaid
erDiagram
    users["users"] {
        uuid id PK
        text normalized_email UK
        text password_hash
        text status
    }
    userSettings["user_settings"] {
        uuid user_id PK, FK
        text interface_language
        text timezone
        smallint cards_per_lesson
        boolean writing_enabled
        uuid effective_policy_version FK
        bigint version
    }
    devices["devices"] {
        uuid id PK
        uuid user_id FK
        text platform
        bigint last_result_sequence
        uuid last_result_id FK "nullable"
    }
    authSessions["auth_sessions"] {
        uuid id PK
        uuid user_id FK
        uuid device_id FK
        text credential_hash
        uuid replaced_by_id FK "nullable"
        timestamptz expires_at
    }
    learningPolicies["learning_policies"] {
        uuid id PK
        uuid user_id FK "nullable: common policy"
        text algorithm_version
        jsonb config
    }
    lessonResults["lesson_results"] {
        uuid id PK
        uuid device_id FK
    }

    users ||--|| userSettings : settings
    users ||..o{ devices : registers
    users ||..o{ authSessions : authenticates
    devices ||..o{ authSessions : sessions
    authSessions |o..o{ authSessions : replaced_by
    users |o..o{ learningPolicies : owner
    learningPolicies ||..o{ userSettings : effective_policy
    devices ||..o{ lessonResults : sends
    lessonResults |o..o{ devices : last_result
```

Одна запись настроек на пользователя создаётся при регистрации. FK сам по себе не гарантирует существование настроек у каждого пользователя — это инвариант команды регистрации. `auth_sessions` — сессии авторизации, не занятий. Ссылка на последний отчёт устройства не делает его владельцем отчёта: согласованность пользователя/устройства проверяется отдельно. Кардинальность самоссылки сессий отражает только описанный FK; правила единственного преемника при ротации проверяются командой авторизации.

## 3. Словарь и источники

```mermaid
erDiagram
    languages["languages"] {
        uuid id PK
        text code UK
        text status
        bigint catalog_version
    }
    users["users"] {
        uuid id PK
    }
    lexicalEntries["lexical_entries"] {
        uuid id PK
        uuid language_id FK
        text lemma
        text kind
        text visibility
        uuid owner_user_id FK "nullable"
    }
    senses["senses"] {
        uuid id PK
        uuid lexical_entry_id FK
        uuid definition_language_id FK
        text definition
        text visibility
        uuid owner_user_id FK "nullable"
    }
    contentSources["content_sources"] {
        uuid id PK
        text provider
        text dataset_version
        text license_name "nullable"
        text attribution "nullable"
    }
    senseSources["sense_sources"] {
        uuid source_id PK, FK
        text external_sense_id PK
        uuid sense_id FK
        text external_entry_id
    }

    languages ||..o{ lexicalEntries : source_language
    languages ||..o{ senses : definition_language
    users |o..o{ lexicalEntries : private_owner
    users |o..o{ senses : private_owner
    lexicalEntries ||..o{ senses : meanings
    contentSources ||--o{ senseSources : identifies
    senses ||..o{ senseSources : provenance
```

Одинаковое написание не является уникальным ключом лексической единицы. Стабильный внешний ID смысла уникален в пределах источника, а не всего приложения. Приватный владелец nullable только для общего контента; общая запись не может зависеть от приватной словарной записи.

## 4. Карточки, ревизии и медиа

```mermaid
erDiagram
    senses["senses"] {
        uuid id PK
    }
    languages["languages"] {
        uuid id PK
    }
    users["users"] {
        uuid id PK
    }
    cards["cards"] {
        uuid id PK
        uuid sense_id FK
        uuid language_id FK
        uuid translation_language_id FK
        uuid owner_user_id FK "nullable"
        text visibility
        integer published_revision FK "nullable: composite with id"
        integer draft_revision FK "nullable: composite with id"
    }
    cardRevisions["card_revisions"] {
        uuid card_id PK, FK
        integer revision PK
        text expression
        text translation
        jsonb morphology
        jsonb accepted_forms
        text state
    }
    cardArticleForms["card_article_forms"] {
        uuid id PK
        uuid card_id FK
        integer revision FK
        text article
        text written_form
        integer position
    }
    cardPronunciations["card_pronunciations"] {
        uuid id PK
        uuid card_id FK
        integer revision FK
        uuid form_id FK "nullable"
        uuid asset_id FK "nullable"
        uuid source_id FK "nullable"
        text locale
        text ipa "nullable"
    }
    mediaAssets["media_assets"] {
        uuid id PK
        text storage_key UK
        text kind
        text sha256
        uuid owner_user_id FK "nullable"
        uuid source_id FK "nullable"
        uuid operation_id "nullable: external operation"
        text state
    }
    cardRevisionMedia["card_revision_media"] {
        uuid card_id PK, FK
        integer revision PK, FK
        text role PK
        uuid asset_id FK
    }
    contentSources["content_sources"] {
        uuid id PK
    }
    cardRevisionSources["card_revision_sources"] {
        uuid card_id PK, FK
        integer revision PK, FK
        uuid source_id PK, FK
    }

    senses ||..o{ cards : meaning
    languages ||..o{ cards : source_language
    languages ||..o{ cards : translation_language
    users |o..o{ cards : private_owner
    cards ||--o{ cardRevisions : versions
    cardRevisions |o..o| cards : published_pointer
    cardRevisions |o..o| cards : draft_pointer
    cardRevisions ||..o{ cardArticleForms : article_forms
    cardRevisions ||..o{ cardPronunciations : pronunciations
    cardArticleForms |o..o{ cardPronunciations : spoken_form
    mediaAssets |o..o{ cardPronunciations : audio
    cardRevisions ||--o{ cardRevisionMedia : media_roles
    mediaAssets ||..o{ cardRevisionMedia : image
    users |o..o{ mediaAssets : private_owner
    contentSources |o..o{ mediaAssets : provenance
    contentSources |o..o{ cardPronunciations : provenance
    cardRevisions ||--o{ cardRevisionSources : citations
    contentSources ||--o{ cardRevisionSources : cited_source
```

FK ревизии — пара `(card_id, revision)`, а не отдельная ссылка на номер revision. Указатели карточки используют пары `(id, published_revision)` и `(id, draft_revision)`, поэтому указывают только на собственные версии. Одна ревизия может быть текущей опубликованной/черновой максимум у своей карточки.

У общей неархивной карточки уникальна пара `(sense_id, translation_language_id)`. Для приватных карточек такого ограничения нет. Порядок формы уникален в пределах ревизии; default-произношение ограничено одним на каждую форму ревизии, включая основную форму с `form_id=null`. Готовое аудио переиспользуется, но новое содержимое файла получает новый `media_assets.id` и ключ.

## 5. Папки и готовые подборки

```mermaid
erDiagram
    users["users"] {
        uuid id PK
    }
    languages["languages"] {
        uuid id PK
    }
    folders["folders"] {
        uuid id PK
        uuid user_id FK
        uuid language_id FK
        uuid parent_id FK "nullable: root"
        uuid origin_collection_id FK "nullable"
        boolean is_root
        text name
    }
    folderCards["folder_cards"] {
        uuid folder_id PK, FK
        uuid card_id PK, FK
    }
    cards["cards"] {
        uuid id PK
    }
    catalogCollections["catalog_collections"] {
        uuid id PK
        uuid language_id FK
        text title
        bigint version
    }
    catalogCollectionCards["catalog_collection_cards"] {
        uuid collection_id PK, FK
        uuid card_id PK, FK
        integer position
    }

    users ||..o{ folders : owns
    languages ||..o{ folders : language
    folders |o..o{ folders : parent
    folders ||--o{ folderCards : contains
    cards ||--o{ folderCards : included_in
    languages ||..o{ catalogCollections : language
    catalogCollections ||--o{ catalogCollectionCards : composition
    cards ||--o{ catalogCollectionCards : included_in
    catalogCollections |o..o{ folders : copied_from
```

Для корней уникальна пара `(user_id, language_id)`. Проверка дерева запрещает циклы и родителя другого владельца/языка. Связь `copied_from` сохраняет происхождение, но не означает синхронизацию состава личной папки с подборкой. Удаление связи из папки не удаляет карточку или её прогресс.

## 6. Прогресс и история обучения

```mermaid
erDiagram
    users["users"] {
        uuid id PK
    }
    cards["cards"] {
        uuid id PK
    }
    learningPolicies["learning_policies"] {
        uuid id PK
    }
    userCardProgress["user_card_progress"] {
        uuid user_id PK, FK
        uuid card_id PK, FK
        uuid policy_version FK
        uuid last_applied_result_id FK "nullable"
        uuid learning_cycle_id
        uuid review_occurrence_id "nullable"
        text state
        smallint score
        timestamptz next_review_at "nullable"
        bigint version
    }
    lessonResults["lesson_results"] {
        uuid id PK
        uuid user_id FK
        uuid device_id FK
        uuid language_id FK
        uuid policy_version FK
        uuid previous_result_id FK "nullable"
        uuid lesson_id "local ID"
        bigint device_sequence
        date activity_date
        text mode
    }
    lessonResultCards["lesson_result_cards"] {
        uuid result_id PK, FK
        uuid card_id PK, FK
        integer card_revision FK
        uuid base_result_id FK "nullable"
        uuid learning_cycle_id
        text application
    }
    exerciseResults["exercise_results"] {
        uuid result_id PK, FK
        uuid card_id PK, FK
        text type PK
        boolean first_answer_correct
        uuid pronunciation_id FK "nullable"
        uuid audio_asset_id FK "nullable"
        uuid form_id FK "nullable"
    }
    learningEvents["learning_events"] {
        uuid id PK
        uuid user_id FK
        uuid card_id FK
        uuid result_id FK "nullable"
        uuid learning_cycle_id
        text type
        jsonb before_state
        jsonb after_state
    }
    reviewCredits["review_credits"] {
        uuid user_id PK, FK
        uuid card_id PK, FK
        uuid learning_cycle_id PK
        uuid review_occurrence_id PK
        uuid result_id FK
    }

    users ||--o{ userCardProgress : learns
    cards ||--o{ userCardProgress : progress
    learningPolicies ||..o{ userCardProgress : rules
    users ||..o{ lessonResults : completes
    learningPolicies ||..o{ lessonResults : fixed_rules
    lessonResults |o..o{ lessonResults : predecessor
    lessonResults |o..o{ userCardProgress : last_applied
    lessonResults ||--|{ lessonResultCards : reports
    cards ||--o{ lessonResultCards : practiced
    lessonResults |o..o{ lessonResultCards : base_result
    lessonResultCards ||--o{ exerciseResults : answers
    users ||..o{ learningEvents : acts
    cards ||..o{ learningEvents : changes
    lessonResults |o..o{ learningEvents : causes
    userCardProgress ||--o{ reviewCredits : credited_occurrences
    lessonResults ||..o{ reviewCredits : earns
```

Чтобы не перегружать схему, следующие межобластные FK перечислены отдельно:

| Дочерняя запись | Ссылка | Обязательность |
| --- | --- | --- |
| `lesson_results` | `device_id → devices.id`, `language_id → languages.id` | Обязательные. |
| `lesson_result_cards` | `(card_id, card_revision) → card_revisions.(card_id, revision)` | Обязательная пара. |
| `exercise_results` | `pronunciation_id → card_pronunciations.id`, `audio_asset_id → media_assets.id`, `form_id → card_article_forms.id` | Nullable; наличие зависит от упражнения. |

Уникальны `(user_id, lesson_id)` и `(user_id, device_id, device_sequence)`. PK `review_credits` не позволяет дважды зачесть один срок в одном цикле. Поля `learning_cycle_id`, `review_occurrence_id` и локальный `lesson_id` не являются FK на скрытые таблицы сессий. Для новой карточки строка прогресса может отсутствовать — API возвращает виртуальное начальное состояние.

Минимум одна карточка в принятом уроке — инвариант приёма отчёта. Наличие/количество упражнений проверяется по плану и политике; ER-связь допускает пустой набор, но не объявляет его успешным результатом.

## 7. Импорт и фоновые задания

```mermaid
erDiagram
    users["users"] {
        uuid id PK
    }
    imports["imports"] {
        uuid id PK
        uuid user_id FK
        uuid language_id FK
        uuid active_job_id FK "nullable"
        uuid applied_folder_id FK "nullable"
        text status
        bigint version
    }
    importCandidates["import_candidates"] {
        uuid id PK
        uuid import_id FK
        text analysis_item_key
        uuid selected_sense_id FK "nullable"
        uuid selected_card_id FK "nullable"
        uuid created_card_id FK "nullable"
        text resolution_state
        boolean selected
    }
    candidateSenseOptions["candidate_sense_options"] {
        uuid candidate_id PK, FK
        uuid sense_id PK, FK
        integer rank
    }
    candidateOccurrences["candidate_occurrences"] {
        uuid id PK
        uuid candidate_id FK
        jsonb segments
        text context
    }
    senses["senses"] {
        uuid id PK
    }
    cards["cards"] {
        uuid id PK
    }
    jobs["jobs"] {
        uuid id PK
        uuid user_id FK "nullable: system job"
        uuid parent_job_id FK "nullable"
        uuid resource_id "polymorphic reference, not FK"
        text kind
        text status
        uuid attempt_token "nullable"
    }
    jobAttempts["job_attempts"] {
        uuid id PK
        uuid job_id FK
        integer attempt_number
        uuid operation_id "external operation, not FK"
        text status
    }

    users ||..o{ imports : submits
    users |o..o{ jobs : requests
    jobs |o..o{ jobs : parent
    jobs ||..o{ jobAttempts : attempts
    jobs |o..o{ imports : active_job
    imports ||..o{ importCandidates : produces
    importCandidates ||--o{ candidateSenseOptions : alternatives
    senses ||--o{ candidateSenseOptions : candidate_meaning
    importCandidates ||..o{ candidateOccurrences : occurrences
    senses |o..o{ importCandidates : selected_sense
    cards |o..o{ importCandidates : selected_card
    cards |o..o{ importCandidates : created_card
```

Межобластные FK: `imports.language_id → languages.id` обязателен; `imports.applied_folder_id → folders.id` nullable. Уникальны `(import_id, analysis_item_key)` и `(job_id, attempt_number)`. Выбранный смысл и созданная карточка появляются по мере разрешения кандидата; они не обязательны при первом результате анализа.

`jobs.resource_id` — полиморфная предметная ссылка, а `operation_id` — ID операции специализированного сервиса. FK-линии к ним намеренно не нарисованы. Реестр операций отдельного сервиса описан в [контрактах сервисов](service-interfaces.md), он не является частью предметной схемы основного приложения.

## 8. Офлайн-пакеты и синхронизация

```mermaid
erDiagram
    users["users"] {
        uuid id PK
    }
    folders["folders"] {
        uuid id PK
    }
    jobs["jobs"] {
        uuid id PK
    }
    learningPolicies["learning_policies"] {
        uuid id PK
    }
    offlinePackages["offline_packages"] {
        uuid id PK
        uuid user_id FK
        uuid folder_id FK "nullable"
        uuid job_id FK
        uuid policy_version FK "nullable"
        bigint snapshot_revision "nullable, not FK"
        text status
        timestamptz expires_at
    }
    offlinePackageItems["offline_package_items"] {
        uuid package_id PK, FK
        integer ordinal PK
        uuid card_id FK
        integer card_revision FK
        jsonb progress_snapshot
        jsonb asset_manifest
    }
    cardRevisions["card_revisions"] {
        uuid card_id PK, FK
        integer revision PK
    }
    syncSnapshots["sync_snapshots"] {
        uuid id PK
        uuid user_id FK
        bigint base_revision "not FK"
        jsonb payload
        timestamptz expires_at
    }
    changeLog["change_log"] {
        bigint revision PK
        integer ordinal PK
        uuid audience_user_id FK "nullable: public change"
        text entity_type
        uuid entity_id "polymorphic reference, not FK"
        text operation
    }

    users ||..o{ offlinePackages : prepares
    folders |o..o{ offlinePackages : source_folder
    jobs ||..o{ offlinePackages : builds
    learningPolicies |o..o{ offlinePackages : fixed_policy
    offlinePackages ||--o{ offlinePackageItems : manifest
    cardRevisions ||..o{ offlinePackageItems : fixed_content
    users ||..o{ syncSnapshots : snapshots
    users |o..o{ changeLog : private_audience
```

Дополнительно `offline_package_items.card_id → cards.id`; пара `(card_id, card_revision)` ссылается на ревизию. Пара `(package_id, card_id)` уникальна. Пакет может быть пустым или ещё строиться, поэтому нижняя граница элементов — ноль. `jobs → offline_packages` показывает кардинальность описанного FK: ограничение UNIQUE на job_id в проекте БД пока не задано.

Номера `base_revision` и `snapshot_revision` — границы снимков, не ссылки на существующую строку журнала. В `change_log.entity_id` может находиться ID уже удалённого объекта: обычный FK мешал бы хранить tombstone.

## 9. Независимые технические таблицы

```mermaid
erDiagram
    syncClock["sync_clock"] {
        smallint id PK
        bigint revision
    }
    idempotencyKeys["idempotency_keys"] {
        text scope PK
        text key PK
        text request_hash
        text state
        uuid resource_id "nullable, polymorphic reference"
        timestamptz expires_at
    }
    schemaMigrations["schema_migrations"] {
        text version PK
        text checksum
        timestamptz applied_at
    }
```

`sync_clock` — одна изменяемая строка счётчика, а не родитель всех записей журнала. `idempotency_keys.scope` кодирует область запроса, но не является FK на пользователя. Эти таблицы не соединяются фиктивными отношениями только ради общего рисунка.

## Границы диаграммы

Схема охватывает серверные таблицы из разделов 2–9 [описания БД](database.md). Локальные кеши SQLite/IndexedDB, S3-файлы и технический реестр внешних сервисов не являются таблицами этой PostgreSQL-схемы.

ER-диаграмма не выражает проверки прав, одинакового языка, отсутствия циклов, допустимых переходов статусов и порядка транзакций. Эти правила, а также nullable-поля, не показанные среди сокращённого набора атрибутов, остаются в описании БД. Новые сущности или продуктовые правила этим документом не вводятся.
