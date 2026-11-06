# Визуальные графики: этапы, потоки, результат

Все схемы в Mermaid — открываются в VS Code (Markdown Preview) и на GitHub.
Нумерация — как в `plan.md` (этапы 0–9); группы в `tasks.md` идут 1–10 и в заголовке каждой указан номер этапа.

> **Актуальность (2026-11-03).** Схемы §10–§11 описывают механику `startOnLink`, которую PR #428 (`ca12e587e5`,
> ветка `FUSIO-1307-livekit-shared-link`) убрал из модели: статуса `Scheduled` нет, перезапуска завершённой
> встречи нет, ссылка — указатель на серию с `meetingAccess.start`. Читать их как историю решения; фактическое
> состояние — `audit-pr-428.md` §4.

## 1. Карта этапов и зависимостей

Что показывает: какие этапы можно вести параллельно, а какие ждут друг друга.

```mermaid
flowchart TD
  S0["Этап 0<br/>Стенд и базовая линия"]

  subgraph P1["P1 — правильность поведения"]
    S1["Этап 1<br/>Отмена и перенос вхождения"]
    S2["Этап 2<br/>Миксин только на мастере"]
  end

  subgraph P2["P2 — права и UX"]
    S3["Этап 3<br/>Владелец постоянной встречи"]
    S4["Этап 4<br/>Навигатор: сегмент и кнопки"]
  end

  subgraph P3["P3 — гигиена"]
    S7["Этап 6<br/>Язык и гостевые режимы"]
    S8["Этап 7<br/>Чистка полей модели"]
  end

  S5["Этап 5<br/>Внеплановый вход по ссылке"]
  S9["Этап 8<br/>Сквозная проверка"]

  S0 --> S1
  S0 --> S2
  S0 --> S3
  S0 --> S4
  S0 --> S7
  S0 --> S8
  S1 --> S5
  S1 --> S9
  S2 --> S9
  S3 --> S9
  S5 --> S9
  S7 --> S9
  S8 --> S9
```

## 2. Что изменится для пользователя

Что показывает: конкретные симптомы «до» и «после», без внутренних деталей.

```mermaid
flowchart LR
  subgraph BEFORE["Было"]
    B1["Отменил встречу в календаре —<br/>ссылка всё равно открывает сессию"]
    B2["Миксин можно поставить на вхождение —<br/>у серии появляется вторая ссылка"]
    B3["Открыл протокол во время звонка —<br/>навигатор сбросил выбранный пункт"]
    B4["Вне окна вхождения ссылка мертва:<br/>«встреча ещё не началась»"]
    B5["Владельца постоянной встречи<br/>можно убрать из участников"]
    B6["Русский пользователь получает<br/>англоязычную транскрипцию"]
  end

  subgraph AFTER["Стало"]
    A1["Отменённое вхождение:<br/>отказ и следующая действительная дата"]
    A2["Одна серия — одна ссылка:<br/>инстанс не становится встречей"]
    A3["Пункт навигатора и адрес<br/>сохраняются при возврате"]
    A4["Встреча начинается в любой момент,<br/>комната пересоздаётся, ссылка та же"]
    A5["Владелец всегда участник,<br/>проверка на сервере"]
    A6["Язык из интерфейса,<br/>можно выбрать"]
  end

  B1 -.-> A1
  B2 -.-> A2
  B3 -.-> A3
  B4 -.-> A4
  B5 -.-> A5
  B6 -.-> A6
```

## 3. Порядок работ во времени

Что показывает: последовательность и относительную длительность при последовательном исполнении (календарь условный).

