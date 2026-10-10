## План по реализации пайплайна
Пайплайн предназначен для выведения правил ревью из существующих комментариев к PR

## Масштаб «сотни комментариев»

После группировки в треды и отсева ботов/шума у вас останется порядка 100–300 тредов. Из этого следует:

- **Никакой эмбеддинг-кластеризации** (UMAP/HDBSCAN) — на таком объёме она даёт артефакты. Слияние правил делает LLM за один проход, всё влезает в контекст.
- **Ожидайте 5–12 правил на выходе, не 25.** Если выживет три правила с прочными основаниями — это нормальный результат, не провал.
- **Воспроизводимость вместо статистики.** Индукцию правил прогоняем трижды с перемешанными батчами; правило живёт, только если возникло минимум в двух прогонах.
- **Holdout будет тонким** (30–50 тредов). Валидация на шаге 7 — это sanity-check в абсолютных числах, а не метрика с процентами.

Пороги консервативности, зашитые в код между шагами: правило требует $\geq 3$ разных тредов, $\geq 2$ разных PR, `confidence` $\geq 0.6$, воспроизведение в $\geq 2$ из 3 прогонов. Правила с одним ревьюером-источником не отбрасываются, но помечаются и идут на ручную курацию в первую очередь.

## Структура проекта

```
review-rules/
├── PIPELINE.md
├── taxonomy.yaml
├── schemas/            # JSON Schema на каждый *.jsonl
├── prompts/            # p1_normalize.txt … p7_review.txt
├── scripts/
│   ├── extract.py      # шаг 0
│   ├── prepare.py      # шаг 0b: треды, признаки, holdout
│   ├── run_llm.py      # батчинг, retry, валидация JSON
│   ├── filter_rules.py # механические пороги после шага 2
│   └── evaluate.py     # матчинг на шаге 7
└── data/
    ├── raw/comments.jsonl
    ├── work/threads.jsonl threads_llm.jsonl claims.jsonl rules_run{1,2,3}.jsonl
    ├── work/rules_filtered.jsonl canonical.jsonl canonical_curated.jsonl
    ├── work/priority.jsonl rubric.md
    └── eval/threads.jsonl findings.jsonl report.json
```

Общий системный префикс для всех LLM-шагов 1–7:

```text
Ты — компонент автоматического пайплайна. Отвечай только валидным JSON по указанной схеме, без markdown и пояснений. Не придумывай данные, которых нет во входе. При неопределённости используй предусмотренные схемой значения (null, needs_context, insufficient_data), а не догадки. Если во входе есть поле, не описанное в задании, — игнорируй его.
```

---

## Шаг 0 — выгрузка (промпт для агента-кодера)

