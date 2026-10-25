### Ваша задача на этом этапе — не писать код, а спроектировать фундаментальную архитектуру данных (Data Blueprint), которая выдержит расчет катастрофических фазовых переходов без деградации производительности и нарушения консистентности. 

Любая попытка спроектировать "плоскую" базу данных без учета темпоральности или смешать сырые данные с расчетными витринами будет расценена как архитектурный брак. Проект не будет допущен к этапу физического развертывания (DDL).

Ниже приведено исчерпывающее руководство по логическому и графическому проектированию.

---

## МЕТОДИЧЕСКОЕ РУКОВОДСТВО: ПАРА 1. ЛОГИЧЕСКОЕ И ГРАФИЧЕСКОЕ ПРОЕКТИРОВАНИЕ

#### Шаг 1. Идентификация предметной области и выбор ПАО
До начала отрисовки ER-диаграммы вы обязаны определить физический смысл вашей модели.
1.  **Выбор эмитента:** Вы выбрали публичное акционерное общество (ПАО) из реального сектора (например, нефтегаз, металлургия, ритейл, телеком). Финансовый сектор (банки) брать **запрещено**, так как их модель баланса и расчета ликвидности кардинально отличается от корпоративного сектора.
2.  **Определение экзогенных факторов:** Выделите 1-2 ключевых макроэкономических или геополитических параметра, импульсное возмущение которых способно привести компанию к дефолту (например: ключевая ставка ЦБ РФ, дисконт на нефть марки Urals, индекс цен на металлы).
3.  **Определение ковенант:** Зафиксируйте пределы устойчивости (ковенанты) для выбранного ПАО (например, `Net Debt / EBITDA > 4.0`, `ICR < 1.5`).

#### Шаг 2. Проектирование слоя `staging` (Raw Data Layer)
Этот слой опирается исключительно на контракты данных из репозитория `12.10.2026_ELT-Pipeline-Stagging.git`. 

**Архитектурные директивы:**
*   **Категорический запрет на связи:** В схеме `staging` не должно быть никаких Foreign Keys (FK).
*   **Запрет на трансформации:** Структура таблиц должна 1:1 соответствовать JSON/CSV ответам от API (MOEX, Yahoo Finance, ЦБ РФ, парсинг отчетности).
*   **Обязательные мета-поля:** Каждая таблица должна содержать технические поля: `load_dt` (timestamp загрузки) и `source_system` (источник).

**Требуемые сущности (пример):**
*   `stg_raw_financials` (сырые строки из РСБУ/МСФО).
*   `stg_raw_market_data` (котировки акций, объемы торгов).
*   `stg_raw_macro` (сырые ряды макропоказателей).

#### Шаг 3. Проектирование слоя `core` (Operational Data Store & DWH)
Здесь вы перерабатываете хаотичную логику из `12.10.2026_ELT-Pipeline-Self.git` в строгую реляционную модель (3NF или Star Schema). Это ядро вашей системы.

**Архитектурные директивы:**
*   **SCD Type 2:** Все измерения (Dimensions) обязаны поддерживать версионирование. Добавьте поля: `surrogate_key` (PK), `business_key`, `valid_from`, `valid_to`, `is_current`.
*   **Непрерывность фазовой траектории:** Фактовые таблицы должны обеспечивать построение непрерывного временного ряда (Time Series) без разрывов.

**Требуемые сущности (Обязательный минимум):**
1.  **`dim_company` (Измерение):** Профиль ПАО (сектор, тикер, дата IPO). Поддерживает SCD-2 (например, при смене юрисдикции или тикера).
2.  **`dim_covenant_threshold` (Измерение):** Справочник критических границ метрик. 
    *   *Поля:* `covenant_id`, `metric_name` (например, 'ND/EBITDA'), `critical_value`, `warning_value`, `valid_from`, `valid_to`, `is_current`.
3.  **`fact_endogenous_state` (Факт):** Таблица внутренних состояний системы.
    *   *Поля:* `state_id`, `company_sk` (FK), `report_date`, `ebitda`, `net_debt`, `icr`, `altman_z_score`, `fcf`.
4.  **`fact_exogenous_shock` (Факт):** Регистрация внешних импульсов.
    *   *Поля:* `shock_id`, `parameter_name` (например, 'CB_RATE'), `event_timestamp`, `absolute_value`, `delta_change` (амплитуда $\Delta$).

#### Шаг 4. Проектирование слоя `sandbox` (What-If Layer)
Слой для материализации результатов Pushdown-вычислений и симуляций. Базовые данные из `core` сюда не дублируются, здесь хранятся только результаты стресс-тестов.

**Требуемые сущности:**
1.  **`sandbox.simulation_results`:** Результаты параметрического сдвига.
    *   *Поля:* `sim_id`, `company_sk`, `simulated_timestamp`, `exogenous_param`, `simulated_value`, `projected_nd_ebitda`, `is_bifurcation_reached` (Boolean).
2.  **`sandbox.audit_log`:** Журнал аудита запусков симуляций.
    *   *Поля:* `run_id`, `run_timestamp`, `exogenous_vector_json`, `step_size`, `user_id`.