```mermaid
gantt
  title Love meetings: этапы доделок
  dateFormat YYYY-MM-DD
  axisFormat %d.%m
  section База
  Стенд и базовая линия            :a0, 2026-11-02, 1d
  section Правильность
  Отмена и перенос вхождения       :a1, after a0, 2d
  Миксин только на мастере         :a2, after a0, 1d
  section Права и UX
  Владелец постоянной встречи      :a3, after a0, 1d
  Навигатор: сегмент и кнопки      :a4, after a0, 1d
  Внеплановый вход по ссылке       :a5, after a1, 2d
  section Гигиена
  Язык и гостевые режимы           :a7, after a0, 1d
  Чистка полей модели              :a8, after a0, 1d
  section Финал
  Сквозная проверка                :a9, after a1 a2 a3 a5 a7 a8, 1d
```

## 4. Как теперь резолвится гостевая ссылка

Что показывает: точка, куда встраивается этап 1 — переопределения вхождения читаются до расчёта окна.

```mermaid
sequenceDiagram
  autonumber
  participant G as Гость
  participant L as love-сервис
  participant AC as account-shortlink
  participant CAL as календарь
  participant DB as сессии

  G->>L: открыть /meetings/{shortId}
  L->>AC: resolveShortLink
  AC-->>L: payload eventId, workspace, linkVersion
  L->>CAL: мастер события
  L->>CAL: переопределения вхождения (isCancelled, originalStartTime)
  CAL-->>L: набор вхождений с учётом отмены и переноса
  L->>L: окно входа и версия ссылки

  alt ссылка отозвана или истекла
    L-->>G: отказ: ссылка отозвана или истекла
  else вхождение отменено
    L-->>G: встреча не открыта и следующая действительная дата
  else окно закрыто
    L-->>G: встреча не открыта и следующая дата
  else окно открыто
    L->>DB: найти живую сессию или создать
    DB-->>L: meetingId
    L-->>G: meetingId и время вхождения
  end
```

## 5. Жизненный цикл сессии

Что показывает: где именно действуют окно входа, терминальность завершения и повторный вход.

```mermaid
stateDiagram-v2
  [*] --> ВнеОкна: серия или постоянная встреча создана
  ВнеОкна --> ОкноОткрыто: за 15 минут до вхождения
  ОкноОткрыто --> Pending: вход участника или гостя
  Pending --> Active: комната LiveKit создана
  Active --> Finished: комната опустела или нажали End
  Finished --> ОкноОткрыто: повторный вход в то же вхождение
  ОкноОткрыто --> ВнеОкна: через 4 часа после старта
  ВнеОкна --> [*]

  note right of Finished
    Терминален: meetingEnd не перезаписывается,
    следующее вхождение получает новую сессию
  end note
```

## 6. Модель: где живёт что

Что показывает: почему настройки встречи живут в миксине мастера, а время и участники — в событии.

```mermaid
classDiagram
  class Event {
    AccessLevel access
    Timestamp date
    Timestamp dueDate
    RecurRule rules
    TimestampArray rdate
    TimestampArray exdate
    AccountList participants
  }

  class ReccuringInstance {
    Ref recurringEventId
    Timestamp originalStartTime
    boolean isCancelled
  }

  class MeetingEventLink {
    RoomType type
    number linkVersion
    MeetingAccess meetingAccess
    boolean private
    string language
    boolean startWithRecording
    boolean startWithTranscription
  }

  class MeetingMinutes {
    string eventId
    Ref meeting
    Timestamp occurrence
    MeetingStatus status
    Ref roomId
  }

  class PermanentMeeting {
    string name
    RoomType type
    string language
    MemberList members
  }

  Event <|-- ReccuringEvent
  ReccuringEvent "1" --> "*" ReccuringInstance : вхождения
  Event ..> MeetingEventLink : миксин только на мастере
  MeetingMinutes ..> Event : eventId серии
  MeetingMinutes ..> PermanentMeeting : meeting
```

## 7. Навигатор офиса и адреса

Что показывает: что означает каждый URL-сегмент и куда ведёт выбор.