```text
Напиши extract.py на Python 3.11+ и httpx: выгрузка inline-комментариев code review из GitHub через GraphQL.

ВХОД: GITHUB_TOKEN из окружения; repos.txt со строками owner/name; --since/--until (фильтр по mergedAt); --out, --state, --max-prs, --concurrency (по умолчанию 4), --verbose.

ИСТОЧНИК — только GraphQL https://api.github.com/graphql. REST не использовать: он не отдаёт isResolved/isOutdated.

Запросы:
1. search(query:"repo:O/N is:pr is:merged merged:SINCE..UNTIL", type:ISSUE, first:50, after:$cursor) — перечисление PR, пагинация по cursor.
2. Для каждого PR: number, mergedAt, additions, deletions, changedFiles, author{login}, reviewDecision,
   reviewThreads(first:100, after:$cursor){ nodes{ id isResolved isOutdated path line originalLine startLine diffSide
     comments(first:100){ nodes{ id databaseId body createdAt author{login} authorAssociation diffHunk originalCommit{oid} commit{oid} replyTo{id} } } } }
   reviews(first:100){ nodes{ state submittedAt author{login} } }  — нужны ТОЛЬКО для вычисления blocked_merge, тело ревью не сохранять.
   commits(first:100){ nodes{ commit{ committedDate oid } } }
   Все вложенные коллекции пагинируй, если hasNextPage.

ВЫХОД: один файл comments.jsonl, строка = inline-комментарий:
{"id","repo","pr_id","pr_merged_at","pr_author","thread_id","path","line_range":{"start","end","side"},
 "diff_hunk","comment_body","author","author_association","created_at","is_reply",
 "thread_resolved","thread_outdated","comment_commit_oid","original_commit_oid",
 "pr_size":{"additions","deletions","changed_files"},
 "pr_changes_requested_count","blocked_merge","blocked_merge_reason","schema_version":"1","run_id"}

blocked_merge = true, если после ревью со state=CHANGES_REQUESTED появился коммит с committedDate > submittedAt. Это эвристика, запиши в blocked_merge_reason текст условия. Если ревью CHANGES_REQUESTED нет — false; если данные о коммитах не получены — null.
Любое неизвестное значение — null, не false и не "". Боты и шум НЕ фильтруются на этом шаге.

НАДЁЖНОСТЬ: ретраи с экспоненциальной задержкой на 403/429/5xx, уважение Retry-After, при тексте про secondary rate limit пауза ≥60 с; чтение X-RateLimit-Remaining, пауза при остатке <100; state.json с обработанными pr_id, повторный запуск дописывает без дублей; ошибка одного PR → запись в errors.jsonl, продолжение. Токен нигде не логируется.

СТРУКТУРА: build_pr_list_query(), build_pr_query() отдельными функциями, возвращающими строку GraphQL; dry_run() при --max-prs 3 печатает две записи без записи в файл. Тесты не нужны. README: команды запуска, нужные скоупы PAT (classic: repo; fine-grained: Pull requests Read + Contents Read + Metadata Read с явным перечислением репозиториев), оценка числа запросов.
```

<details>
<summary>Проверка пробного прогона (--max-prs 5)</summary>

1. `diff_hunk` непустой и содержит код, не только строку `@@`.
2. `thread_resolved` встречается и как `true`, и как `false`; если везде `null` — запрос тредов собран неверно.
3. `is_reply=true` ровно у комментариев с `replyTo`.
4. `pr_merged_at` заполнен у всех записей.
5. В файле есть записи от `*[bot]` — фильтр не протёк на шаг 0.

</details>

## Шаг 0b — подготовка (промпт для агента-кодера)

```text
Напиши prepare.py: из data/raw/comments.jsonl собрать треды, признаки и holdout. Сеть не используется.

1. build_threads(): группировка по thread_id, комментарии по created_at.
   Поля треда: thread_id, repo, pr_id, pr_merged_at, path, line_range, diff_hunk (из первого комментария),
   comments:[{id, author_role, is_pr_author, body, created_at}],
   where author_role: "bot" если login оканчивается на "[bot]" или входит в bots.txt; "member" если authorAssociation ∈ {MEMBER, OWNER, COLLABORATOR}; иначе "external".
   is_pr_author = author == pr_author.
   Удали треды, где ВСЕ комментарии от ботов ИЛИ первый комментарий от бота. Остальные боты в середине треда остаются с author_role="bot".
   Удали треды, у которых первый комментарий короче 4 символов после trim.

2. derive_features() на уровне треда, все null-safe:
   n_comments, n_participants (без ботов), has_author_reply, first_reviewer_hash (sha1 логина, 8 символов — для подсчёта разнообразия ревьюеров без раскрытия имён),
   resolved, outdated, code_changed_after = outdated OR any(comment_commit_oid != original_commit_oid) (null если нечего сравнивать),
   resolved_after_fix = resolved AND code_changed_after, resolved_without_fix = resolved AND code_changed_after == false,
   thread_turnaround_hours, days_before_merge, blocked_merge (с PR), pr_changes_requested_count, pr_size.

3. holdout_split(): отсортируй PR по pr_merged_at, последние 15% PR (не тредов!) → data/eval/threads.jsonl, остальные → data/work/threads.jsonl. Запиши границу даты в split.json.

4. make_llm_view(): data/work/threads_llm.jsonl — копия work/threads.jsonl, из которой УДАЛЕНЫ: resolved, outdated, code_changed_after, resolved_after_fix, resolved_without_fix, blocked_merge, pr_changes_requested_count, n_comments, thread_turnaround_hours, days_before_merge, first_reviewer_hash, pr_size, pr_merged_at. Остаются: thread_id, repo, pr_id, path, line_range, diff_hunk, comments (без created_at). Логины авторов заменить на "reviewer_1", "pr_author", "reviewer_2" по порядку появления.

5. Напечатай сводку: тредов всего / после фильтра / в work / в eval; доля null по каждому вычисляемому полю; распределение n_comments; число уникальных first_reviewer_hash. Если уникальных ревьюеров ≤2 — выведи предупреждение «разнообразие ревьюеров низкое, правила будут отражать стиль 1–2 человек».
```