#### Шаг 5. Графическая реализация (ER-диаграмма)
Используйте любой профессиональный инструмент (Draw.io, dbdiagram.io, DataGrip).

**Требования к графике:**
1.  **Нотация:** Строго Crow's Foot (Воронья лапка) для отображения кардинальности связей (1:1, 1:M, M:N).
2.  **Цветовое кодирование слоев:** 
    *   `staging` — серый цвет.
    *   `core` — синий (измерения) и зеленый (факты).
    *   `sandbox` — оранжевый/красный.
3.  **Типизация:** Для каждого атрибута должен быть указан физический тип данных СУБД (PostgreSQL/ClickHouse), например: `NUMERIC(15,4)`, `TIMESTAMP`, `VARCHAR(255)`, `BOOLEAN`.

---

### ЧЕК-ЛИСТ ДЛЯ ЗАЩИТЫ (КРИТЕРИИ ПРИЕМКИ)

1.  **Изоляция слоев:** *"Покажите, как данные из `staging` попадают в `core`. Где происходит расчет Z-счета Альтмана?"* (Правильный ответ: расчет происходит на лету при `INSERT INTO core.fact_endogenous_state ... SELECT` средствами SQL, а не в Pandas).
2.  **Темпоральность:** *"Что произойдет в вашей модели, если совет директоров ПАО сегодня изменит дивидендную политику или ковенанту по долгу, а нам нужно пересчитать симуляцию за прошлый год?"* (Правильный ответ должен опираться на механику `valid_from` / `valid_to` в `dim_covenant_threshold`).
3.  **Точка бифуркации:** *"Как ваша структура БД позволяет SQL-запросу (оконной функции) определить момент пересечения метрикой критического порога?"* (Ожидаю увидеть связь между `sandbox.simulation_results` и `dim_covenant_threshold`).

**Ваша задача на текущую пару:** Предоставить готовую ER-диаграмму в формате изображения или ссылки на dbdiagram.io, сопровождаемую кратким текстовым обоснованием выбора ПАО и экзогенных шоков. Приступайте.

---

# Методические указания по инструментальному проектированию

## Категорически не принимается подход, при котором структура корпоративного хранилища рисуется мышкой в графических редакторах. Визуальное проектирование (drag-and-drop) порождает ошибки, не поддается версионированию в Git и не позволяет автоматически генерировать DDL-скрипты. 

Наш стандарт — **Infrastructure-as-Code (IaC)**. Для логического и графического проектирования мы используем сервис **dbdiagram.io**, который работает на базе языка разметки **DBML (Database Markup Language)**. Это ваш строгий контракт данных.

Ниже представлено дополнение к методическому пособию: исчерпывающее руководство по работе с dbdiagram.io с учетом наших архитектурных требований.

---

### ИНСТРУМЕНТАРИЙ АРХИТЕКТОРА: РУКОВОДСТВО ПО DBDIAGRAM.IO (DBML)

#### 1. Почему dbdiagram.io?
*   **Декларативность:** Вы описываете структуру кодом, а сервис сам строит ER-диаграмму в нотации Crow's Foot.
*   **Экспорт в DDL:** Написанный DBML-код в один клик экспортируется в готовый SQL-скрипт (PostgreSQL), который вы будете использовать на Паре 2.
*   **Группировка слоев:** Поддержка `TableGroup` позволяет визуально изолировать схемы `staging`, `core` и `sandbox`, что является нашим жестким требованием.

#### 2. Базовый синтаксис DBML (Директивы)
Ваш код должен быть строго типизирован. Я буду штрафовать за использование абстрактных типов вроде `String` или `Number`. Мы проектируем под PostgreSQL/ClickHouse.

*   **Создание таблицы и схемы:** `Table schema_name.table_name { ... }`
*   **Типы данных:** `varchar(255)`, `numeric(15,4)` (для финансовых метрик), `timestamp` (для дат и времени).
*   **Ограничения (Constraints):** Указываются в квадратных скобках: `[pk]` (Primary Key), `[not null]`, `[default: current_timestamp]`.
*   **Связи (Foreign Keys):** Описываются отдельно через оператор `Ref`. 
    *   `>` означает связь "многие к одному" (Many-to-One).
    *   `-` означает связь "один к одному" (One-to-One).