```mermaid
flowchart LR
  URL["/love/&lt;segment&gt;"]
  URL --> M["meetings<br/>Календарь встреч"]
  URL --> P["permanent<br/>Таблица постоянных встреч"]
  URL --> F["&lt;floorId&gt;<br/>Этаж с людьми"]

  M --> MP["Панель встречи:<br/>редактор события, присоединиться, протоколы"]
  P --> PP["Панель постоянной встречи:<br/>название, присоединиться, протоколы"]
  F --> FP["Комнаты этажа:<br/>звонок, прошедшие встречи этажа"]
```

## 8. Риски этапов

Что показывает: где нужен запас по времени и предварительная проверка.

```mermaid
quadrantChart
  title Риски: вероятность против влияния
  x-axis "Низкая вероятность" --> "Высокая вероятность"
  y-axis "Малое влияние" --> "Большое влияние"
  quadrant-1 "Держать под контролем"
  quadrant-2 "Проверить заранее"
  quadrant-3 "Принять"
  quadrant-4 "Мониторить"
  "Права на PersonSpace": [0.55, 0.85]
  "Гонка сборки стенда": [0.6, 0.7]
  "Серверный запрет миксина": [0.35, 0.75]
  "Удаление linkId ломает клиента": [0.3, 0.65]
  "Навигатор и переходы из звонка": [0.6, 0.4]
  "Тяжёлая выборка протоколов": [0.4, 0.35]
  "Флаки в sanity": [0.8, 0.2]
```

## 9. Зоны кода по этапам

Что показывает: какие каталоги трогает работа и где искать при отладке.

```mermaid
mindmap
  root((Love meetings))
    Модель
      plugins/love/src/types.ts
      models/love/src/index.ts
    Резолв
      plugins/love/src/utils.ts
      services/love/src/workspaceClient.ts
      services/love/src/guests.ts
    Триггеры
      server-plugins/calendar-resources
      server-plugins/love-resources
    UI
      OfficeNavigator.svelte
      EditScheduledMeeting.svelte
      EditPermanentMeeting.svelte
      MeetingMinutesSection.svelte
    Тесты
      unit в пакетах
      tests/sanity/tests/love
      playwright панели и пароль
    Документация
      foundation-docs virtual-office.mdx
```

## 10. Внеплановый старт по ссылке (этап 5)

Что показывает: как решается вопрос «сейчас не время встречи, но я хочу подключиться» — и почему ссылка при этом не меняется.

```mermaid
flowchart TD
  L["Открыли ссылку /mеetings/{shortId}"] --> V{"Ссылка отозвана<br/>или истекла?"}
  V -->|да| E1["Отказ: ссылка отозвана или истекла"]
  V -->|нет| S{"Есть живая сессия<br/>по eventId?"}
  S -->|да| J["Возвращаем её: входим в текущую комнату"]
  S -->|нет| W{"Сейчас есть<br/>вхождение в окне?"}
  W -->|да| C["Создаём сессию вхождения<br/>occurrence = originalStartTime"]
  W -->|нет| P{"Политика start<br/>разрешает внеплановый старт?"}
  P -->|members и это участник| C2["Создаём внеплановую сессию<br/>occurrence = null"]
  P -->|link и это гость| C2
  P -->|гость при members| E2["Отказ: встреча не открыта<br/>и дата следующего вхождения"]
  C --> R["LiveKit-комната создаётся при входе"]
  C2 --> R
  R --> F["Пустая комната завершается автоматически"]
  F -.->|повторный вход| L
```

## 11. Гостевая ссылка всегда активна (выполнено в `FUSIO-1307-livekit-shared-link`)

Что показывает: что происходит при входе по ссылке после правки — встреча перезапускается, комната создаётся, `meetingId` и ссылка не меняются.