---

## Шаг 1 — нормализация тредов → claims

Батч 20–30 тредов. Вход — `threads_llm.jsonl`.

```text
Тебе дан батч тредов code review. Каждый тред: thread_id, path, line_range, diff_hunk, comments (упорядочены, первый — исходное замечание, остальные — обсуждение; роли: reviewer_N, pr_author, bot).

ЗАДАЧА: для каждого треда выделить утверждения о коде (claims) и классифицировать их.

ПРАВИЛА:
1. Опирайся только на diff_hunk и текст комментариев. Если невозможно понять, о каком коде речь, — needs_context=true, normalized=null.
2. Обычно в треде один claim. Несколько — только если ревьюер явно перечисляет разные проблемы. Реплики pr_author — контекст для понимания, не источник claims.
3. Реплики bot игнорируй полностью.
4. Если в обсуждении ревьюер сам отозвал замечание («а, понял, всё верно») — пометь withdrawn=true, но claim всё равно выдели.
5. Для каждого треда ровно одна запись в выводе, порядок входа сохрани. Если claims нет — пустой список и reason.

КАТЕГОРИИ claim:
- noise: благодарности, "+1", LGTM, эмодзи, реакция без содержания
- style_only: форматирование, именование, импорты, длина строк — всё, что ловит линтер
- question: уточняющий вопрос без утверждения о дефекте
- defect: ошибка, краевой случай, уязвимость, гонка, утечка, нарушенный контракт, неверная логика
- design: структура, API, абстракция, тестируемость, дублирование
- docs: документация, комментарии в коде, сообщения об ошибках для пользователя
- process: не о коде (разбей PR, нет тестов в CI, нужен другой ревьюер)

Для defect/design/docs заполни:
- normalized: одна нейтральная фраза — проверяемое требование к коду, сформулированное как инвариант («при отсутствии X поле Y должно…»). Без обращений и императива.
- evidence: дословная цитата из комментария ревьюера (≤200 символов)
- code_anchor: идентификатор или строка из diff_hunk, к которой относится claim, либо null
- severity_hint: low | medium | high — только по тексту и хунку
- language: язык кода в diff_hunk, если определим, иначе null

ВЫВОД:
{"threads":[{"thread_id":"...","claims":[{"claim_id":"<thread_id>#1","category":"...","normalized":"...","evidence":"...","code_anchor":"...","severity_hint":"...","language":"...","needs_context":false,"withdrawn":false}],"reason":null}]}

Батч:
{{BATCH_JSON}}
```

Код после шага 1: в `claims.jsonl` идут только `category ∈ {defect, design, docs}`, `needs_context=false`, `withdrawn=false`. К каждому claim код дописывает `pr_id` и `first_reviewer_hash` из `threads.jsonl` — модель на шаге 2 их **не** видит.

## Шаг 2 — индукция правил (3 прогона)

Батч 40–60 claims. Три прогона с разным `shuffle_seed`; каждый прогон пишет свой `rules_run{k}.jsonl` с `run_id`. При сотнях тредов один прогон — это 2–5 батчей.

