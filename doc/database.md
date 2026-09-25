# База данных Kompotik

[ER-диаграмма: общий вид и связи таблиц по областям](database-er.md).

Статус: проект логической схемы PostgreSQL, не SQL-миграция. Дата: 25 сентября 2026 года. Имена таблиц и детализация полей предлагаются для обсуждения.

Основа: [архитектура v0.5](architecture.md), [функциональные требования](https://docs.google.com/document/d/13DhyOBaIGAJRgvpVTTC32wxpCtCYCFFowQn9RIn3KDU/edit), [нефункциональные требования](https://docs.google.com/document/d/1c5UXA0RfIJ75Z2sQTfz0E-spPfpsIqhiP_baDNj83vI/edit). Поздние уточнения из архитектуры имеют приоритет. Контракты: [API](api.md), [сервисы](service-interfaces.md), [экраны](user-interface.md).

## 1. Схема связей и правила записи

```text
languages --< lexical_entries --< senses --< cards --< card_revisions
                                                  |          |
                                                  |          +--< card_article_forms
                                                  |          +--< card_pronunciations --> media_assets
                                                  |          +--< card_revision_media --> media_assets
users --< folders --< folder_cards >---------------+
  |          |
  |          +--> folders (родитель)
  +--< user_card_progress >------------------------+
  +--< lesson_results --< lesson_result_cards --< exercise_results
  +--< learning_events
  +--< imports --< import_candidates --< candidate_occurrences
  +--< jobs --< job_attempts

catalog_collections --< catalog_collection_cards --> cards
```

`--<` означает «один ко многим», `>--` — «многие к одному». Схема показывает главные связи; служебные таблицы описаны ниже.

Основной сервер — единственный владелец предметных данных. Файлы находятся в S3/MinIO; в БД — метаданные и связи. Основной HTTP-процесс и worker используют одну БД. Сервисы генерации не записывают непосредственно в карточки и прогресс.

Обозначения: `id` — `uuid` PK, если не сказано иное; `?` — nullable; `FK` — внешний ключ. Время — `timestamptz`, даты статистики — `date`; durations — целые секунды. По умолчанию поля обязательны. Все перечисленные `*_id` со ссылкой на предметную таблицу — FK, кроме явно названных внешних/локальных идентификаторов. `created_at` — серверное время создания, `updated_at` — последнее изменение, `version bigint` — версия оптимистической блокировки. Каждая строка ниже перечисляет поля таблицы, без неявного требования добавлять общие поля везде.

## 2. Аккаунты и настройки

| Таблица | Поля | Назначение и ограничения |
| --- | --- | --- |
| `users` | `id`, `email text`, `normalized_email text`, `password_hash text`, `status text`, `created_at`, `updated_at` | Аккаунт. Уникальный нормализованный email; правила нормализации фиксируются отдельно, без эвристик удаления точек у провайдеров. Пароль только в виде стойкого хеша. |
| `user_settings` | `user_id uuid PK/FK users`, `interface_language text`, `theme text`, `timezone text`, `cards_per_lesson smallint`, `writing_enabled boolean`, `intervals_seconds integer[]`, `failure_interval_seconds integer`, `preferred_audio_locale text?`, `effective_policy_version uuid FK learning_policies`, `version`, `updated_at` | Настройки: язык UI ru/en/fr, светлая/тёмная тема; размер урока проверяется по допустимым границам политики (сейчас 1–8). Интервалы непустые, положительные, возрастающие. |
| `devices` | `id` (клиентский UUID), `user_id FK users`, `platform text`, `name text?`, `last_result_sequence bigint`, `last_result_id uuid? FK lesson_results`, `created_at`, `last_seen_at timestamptz` | Установка, привязанная к аккаунту; последовательность принятых уроков. При смене аккаунта используется отдельная регистрация устройства/ID. ID не является секретом. |
| `auth_sessions` | `id`, `user_id FK users`, `device_id FK devices`, `kind text`, `credential_hash text`, `refresh_family_id uuid?`, `replaced_by_id uuid? FK auth_sessions`, `expires_at timestamptz`, `revoked_at timestamptz?`, `created_at` | Web-сессии или семейство native refresh tokens; значения токенов в открытом виде не хранятся. Это не учебные сессии. |

Схему одноразовых email-токенов добавляем при определении подтверждения/восстановления. Короткоживущий access token не требует отдельной строки на каждый запрос.

## 3. Языки, словарь и происхождение

| Таблица | Поля | Назначение и ограничения |
| --- | --- | --- |
| `languages` | `id`, `code text`, `display_name text`, `status text`, `catalog_version bigint`, `created_at`, `updated_at` | Коды уникальны. `status=preparing|published|disabled`; ru может быть языком перевода без собственного учебного корня. |
| `content_sources` | `id`, `provider text`, `dataset_version text`, `url text?`, `license_name text?`, `license_url text?`, `attribution text?`, `imported_at timestamptz` | Происхождение словаря/медиа. Отсутствие сведений о правах требует проверки до публикации, не означает разрешение. |
| `lexical_entries` | `id`, `language_id FK languages`, `lemma text`, `normalized_lemma text`, `kind text`, `part_of_speech text?`, `visibility text`, `owner_user_id uuid? FK users`, `created_at`, `updated_at` | Слово либо выражение (`word|expression`). Одинаковое написание не является уникальным ключом: возможны омонимы. Общая запись имеет owner=null, приватная — владельца. |
| `senses` | `id`, `lexical_entry_id FK lexical_entries`, `definition text`, `definition_language_id FK languages`, `domain text?`, `visibility text`, `owner_user_id uuid? FK users`, `created_at`, `updated_at` | Конкретное значение. Приватный смысл не публикуется автоматически. Общий смысл не может принадлежать приватной лексической записи. |
| `sense_sources` | `sense_id FK senses`, `source_id FK content_sources`, `external_entry_id text`, `external_sense_id text`, `source_url text?` | PK `(source_id, external_sense_id)` при гарантированной стабильности ID источника; иначе импортёр сначала формирует составной стабильный ID. Несколько источников могут ссылаться на один смысл. |

Общая карточка не может ссылаться на приватный смысл. Собственная карточка может ссылаться на общий либо собственный приватный смысл. Язык карточки выводится из lexical entry и дублируется для проверок/индексов с контролем согласованности. Другие значения слова находятся через `lexical_entry_id`. Таблица произвольных синонимов пока не вводится.

## 4. Карточки, ревизии и медиа

| Таблица | Поля | Назначение и ограничения |
| --- | --- | --- |
| `cards` | `id`, `sense_id FK senses`, `language_id FK languages`, `translation_language_id FK languages`, `visibility text`, `owner_user_id uuid? FK users`, `state text`, `published_revision integer?`, `draft_revision integer?`, `version`, `created_at`, `updated_at`, `archived_at timestamptz?` | Устойчивая идентичность карточки и указатели версий. `state=draft|generating|ready|failed`. При новой черновой версии существующая опубликованная продолжает работать. Частичный UNIQUE `(sense_id, translation_language_id)` для неархивных общих карточек. |
| `card_revisions` | `card_id FK cards`, `revision integer`, `expression text`, `translation text`, `definition text?`, `examples jsonb`, `morphology jsonb`, `accepted_forms jsonb`, `dictionary_links jsonb`, `state text`, `created_at`, `published_at timestamptz?` | PK `(card_id,revision)`. Полный снимок текста, форм и грамматики. Опубликованная версия неизменяема. `state=draft|generating|published|failed|superseded`. Архивирование карточки не удаляет ревизии. |
| `card_revision_sources` | `card_id`, `revision`, `source_id FK content_sources`, `source_url text?`, `note text?` | PK `(card_id,revision,source_id)`, составной FK на ревизию. Атрибуция конкретного снимка контента. |
| `card_article_forms` | `id`, `card_id`, `revision`, `article text`, `written_form text`, `grammatical_note text?`, `position integer` | Составной FK на ревизию. Список допустимых вариантов «артикль + форма», а не выбор единственного артикля. UNIQUE `(card_id,revision,position)`. |
| `card_pronunciations` | `id`, `card_id`, `revision`, `form_id uuid? FK card_article_forms`, `locale text`, `ipa text?`, `asset_id uuid? FK media_assets`, `source_id uuid? FK content_sources`, `is_default boolean`, `state text` | Составной FK на ревизию. `form_id=null` — основная форма без отдельного артикля. Форма обязана принадлежать той же ревизии. `state=pending|ready|failed`. Не более одного default для каждой формы ревизии, включая null. |
| `media_assets` | `id`, `kind text`, `storage_key text`, `mime_type text`, `sha256 text`, `byte_size bigint`, `width integer?`, `height integer?`, `duration_ms integer?`, `visibility text`, `owner_user_id uuid? FK users`, `state text`, `source_id uuid? FK content_sources`, `operation_id uuid?`, `created_at`, `deleted_at timestamptz?` | Уникальный ключ S3, неизменяемый после готовности файл. `kind=image|audio`, `state=staged|ready|failed|deleted`. `operation_id` — ID внешней операции, не FK на job. Видимость файла не шире разрешённого контента. |
| `card_revision_media` | `card_id`, `revision`, `role text`, `asset_id FK media_assets` | PK `(card_id,revision,role)`; составной FK на ревизию. В первой версии `role=image`. Аудио связано через произношения. |

`examples` — массив `{text, translation?}`; `morphology` — проверяемый по языку объект с частью речи, родом, числом, ограничениями и формами; `accepted_forms` — допустимые ответы с явно отделяемыми артиклями; `dictionary_links` — проверенные URL источников. JSONB выбран для небольших структур внутри снимка, а не для основных связей и владельцев. Схемы JSON версионируются вместе с форматом ревизии/API.

`published_revision` и `draft_revision` ссылаются составными FK `(id, revision)` на `card_revisions`; вставка карточки начинается с null, затем добавляется ревизия. Новое изображение/произношение опубликованной карточки создаёт новую ревизию, старый файл не перезаписывается. При правке самого смысла создаётся новый `card_id`; прогресс не переносится автоматически.

Изображение обязательно для публикации. Обязательность аудио остаётся продуктовым вопросом; выбранная политика готовности проверяется одной функцией публикации, а не разными условиями клиента/worker. Повтор аудио для нескольких смыслов может переиспользовать файл только при одинаковой форме, произношении и разрешённой области доступа.

## 5. Библиотека и готовые подборки

| Таблица | Поля | Назначение и ограничения |
| --- | --- | --- |
| `folders` | `id`, `user_id FK users`, `language_id FK languages`, `parent_id uuid? FK folders`, `is_root boolean`, `name text`, `source_label text?`, `origin_collection_id uuid? FK catalog_collections`, `origin_collection_version bigint?`, `version`, `created_at`, `updated_at` | Папка-набор. UNIQUE `(user_id,language_id)` WHERE is_root. Только корень имеет parent=null. Родитель того же владельца/языка. |
| `folder_cards` | `folder_id FK folders`, `card_id FK cards`, `added_at timestamptz` | PK `(folder_id,card_id)`. Только карточка языка папки и доступная владельцу. Прогресс здесь не хранится. |
| `catalog_collections` | `id`, `language_id FK languages`, `title text`, `description text`, `goal text?`, `level text?`, `status text`, `version`, `created_at`, `updated_at` | Редакторская подборка; `draft|published|archived`. Состав при копировании читается вместе с версией под блокировкой. |
| `catalog_collection_cards` | `collection_id FK catalog_collections`, `card_id FK cards`, `position integer` | PK `(collection_id,card_id)`. Только общие готовые карточки того же языка. Изменение состава увеличивает версию подборки. |

Счётчики и завершённость папок — вычисляемая проекция по уникальным `card_id` всего поддерева. Постоянного поля `folder.learned` нет. Удаление поддерева удаляет его `folder_cards`, но не `cards`, `user_card_progress` и историю. Общая библиотека пользователя — объединение связей его папок; собственная карточка без папки сохраняется среди его карточек, но не входит в счётчик библиотеки.

Для перемещения дерева одной проверки до транзакции недостаточно. Предложение: сериализовать изменения дерева одного пользователя транзакционной блокировкой, затем проверить предков, язык и владельца. Составные FK обеспечивают совпадение владельца/языка родителя; триггер/предметная команда проверяет отсутствие цикла и допустимость связи с карточкой.

## 6. Политики, прогресс и история

| Таблица | Поля | Назначение и ограничения |
| --- | --- | --- |
| `learning_policies` | `id` (значение `policy_version` в API), `user_id uuid? FK users`, `algorithm_version text`, `config jsonb`, `config_hash text`, `created_at` | Неизменяемая эффективная политика: веса, порог, интервалы, размер урока, написание, нормализация, правила замены упражнений. null owner — общая базовая политика. |
| `user_card_progress` | `user_id FK users`, `card_id FK cards`, `state text`, `score smallint`, `interval_index integer`, `review_count integer`, `learning_cycle_id uuid`, `schedule_version bigint`, `review_occurrence_id uuid?`, `next_review_at timestamptz?`, `last_result_at timestamptz?`, `last_applied_result_id uuid? FK lesson_results`, `learned_at timestamptz?`, `learned_source text?`, `policy_version FK learning_policies`, `version`, `updated_at` | PK `(user_id,card_id)`. `state=new|learning|reviewing|learned`, score 0–100. learned ⇒ срок/occurrence=null. review_count относится к текущему циклу, полная история хранится отдельно. |
| `lesson_results` | `id` (клиентский `result_id`), `user_id FK users`, `lesson_id uuid` (локальный), `device_id FK devices`, `device_sequence bigint`, `previous_result_id uuid? FK lesson_results`, `schema_version integer`, `language_id FK languages`, `mode text`, `started_at timestamptz`, `completed_at timestamptz`, `timezone text`, `activity_date date`, `received_at timestamptz`, `policy_version FK learning_policies`, `payload_hash text`, `payload jsonb`, `time_quality text` | Только принятые завершённые уроки. UNIQUE `(user_id,lesson_id)` и `(user_id,device_id,device_sequence)`. Неизменяемое исходное тело и нормализованная дата статистики. |
| `lesson_result_cards` | `result_id FK lesson_results`, `card_id FK cards`, `card_revision integer`, `learning_cycle_id uuid`, `base_progress_version bigint`, `base_result_id uuid? FK lesson_results`, `base_schedule_version bigint?`, `review_occurrence_id uuid?`, `completed boolean`, `exercise_plan jsonb`, `application text`, `reason text?`, `progress_before jsonb`, `progress_after jsonb` | PK `(result_id,card_id)`, FK `(card_id,card_revision)` на ревизию. `application=applied|history_only`; сохраняет решение сервера, включая устаревший цикл. |
| `exercise_results` | `result_id`, `card_id`, `type text`, `first_answer_correct boolean`, `hint_used boolean`, `attempts integer`, `errors integer`, `pronunciation_id uuid? FK card_pronunciations`, `audio_asset_id uuid? FK media_assets`, `form_id uuid? FK card_article_forms`, `substitution_for text?` | PK `(result_id,card_id,type)`, составной FK на `lesson_result_cards`. По типу один агрегированный результат первого оцениваемого ответа; повтор типа не даёт второй вес. attempts ≥ 1, errors от 0 до attempts. |
| `learning_events` | `id`, `user_id FK users`, `card_id FK cards`, `type text`, `result_id uuid? FK lesson_results`, `learning_cycle_id uuid`, `occurred_at timestamptz`, `received_at timestamptz`, `expected_progress_version bigint?`, `before_state jsonb`, `after_state jsonb`, `payload_hash text?` | Неизменяемый журнал: `lesson_applied`, `known`, `progress_reset`. Для ручных действий ID задаёт клиент и служит дедупликации. Исторические отчёты без применения не создают фиктивный переход. |
| `review_credits` | `user_id`, `card_id`, `learning_cycle_id uuid`, `review_occurrence_id uuid`, `result_id FK lesson_results`, `credited_at timestamptz` | PK `(user_id,card_id,learning_cycle_id,review_occurrence_id)`, FK `(user_id,card_id)` на прогресс. Один срок повторения нельзя зачесть дважды. |

Новая карточка может не иметь строки прогресса: API возвращает виртуальный `new`, версию 0 и детерминированный исходный UUID цикла. Первое изменение материализует эту запись атомарным upsert. Сброс назначает новый UUID; его нельзя получить повторной материализацией исходного состояния. ID цикла не является FK на отдельную учебную сессию.

Состояние `learning` может присутствовать локально в начатом уроке, хотя незавершённый урок не отправляется на сервер. Не требуется отдельная серверная запись старта ради этого статуса. Локальный клиент накладывает черновик на подтверждённый прогресс.

Принятие урока, фиксация ключей уникальности, `review_credits`, записи истории, изменения прогресса и запись журнала синхронизации происходят одной транзакцией. Строки прогресса блокируются по стабильному порядку ID. Внешние запросы внутри транзакции не выполняются.

При конкурентных первичных уроках проверяется состояние цикла под блокировкой: после первого завершённого первичного урока второй сохраняется только исторически. Для повторений дополнительно действует уникальность `review_credits`. Это соответствует предложенной политике конфликтов в [API](api.md), не требует таблицы серверных учебных сессий.

Сброс: score=0, state=new, interval_index=0, review_count=0, новый цикл, новый schedule_version; срок, occurrence, last_result_at, last_applied_result_id и текущие learned-поля очищаются. История и старые зачёты остаются. Поздний урок старого цикла получает `history_only`, а не отменяет сброс.

Статистика строится по этим таблицам; отдельная таблица счётчиков на старте не требуется. `activity_date` фиксируется по поясу занятия после проверки времени. Активность учитывает принятые завершённые отчёты, включая исторические, с DISTINCT по карточке/дню/режиму; переход в learned учитывается только по применённым событиям. «Впервые выучена» — первое такое событие за всю историю карточки пользователя, не за каждый сброшенный цикл. Ручное `known` не считается уроком.

## 7. Исходные материалы и результаты анализа

| Таблица | Поля | Назначение и ограничения |
| --- | --- | --- |
| `imports` | `id`, `user_id FK users`, `language_id FK languages`, `title text`, `source_type text`, `source_text text?`, `source_hash text`, `status text`, `analysis_version text?`, `catalog_version bigint?`, `active_job_id uuid? FK jobs`, `version`, `applied_folder_id uuid? FK folders`, `applied_at timestamptz?`, `apply_receipt jsonb?`, `created_at`, `updated_at`, `source_expires_at timestamptz?` | Приватный материал и жизненный цикл `queued|analyzing|awaiting_selection|applied|failed|cancelled`. Для предлагаемого первого входа source_type=text. source_text может очищаться по будущей политике хранения. Квитанция применения остаётся для идемпотентного ответа. |
| `import_candidates` | `id`, `import_id FK imports`, `analysis_item_key text`, `lemma text`, `kind text`, `part_of_speech text?`, `proposed_definition text?`, `proposed_translation text?`, `resolution_state text`, `selected boolean`, `selected_sense_id uuid? FK senses`, `selected_card_id uuid? FK cards`, `new_sense jsonb?`, `created_card_id uuid? FK cards`, `version` | UNIQUE `(import_id,analysis_item_key)`. `matched|ambiguous|unmatched|confirmed_new`. Выбранный старый смысл и new_sense взаимоисключающие. card обязан соответствовать выбранному смыслу. |
| `candidate_sense_options` | `candidate_id FK import_candidates`, `sense_id FK senses`, `rank integer`, `confidence numeric?`, `explanation text?` | PK `(candidate_id,sense_id)`. Варианты сопоставления; confidence не объявляется калиброванной вероятностью. |
| `candidate_occurrences` | `id`, `candidate_id FK import_candidates`, `segments jsonb`, `surface text`, `context_start integer`, `context_end integer`, `context text` | Вхождения конкретного смысла. segments — упорядоченные диапазоны `[start,end)` Unicode code points исходного текста; допускаются разорванные выражения. Контекст приватен. |

Нельзя объединять кандидатов только по `lemma`: одно слово в тексте может иметь разные смыслы. Нельзя считать один словарный смысл одинаковым с другим только из-за перевода. Изменение кандидата блокирует родительский импорт и повышает его версию, чтобы отбор и применение были согласованы. Применение повторно использует выбранные карточки, создаёт недостающие приватные и задания; исходный текст не попадает в общий каталог.

## 8. Очередь и операции сервисов

| Таблица | Поля | Назначение и ограничения |
| --- | --- | --- |
| `jobs` | `id`, `user_id uuid? FK users`, `parent_job_id uuid? FK jobs`, `kind text`, `resource_type text`, `resource_id uuid`, `target_revision integer?`, `payload_version integer`, `payload jsonb`, `status text`, `stage text?`, `attempt_count integer`, `max_attempts integer`, `available_at timestamptz`, `lease_until timestamptz?`, `attempt_token uuid?`, `cancel_requested_at timestamptz?`, `result jsonb?`, `error_code text?`, `created_at`, `updated_at` | Очередь PostgreSQL. `pending|running|succeeded|failed|cancelled`. resource_id — полиморфная ссылка, проверяемая предметной командой; не фиктивный SQL FK. Системное наполнение может иметь user=null. |
| `job_attempts` | `id`, `job_id FK jobs`, `attempt_number integer`, `attempt_token uuid`, `service text`, `operation_id uuid`, `request_hash text`, `status text`, `started_at timestamptz`, `finished_at timestamptz?`, `error_code text?`, `provider_request_id text?`, `estimated_cost numeric?`, `actual_cost numeric?`, `currency text?` | UNIQUE `(job_id,attempt_number)`. Попытки и расходы. Несколько транспортных повторов сохраняют один operation_id; осознанная новая генерация получает другой. |

Примеры видов задания: `analyze_text`, `prepare_card`, `generate_image`, `generate_audio`, `build_offline_package`. Оркестрация принадлежит worker основного приложения. Поле result содержит компактную типизированную квитанцию, большие данные анализа нормализуются в таблицы кандидатов. Секретов провайдера в payload нет.

Worker получает аренду короткой транзакцией, выполняет внешнюю работу без соединения БД, фиксирует ответ только при совпадении актуального `attempt_token`. Истёкшая аренда допускает повторное исполнение, поэтому нужны идемпотентные операции. Отмена проверяется до публикации результата; запоздавший файл не становится видимым автоматически.

## 9. Доставка изменений и временные снимки

| Таблица | Поля | Назначение и ограничения |
| --- | --- | --- |
| `sync_clock` | `id smallint PK`, `revision bigint` | Одна строка. Предлагаемый простой механизм порядка: каждая транзакция синхронизируемых изменений вначале блокирует её и увеличивает revision, удерживая до commit. Rollback откатывает счётчик. Это сериализует записи, приемлемость проверяется нагрузкой. |
| `change_log` | `revision bigint`, `ordinal integer`, `audience_user_id uuid? FK users`, `entity_type text`, `entity_id uuid`, `operation text`, `entity_version bigint?`, `payload jsonb`, `created_at` | PK `(revision,ordinal)`. upsert или delete/tombstone. null audience — общедоступное изменение; private только с user. Курсор включает обе части и область доступа. |
| `sync_snapshots` | `id`, `user_id FK users`, `base_revision bigint`, `payload jsonb`, `created_at`, `expires_at timestamptz` | Временный фиксированный полный снимок для постраничной начальной загрузки. Для небольшой библиотеки материализуется целиком; при росте — отдельные строки элементов. |
| `offline_packages` | `id`, `user_id FK users`, `folder_id uuid? FK folders`, `snapshot_revision bigint?`, `status text`, `policy_version uuid? FK learning_policies`, `requested_audio_locales text[]`, `byte_size bigint?`, `job_id FK jobs`, `created_at`, `expires_at timestamptz` | Временный снимок поддерева: `pending|building|ready|failed`. Не отражает фактическую готовность устройства. При удалении папки FK обнуляется, новые скачивания пакета запрещаются предметной проверкой. |
| `offline_package_items` | `package_id FK offline_packages`, `ordinal integer`, `card_id FK cards`, `card_revision integer`, `progress_snapshot jsonb`, `content_snapshot jsonb`, `asset_manifest jsonb` | PK `(package_id,ordinal)`, UNIQUE `(package_id,card_id)`, FK ревизии. Manifest: ID, hash, размер и назначение медиа; без подписанных URL. |
| `idempotency_keys` | `scope text`, `key text`, `request_hash text`, `state text`, `response_status integer?`, `response_body jsonb?`, `resource_id uuid?`, `created_at`, `expires_at timestamptz` | PK `(scope,key)`, где scope включает пользователя/маршрут/метод либо анонимную регистрацию. Только обычные команды; уроки и события имеют собственные постоянные ключи. |
| `schema_migrations` | `version text PK`, `checksum text`, `applied_at timestamptz` | Учёт применённых SQL-миграций. Одна миграция — один неизменяемый файл. |

Глобальная строка `sync_clock` — конкретное новое предложение, а не требование использовать глобальную блокировку навсегда. Порядок блокировок: сначала clock, затем импорт/дерево/карточки в стабильном порядке. Долгие расчёты и внешние вызовы делаются до короткой фиксирующей транзакции. Полный снимок читается на одном согласованном MVCC-снимке вместе с clock; страничные ответы берутся из материализованного payload.

Курсор продвигается по просмотренным записям, включая отфильтрованные чужие изменения, не раскрывая их. Для общей карточки клиенту передаётся изменение только если карточка относится к его синхронизируемой библиотеке; добавление новой связи обязательно сопровождается актуальным снимком карточки. Это предотвращает потерю обновления, которое было до добавления карточки в библиотеку.

Изменения и tombstone создаются в одной транзакции с предметной записью. Удалённые папки/связи можно удалить физически, оставив tombstone на согласованный срок; карточки с учебной историей архивируются. Устаревший курсор требует полной загрузки. Сроки журналов, snapshots и медиаверсий задаются совместно с пределом офлайна.

## 10. Индексы и контроль целостности

Кроме PK/UNIQUE и индексов FK, нужны:

| Запрос | Индекс/ограничение |
| --- | --- |
| Повторения пользователя | `user_card_progress(user_id,next_review_at,card_id)` WHERE state='reviewing'. |
| Дерево и обратные связи | `folders(user_id,parent_id,id)`, `folder_cards(card_id,folder_id)`. |
| Слова и смыслы | `lexical_entries(language_id,normalized_lemma,id)`, `senses(lexical_entry_id,id)`, `cards(sense_id,visibility,owner_user_id)`. |
| История по дням | `lesson_results(user_id,activity_date,mode,id)`, `learning_events(user_id,card_id,occurred_at,id)`. |
| Выбор заданий | `jobs(available_at,id)` WHERE status='pending'; `jobs(lease_until)` WHERE status='running'. |
| Импорты | `imports(user_id,created_at,id)`, `import_candidates(import_id,resolution_state,id)`. |
| Изменения | `change_log(audience_user_id,revision,ordinal)`. |

Cross-table правила языка, приватности, циклов папок и соответствия произношения ревизии нельзя выразить произвольным CHECK с чтением других таблиц. Используются составные FK где возможно, транзакционные команды и ограниченные constraint triggers. Проверки доступа всегда выполняет сервер, даже если ID известен. Денормализованные поля сверяются при каждой записи.

## 11. Локальная база клиента

Это отдельное хранилище SQLite на Android и IndexedDB/Cache Storage в браузере. Оно не является копией всей серверной БД.

| Локальная запись | Поля/содержание |
| --- | --- |
| `account_state` | account ID, последний sync cursor, server-time offset, версии схем/политик. Секреты native — в защищённом системном хранилище. |
| `cached_cards`, `cached_progress`, `cached_folders` | Серверные снимки с версиями и областью аккаунта. Локальный предварительный прогресс хранится отдельно от подтверждённого. |
| `lesson_drafts` | lesson ID, язык, режим, снимок карточек/политики, план заданий, ответы, текущая позиция, циклы и основания прогресса. |
| `result_outbox` | result ID, device sequence, неизменяемый JSON отчёта, состояние отправки, число попыток, следующая попытка, последняя ошибка. |
| `local_packages`, `cached_assets` | папка/manifest, перечень обязательных файлов, asset ID, hash, локальное местоположение, размер, готовность и ссылки использования. |

Каждый ответ сохраняет черновик; завершение одной локальной транзакцией записывает отчёт, предварительный прогресс и outbox. Обновление приложения не очищает эти записи. Скачанный файл можно удалять только если его не держит пакет или незавершённый урок. Неотправленные результаты не удаляются из-за истечения серверного снимка.

## 12. Открытые решения

До миграций нужно утвердить сроки хранения, пределы размеров JSON/импорта, обязательность аудио и правила конфликтующих цепочек. Здесь предложена логическая структура, SQL-типы/индексы следует проверить реальными запросами. Публичный API аккаунтного удаления и окончательный алгоритм очистки приватных данных требуют отдельного сценария; каскадное удаление истории сейчас не подразумевается.