```mermaid
sequenceDiagram
  autonumber
  participant G as Гость
  participant UI as GuestMeetingApp
  participant L as love-сервис
  participant W as MeetingMinutes
  participant LK as LiveKit

  G->>UI: открыть ссылку
  UI->>L: POST /guestInfo
  L-->>UI: title, meetingStatus, roomFound, meetingScheduledDate
  UI->>G: название, статус, форма входа
  G->>UI: имя и «присоединиться»
  UI->>L: POST /guestJoin
  L->>W: findMeetingById
  alt Scheduled или Finished и startOnLink включён
    L->>W: restartMeeting (status=Active, meetingEnd=null)
  else Scheduled или Finished и startOnLink выключен
    L-->>UI: 403 «встреча не началась / уже завершена»
  end
  alt комнаты нет
    L->>LK: createRoom(name, metadata, agents)
  end
  L->>W: ensurePersonByName
  L-->>UI: token, roomName, person
  UI->>LK: connect
  Note over W,LK: участники, зашедшие позже, переиспользуют эту же встречу<br/>и попадают в ту же комнату
  Note over L,W: /getToken вне окна: владелец встречи (owners)<br/>или владелец workspace переводит Scheduled в Active
```

## 12. Создание встречи: два пути и связь между документами (2026-11-06)

Что показывает: какие документы появляются при создании встречи из формы события (реализовано) и из записи
планировщика (осталось), и кто на кого ссылается после того, как связь переехала в `permanentMeeting`.

```mermaid
flowchart TD
  subgraph Event["Форма события — calendar.class.Event (реализовано)"]
    E1["MeetingData: комната, участники,<br/>сделать постоянной, repeatEvery"] --> E2["createMeeting, фаза post"]
    E2 --> E3["серия Event + makeRecurring"]
    E3 --> E4["миксин MeetingEventLink на мастере<br/>linkVersion, meetingAccess, reminderBefore"]
    E4 --> E5{"переключатель включён?"}
    E5 -->|да| E6["PermanentMeeting<br/>repeatEvery, members, owners"]
    E6 --> E7["MeetingEventLink.permanentMeeting = _id"]
    E5 -->|нет| E8["только серия"]
  end

  subgraph Planner["Запись планировщика — calendar.class.Schedule (осталось)"]
    P1["MeetingScheduleData: комната,<br/>участники, промежуток"] --> P2["createMeetingSchedule, фаза post"]
    P2 --> P3["серия Event + миксин<br/>(тот же код, что у события)"]
    P2 --> P4["PermanentMeeting"]
    P4 --> P5["MeetingSchedule.meetingId = _id"]
  end

  E4 --> T1["Вкладка «Встречи»<br/>SpecialView на миксине"]
  P3 --> T1
  E6 --> T2["Вкладка «Постоянные встречи»<br/>SpecialView на PermanentMeeting"]
  P4 --> T2
  E4 --> B1["Бейдж входа на карточке<br/>EventCardExtensions"]
  P3 --> B1
  B1 --> S["Сессия MeetingMinutes<br/>eventId + occurrence"]
```

Почему так: миксин `MeetingEventLink` объявлен на `calendar.class.Event`, поэтому запись планировщика сама попасть
во вкладку «Встречи» не может — путь планировщика обязан создать серию событий. Ссылки две, каждая со своей стороны:
`MeetingEventLink.permanentMeeting` (серия → постоянная встреча) и `MeetingSchedule.meetingId`
(запись планировщика → постоянная встреча).

## 13. Правка и удаление регулярной встречи (2026-11-06)

Что показывает: что происходит с сессией при правке участников и при удалении серии.

```mermaid
stateDiagram-v2
  [*] --> Серия: создание события или записи планировщика
  Серия --> ЖиваяСессия: вход в комнату, resolveSession
  ЖиваяСессия --> ЖиваяСессия: добавлен участник (OnEventUpdate, $push members)
  ЖиваяСессия --> Завершена: удалён мастер серии (OnEventUpdate, TxRemoveDoc)
  ЖиваяСессия --> Завершена: встреча окончена в панели
  Серия --> Серия: правка периода (retimedRules)
  Серия --> ОтменённоеВхождение: exdate вхождения
  ОтменённоеВхождение --> [*]
  Завершена --> [*]
  note right of ЖиваяСессия
    удаление копии участником ничего не завершает:
    мастер - только копия с access = Owner
  end note
```