```text
Тебе дан батч claims — нормализованных замечаний code review с цитатами. Поля: claim_id, thread_id, normalized, evidence, code_anchor, category, language.

ЗАДАЧА: вывести атомарные правила код-ревью, подтверждённые несколькими независимыми замечаниями.

ЖЁСТКИЕ ПРАВИЛА:
1. Одно свидетельство = один thread_id. Несколько claims из одного треда — одно свидетельство. Правило требует минимум 3 РАЗНЫХ thread_id. Иначе — в singletons.
2. Одно правило = одна проверка. Запрещено объединять разные требования через «и»/«или».
3. Правило должно быть falsifiable: должен существовать код, который его нарушает. «Код должен быть читаемым» — запрещено.
4. Не обобщай за пределы свидетельств. Если все замечания про отсутствие проверки на None после запроса к ORM — правило про это, а не «валидируй все входы».
5. Если формулировка сводится к предпочтению команды («мы так не делаем», «у нас принято») — scope="local_taste".
6. Если свидетельства относятся к разным языкам/стекам и правило верно для всех — scope="universal". Если верно только для одного стека — укажи его в applies_to.
7. Лучше не выдать правило, чем выдать сомнительное. Если сомневаешься, подтверждают ли свидетельства одно и то же, — не объединяй.

ДЛЯ КАЖДОГО ПРАВИЛА:
- id: kebab-case, по смыслу
- statement: одно предложение, проверяемое на диффе
- axis: ровно одно из {{TAXONOMY_AXES}}
- scope: universal | local_taste
- applies_to: язык/фреймворк или null
- evidence_thread_ids: список thread_id (≥3)
- best_evidence_claim_id: один claim, лучше всего иллюстрирующий правило
- counterexample: короткий фрагмент кода, который правило отвергает, выведенный из свидетельств
- confidence: 0..1 — уверенность, что правило обобщается за пределы батча. Снижай, если свидетельства похожи на одну и ту же проблему в одном месте.

ВЫВОД:
{"rules":[{"id":"...","statement":"...","axis":"...","scope":"...","applies_to":null,"evidence_thread_ids":["..."],"best_evidence_claim_id":"...","counterexample":"...","confidence":0.0}],
 "singletons":[{"thread_ids":["..."],"note":"кратко, о чём"}]}

Claims:
{{BATCH_JSON}}
```

<details>
<summary>Механический фильтр между шагами 2 и 3 (filter_rules.py)</summary>

Код считает по `evidence_thread_ids` через `threads.jsonl`: `n_threads`, `n_prs`, `n_reviewers`, `n_repos`. Оставляет правило, если:

- `n_threads` $\geq 3$ и `n_prs` $\geq 2$;
- `confidence` $\geq 0.6$;
- все `evidence_thread_ids` реально существуют в `claims.jsonl` (модели свойственно дописывать id).

Правила со `scope=local_taste` не отбрасываются, а уходят в `rules_local.jsonl` — они понадобятся на шаге 6 в отдельной секции. Правила с `n_reviewers = 1` получают флаг `single_reviewer=true`.

Выход: объединённый `rules_filtered.jsonl` из всех трёх прогонов, у каждого правила сохранён `run_id`.

</details>

## Шаг 3 — слияние прогонов и дедупликация

Один вызов на всё (при сотнях тредов это 20–60 правил). На вход — `id, run_id, statement, axis, scope, applies_to, evidence_thread_ids, counterexample`.

```text
Тебе даны правила код-ревью, выведенные тремя независимыми прогонами (run_id) из одних и тех же данных. Одно и то же требование может быть сформулировано по-разному в разных прогонах.

ЗАДАЧА: сгруппировать правила в канонические кластеры.

ПРАВИЛА:
1. Объединяй правила, только если ОДНО И ТО ЖЕ изменение кода нарушает оба. Общая тема («обработка ошибок») — не основание. Таймауты и ретраи — разные правила.
2. Не объединяй правила про разные библиотеки или стеки, если applies_to различается.
3. Пересечение evidence_thread_ids — сильный сигнал в пользу слияния, пустое пересечение — сигнал против, но не запрет.
4. Правило без пары — кластер из одного элемента. Не приписывай его никуда силой.
5. Канонический statement должен покрывать всех членов кластера и не быть шире их. Если это невозможно — не сливай.
6. Если у кластера члены противоречат друг другу — пометь conflict=true и опиши противоречие.

ВЫВОД:
{"clusters":[{"cluster_id":"...","statement":"...","axis":"...","scope":"...","applies_to":null,"member_rule_ids":["..."],"counterexample":"...","conflict":false,"conflict_note":null,"merge_doubt":null}]}

merge_doubt — одна строка, если слияние спорное; иначе null.

Правила:
{{RULES_JSON}}
```