#### 3. Шаблон проектирования (DBML Blueprint)
Скопируйте этот базовый шаблон в редактор на сайте [dbdiagram.io](https://dbdiagram.io) и расширьте его под выбранное вами ПАО и экзогенные факторы. 

*Внимание: Внимательно изучите, как реализован SCD Type 2 и как изолирован слой staging.*

```dbml
// ==========================================
// LAYER 1: STAGING (RAW DATA)
// Директива: Никаких связей (Ref) в этом слое!
// ==========================================

Table staging.stg_raw_financials {
  raw_id uuid [pk, note: 'Генерируется при инжекции']
  source_system varchar(50) [note: 'Например: MOEX_API, OSINT_Parser']
  load_dt timestamp [default: `current_timestamp`]
  raw_json text [note: 'Сырой ответ API или распарсенный HTML']
  report_period varchar(20)
}

Table staging.stg_raw_macro {
  raw_id uuid [pk]
  load_dt timestamp [default: `current_timestamp`]
  indicator_name varchar(100)
  indicator_value numeric(10,4)
  observation_date date
}

// ==========================================
// LAYER 2: CORE (OPERATIONAL DATA STORE)
// Директива: Строгая 3NF / Star Schema, SCD-2
// ==========================================

Table core.dim_company {
  company_sk serial [pk, note: 'Суррогатный ключ (SCD-2)']
  ticker varchar(10) [not null, note: 'Бизнес-ключ']
  sector varchar(50)
  valid_from timestamp [not null]
  valid_to timestamp [default: `9999-12-31 23:59:59`]
  is_current boolean [default: true]
}

Table core.dim_covenant_threshold {
  covenant_sk serial [pk, note: 'Суррогатный ключ (SCD-2)']
  metric_name varchar(50) [not null, note: 'Например: Net Debt / EBITDA']
  critical_value numeric(10,4) [not null, note: 'Точка бифуркации']
  valid_from timestamp [not null]
  valid_to timestamp [default: `9999-12-31 23:59:59`]
  is_current boolean [default: true]
}

Table core.fact_endogenous_state {
  state_id bigserial [pk]
  company_sk integer [not null]
  report_date date [not null]
  ebitda numeric(15,4)
  net_debt numeric(15,4)
  icr numeric(10,4) [note: 'Interest Coverage Ratio']
  altman_z_score numeric(10,4)
}

Table core.fact_exogenous_shock {
  shock_id bigserial [pk]
  parameter_name varchar(50) [not null, note: 'Например: CB_RATE']
  event_timestamp timestamp [not null]
  absolute_value numeric(10,4)
  delta_change numeric(10,4) [note: 'Амплитуда возмущения']
}

// Определение связей (Foreign Keys) для слоя CORE
Ref: core.dim_company.company_sk < core.fact_endogenous_state.company_sk
Ref: core.dim_company.company_sk < core.fact_exogenous_shock.shock_id // Опционально, если шок специфичен для компании

// ==========================================
// LAYER 3: SANDBOX (WHAT-IF SIMULATION)
// Директива: Изолированная среда для Pushdown-расчетов
// ==========================================

Table sandbox.simulation_results {
  sim_id bigserial [pk]
  run_id uuid [not null]
  company_sk integer [not null]
  simulated_timestamp timestamp
  exogenous_param varchar(50)
  simulated_value numeric(15,4)
  projected_nd_ebitda numeric(10,4)
  is_bifurcation_reached boolean [default: false, note: 'Флаг катастрофического перехода']
}

Table sandbox.audit_log {
  run_id uuid [pk]
  run_timestamp timestamp [default: `current_timestamp`]
  exogenous_vector_json text [note: 'Параметры сдвига']
  step_size numeric(10,4)
}

Ref: sandbox.audit_log.run_id < sandbox.simulation_results.run_id
Ref: core.dim_company.company_sk < sandbox.simulation_results.company_sk

// ==========================================
// ВИЗУАЛЬНАЯ ГРУППИРОВКА (TABLE GROUPS)
// ==========================================
TableGroup Staging_Layer {
  staging.stg_raw_financials
  staging.stg_raw_macro
}

TableGroup Core_Layer {
  core.dim_company
  core.dim_covenant_threshold
  core.fact_endogenous_state
  core.fact_exogenous_shock
}

TableGroup Sandbox_Layer {
  sandbox.simulation_results
  sandbox.audit_log
}
```

#### 4. Пошаговая инструкция для студента
1.  **Инициализация:** Откройте [dbdiagram.io](https://dbdiagram.io) и создайте новую диаграмму.
2.  **Кодирование:** Вставьте базовый шаблон DBML в левую панель редактора.
3.  **Адаптация:** 
    *   Добавьте специфичные для вашего ПАО метрики в `fact_endogenous_state` (например, `fcf` — Free Cash Flow, если это капиталоемкое производство).
    *   Добавьте таблицы-справочники, если они требуются для вашей модели.
4.  **Валидация:** Убедитесь, что на правой панели (визуализация) слои четко разделены цветами/группами, а связи (линии) не тянутся из `staging` в `core`.
5.  **Экспорт DDL (Подготовка к Паре 2):** 
    *   Нажмите кнопку **Export** на верхней панели.
    *   Выберите **Export to PostgreSQL**.
    *   Сохраните полученный `.sql` файл. Этот скрипт станет фундаментом для развертывания базы данных в Google Colab через SQLAlchemy.

**Требование к сдаче:** На защиту Пары 1 вы должны принести не только скриншот красивой схемы, но и ссылку на ваш проект в dbdiagram.io (Share -> Copy Link), чтобы я мог провести аудит вашего DBML-кода на предмет соответствия типам данных и архитектурным паттернам. Отклонения от SCD-2 или отсутствие `TableGroup` приведут к незачету этапа.