Код после шага 3 дописывает каждому кластеру: `runs_covered` (сколько из 3 прогонов дали члена), объединённые `evidence_thread_ids`, `n_threads`, `n_prs`, `n_reviewers`. **Кластеры с `runs_covered` $< 2$ отбрасываются.** Остаётся `canonical.jsonl`.

<details>
<summary>Ручная курация после шага 3 (обязательна)</summary>

На сотнях тредов это 15–30 минут. По каждому кластеру:

1. Открыть 2–3 треда из `evidence_thread_ids` по ссылке `https://github.com/{repo}/pull/{pr_id}#discussion_r{databaseId}` — правило действительно выводимо из этих комментариев?
2. Правило переводимо в линтер/тест/CI-проверку? → пометить `automate_with` и исключить из рубрикатора.
3. `single_reviewer=true` → это норма команды или вкус одного человека? Во втором случае — `scope=local_taste`.
4. `conflict=true` или `merge_doubt` заполнен → разлепить или удалить.
5. Правило описывает стек, которого уже нет в репозитории → удалить.

Результат — `canonical_curated.jsonl` с полем `curation_note`. Правило, по которому вы сомневаетесь дольше минуты, удаляется — это и есть «консервативность» в действии.

</details>

## Шаг 4 — покрытие по осям

Код считает по `canonical_curated.jsonl` число кластеров и тредов на каждую ось из `taxonomy.yaml`, а также `repetitive = n_prs / n_threads < 0.4 OR n_reviewers == 1`. Модели остаётся интерпретация.

```text
Тебе даны канонические правила код-ревью с заранее вычисленной статистикой и список осей таксономии с числом правил по каждой.

ЗАДАЧА:
1. Для каждого правила проверь поле axis. Если ось явно неверна — заполни axis_suggested, исходную не меняй.
2. Для осей с n_clusters = 0 напиши blind_spot_note (одна строка): что можно и чего нельзя заключить из отсутствия данных. Не утверждай, что команда что-то «забыла» или «не умеет».
3. Для правил с repetitive=true напиши repetitive_note: похоже ли на одну нерешённую проблему в одном месте или на системную ошибку, судя по statement и counterexample.

Статистику не пересчитывай и не оспаривай.

ВЫВОД:
{"axes":[{"axis":"...","blind_spot_note":null}],"clusters":[{"cluster_id":"...","axis_suggested":null,"repetitive_note":null}]}

Данные:
{{COVERAGE_JSON}}
```

Выход этого шага — для человека (`coverage.json`), в рубрикатор он не идёт. Его польза при малых данных — честно показать, что правила покрывают 3–4 оси из 12, и это ограничение, а не свойство кода команды.

## Шаг 5 — приоритизация

Код агрегирует по `evidence_thread_ids` из `threads.jsonl`: `resolved_after_fix_rate`, `resolved_without_fix_rate`, `code_changed_rate`, `author_reply_rate`, `mean_n_participants`, `blocked_merge_rate`, `null_rate`. Модель даёт $s$:

```text
Тебе даны правила код-ревью со статистикой исходов по их свидетельствам. Поля stats (доли в 0..1, null — данных нет): n_threads, n_prs, n_reviewers, resolved_after_fix_rate, resolved_without_fix_rate, code_changed_rate, author_reply_rate, mean_n_participants, blocked_merge_rate, null_rate.

ЗАДАЧА: оценить s ∈ [0,1] — стоимость пропуска проблемы, которую ловит правило.

ШКАЛА:
- 0.0–0.2: автор вправе проигнорировать без вреда
- 0.3–0.5: качество без функционального риска
- 0.6–0.8: предотвращённая ошибка, не дошедшая бы до прода
- 0.9–1.0: инцидент, уязвимость, потеря данных

ИНТЕРПРЕТАЦИЯ СТАТИСТИКИ (следуй ей, а не общим соображениям о важности темы):
- resolved_after_fix_rate — главный сигнал принятия: автор согласился и переписал код.
- resolved_without_fix_rate высокий — замечания закрывались без правки: отклонены или несущественны. Понижает s.
- author_reply_rate высокий и mean_n_participants > 2 — спорное замечание, не обязательно важное.
- blocked_merge_rate — эвристика; вес не выше половины от resolved_after_fix_rate.
- null_rate > 0.5 → insufficient_data=true, s=null.
- Severity по формулировке statement учитывай, но статистика исходов важнее.

ВЫВОД:
{"scores":[{"cluster_id":"...","s":0.0,"insufficient_data":false,"reasoning":"одна строка"}]}

Данные:
{{CLUSTERS_WITH_STATS_JSON}}
```

Код считает приоритет. Логарифмы по тредам нужны, чтобы один кластер с 40 однотипными замечаниями не задавил остальные; число PR входит линейно как мера разнообразия:

$$priority = \log(1 + n_{threads}) \cdot n_{prs} \cdot a \cdot s, \qquad a = \begin{cases} resolved\_after\_fix\_rate & \text{если не null} \\ 0.5 & \text{иначе} \end{cases}$$

`insufficient_data=true` → `priority = null`, правило ранжируется в конец, но не удаляется.

## Шаг 6 — рубрикатор

```text
Собери рубрикатор для LLM-ревьюера из канонических правил с приоритетами. Вход: cluster_id, statement, axis, scope, applies_to, priority, counterexample, best_evidence (цитата + diff_hunk), automate_with.

ОГРАНИЧЕНИЯ:
1. Не более 15 правил. Правила с automate_with не включай — вынеси в секцию «Автоматизировать».
2. Для каждого правила: формулировка, ось, applies_to (если есть), пример нарушения (на основе counterexample/best_evidence, не выдумывай новый стек), пример НЕ-нарушения — похожий код, который правило НЕ должно ловить. Если убедительный пример не-нарушения построить не удаётся — правило слишком расплывчато, помести его в «Отложено» с этой причиной.
3. Секция «Когда молчать»: сгенерированный код, lock-файлы, миграции, чистое переименование, изменения только в тестовых фикстурах, а также всё, что явно следует из применимости правил (например, правило про ORM не применяется к файлам без ORM).
4. Правила scope=local_taste — отдельная секция «Соглашения этого репозитория» с пометкой, что они неприменимы к внешнему коду.
5. Не добавляй правил, которых нет во входе. Не дописывай «общеизвестные» проверки.

ФОРМАТ — Markdown:
## Роль
Ты — ревьюер pull request. Оставляешь замечания только по правилам ниже, ссылаясь на номер правила. Если сомневаешься — молчишь.
## Правила
### R1. <название>
- Проверка: ...
- Применимо к: ...
- Нарушение: ```<язык> ...```
- Не нарушение: ```<язык> ...```
## Когда молчать
## Соглашения этого репозитория
## Автоматизировать (не для ревьюера)
## Отложено

Данные:
{{PRIORITY_JSON}}
```

После шага 6 — вторая ручная курация, короткая: прочитать рубрикатор как инструкцию для нового сотрудника. Пункт, который непонятен без контекста пайплайна, переписывается или удаляется.

## Шаг 7 — валидация на holdout

Промпт ревьюера:

```text
Ты — LLM-ревьюер. Дан дифф (один файл или его часть) и рубрикатор. Ты не знаешь и не пытаешься угадать, что написали люди.

ЗАДАЧА: выдать замечания строго по рубрикатору.

ПРАВИЛА:
1. Каждое замечание ссылается на номер правила и на конкретные строки диффа (номера из RIGHT-стороны).
2. Приоритет — минимум ложных срабатываний. Сомневаешься — не пиши.
3. Проблемы вне рубрикатора — в outside_rubric, они не считаются ответом.
4. Пустой findings — нормальный результат.

ВЫВОД:
{"findings":[{"rule_id":"R1","path":"...","line_start":0,"line_end":0,"comment":"...","confidence":0.0}],"outside_rubric":[{"path":"...","line_start":0,"note":"..."}]}

Рубрикатор:
{{RUBRIC_MD}}

Дифф:
{{DIFF}}
```

Baseline — тот же промпт, но вместо рубрикатора одна строка: «Найди дефекты, краевые случаи и проблемы дизайна». Без пары чисел «с рубрикатором / без» результат не интерпретируется.

<details>
<summary>evaluate.py — ground truth и матчинг</summary>

- **Дифф на вход**: для каждого PR из `eval/threads.jsonl` берутся файлы, в которых есть треды; дифф выгружается через `GET /repos/{o}/{r}/pulls/{n}/files` (поле `patch`). Файлы без тредов тоже подаются — иначе precision завышен.
- **Ground truth**: треды holdout прогоняются через промпт 1 (он ничему не учится), остаются `category ∈ {defect, design, docs}`, `needs_context=false`.
- **Матч**: совпадает `path`, диапазоны строк пересекаются с допуском $\pm 3$, и косинусная близость `normalized` ↔ `finding.comment` выше порога (эмбеддер любой multilingual, порог подобрать на 20–30 ручных парах; при малых данных можно и вовсе разметить матчи руками — их будет несколько десятков).
- **Отчёт в абсолютных числах**: GT всего / найдено с рубрикатором / найдено baseline / ложных с рубрикатором / ложных baseline / findings с `confidence < 0.5`. Разбивка по `rule_id`: правило с нулём совпадений и ненулевыми ложными — кандидат на удаление.
- **Критерий успеха**: рубрикатор находит больше GT, чем baseline, при не большем числе ложных. Если recall растёт только за счёт `confidence < 0.5` — это шум.

</details>

---

## Порядок запуска и обвязка

`run_llm.py` — единый раннер для шагов 1–7: читает промпт из `prompts/`, подставляет батч, вызывает модель с `temperature=0`, валидирует ответ по `schemas/`, при ошибке парсинга повторяет запрос с текстом ошибки (до 3 раз), пишет `run_id` и `schema_version` в каждую строку. Батчи не пересекаются; при падении батча остальные продолжаются, упавшие — в `errors.jsonl`.

```
extract.py → prepare.py → [p1 ×1] → filter claims → [p2 ×3, shuffle] → filter_rules.py
→ [p3 ×1] → drop runs_covered<2 → КУРАЦИЯ → coverage (код + p4) → stats + [p5] + формула
→ [p6] → КУРАЦИЯ → eval: p1 на holdout, p7 ×2 (рубрикатор / baseline), evaluate.py
```

`taxonomy.yaml` подставляется в промпты 2 и 4 через `{{TAXONOMY_AXES}}`. `PIPELINE.md` из прошлого ответа актуализируйте тремя строками в журнале решений: тред как единица; `reviews.jsonl` не выгружается; пороги консервативности ($\geq 3$ тредов, $\geq 2$ PR, $\geq 2$ прогонов).

<details>
<summary>Чего ожидать на этом объёме и когда останавливаться</summary>

- Если после шага 3 осталось ≤ 2 правил — не ослабляйте пороги. Либо расширяйте окно дат / список репозиториев, либо честно зафиксируйте, что ревью в команде неоднородны и правила из них не выводятся. Это тоже результат.
- Если `prepare.py` показал ≤ 2 уникальных ревьюеров — всё, что получится, описывает стиль этих людей. Рубрикатор пригоден, но в `PIPELINE.md` это надо записать как ограничение.
- Holdout из 30–50 тредов даст 10–20 GT-замечаний. Разница в 2–3 найденных между рубрикатором и baseline — не доказательство. Доказательство — повторить шаг 7 через квартал на новых PR.
- Промпт 1 — единственный шаг, где стоит выборочно проверить разметку руками (20 тредов). Ошибка там тиражируется дальше, а стоит проверка десять минут.

</details>

Два предположения, которые я сделал молча и которые стоит проверить: репозиториев один или несколько в пределах одной команды (иначе `applies_to` и `local_taste` разъезжаются по репо, и нужна группировка по `repo`), и в команде больше двух активных ревьюеров (иначе см. предупреждение в `prepare.py`). Если оба неверны — скажите, поправлю шаги 0b и 3.
