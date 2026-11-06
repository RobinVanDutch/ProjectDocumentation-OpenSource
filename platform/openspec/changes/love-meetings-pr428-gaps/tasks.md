# Tasks

Порядок — по цене ошибки. Сборку и тесты запускает владелец репозитория; в задачах указано, что именно проверить.

> **Состояние на 2026-11-03.** Группы 1, 3, 4 и 6 закрыты, группа 2 (перенос вхождения) перенесена на следующую
> итерацию, этап 5 (внеплановый старт) закрыт решением «не нужен», остальное открыто — группы 5 (ручные проверки),
> 7.2 (HTTP-тест `authorizeTarget`), 8.2 (тест RSVP), 9 (документация), 10 (сквозной прогон).

## 1. Состояние кода зафиксировано в спеке

- [x] 1.1 `docs/love.md` §8.3: возвращены контракт `/guestJoin` (`personToken` необязателен, выведенный `guest-link:<sha256>`, `400` без имени) и ответы `/guestInfo` с `passwordRequired`/`mode` — проверка: текст совпадает с `services/love/src/guests.ts`.
- [x] 1.2 `docs/features/office-meetings.md`: абзац про создание встречи переписан под модель PR (миксин `MeetingEventLink`, сессию создаёт `resolveSession`, `meetingAccess`), причина `stale` в таблице документации заменена — проверка: в файле нет `startOnLink`, `MeetingStatus.Scheduled`, `restartMeeting`.
- [x] 1.3 `openspec/changes/love-meetings-followups/tasks.md`: проставлены статусы по `audit-pr-428.md` §2 (этап 2 сделан, 1 частично, 3/5/6/7/8 остаются, 4 частично) — проверка: `openspec validate --all --strict`.
- [x] 1.4 `openspec/changes/love-meeting-link/`: шапка «поглощено PR #428», требования про `startOnLink`/`Scheduled` помечены устаревшими — проверка: `openspec validate --all --strict`.
- [x] 1.5 `plan.md` и `diagrams.md`: пометка, что блоки про `startOnLink`/`Scheduled` устарели после PR #428, актуальное состояние — `audit-pr-428.md` и этот change.

## 2. Перенос вхождения серии — отложено на следующую итерацию (решение 2026-11-03)

> Группа не в работе: источник перенесённых времён не согласован (`design.md` § Open Questions 1).
> Остаток работ от неё не зависит, поэтому итерация закрывается без переноса; e2e на перенос едет вместе с ним.

- [ ] 2.1 Проверить, что сервисный аккаунт читает `ReccuringInstance` из `PersonSpace` участников — проверка: пробный запрос из `services/love` возвращает документы override.
- [ ] 2.2 `meetingOccurrences` (`plugins/love/src/utils.ts`): учитывать override — время вхождения заменяется на перенесённое, `nextOccurrence` считается по новому набору.
- [ ] 2.3 `findMeetingTarget`/`resolveSession` (`services/love/src/workspaceClient.ts`): окно входа и `occurrence` сессии считаются по набору с учётом переносов.
- [ ] 2.4 Unit-тесты: перенос на часы, перенос на день, отмена + перенос, перенос без отмены — проверка: `pnpm --filter @hcengineering/love run test`.
- [ ] 2.5 Sanity: `tests/sanity/tests/love/meetings.recurring.tests.ts` — сценарий переноса вхождения.

## 3. Владелец постоянной встречи (P1)

- [x] 3.1 `server-plugins/love-resources` `OnPermanentMeetingUpdate`: правка `members` постоянной встречи без владельца исправляется обратно (`$push` отсутствующих `owners`), в лог пишется предупреждение — триггер не может отклонить транзакцию, поэтому инвариант держится ответной правкой. **Исправлено 03.11:** проверяются обе формы правки — полная перезапись `members` и `$pull` (одиночный id, массив, `{$in}`), потому что `SpaceMembers` снимает участника именно через `$pull` и раньше проходил мимо защиты; проверка: `get_errors` чистый.
- [x] 3.2 UI постоянной встречи: участников правит свой редактор `PermanentMeetingMembersEditor` (`plugins/love-resources`), поставленный через `createAttributePresenter` на `love.class.PermanentMeeting, 'members', 'array'`: владельцы в `protectedAccounts` (снять нельзя), список доступен только владельцу. Общий `SpaceMembersEditor` из `contact-resources` не тронут — он общий для каналов и воронок.
  **Исправлено 03.11:** `createAttributePresenter` действует на весь класс, поэтому тот же редактор вызывался ячейкой таблицы `Table.svelte` без `object` — падало `Cannot read properties of undefined (reading 'owners')`. Теперь `object` необязателен: без документа редактор ведёт себя как общий (без защиты), с документом — защищает владельца; проверка: `get_errors` чистый, `pnpm build --to @hcengineering/love-resources` — 0 ошибок.
- [x] 3.3 Unit-тест триггера: `server-plugins/love-resources/src/__tests__/onPermanentMeetingUpdate.test.ts` — 9 кейсов (владелец возвращается, оба владельца, правка с владельцем не трогается, `$pull` одного владельца, `$pull` с `$in`, `$pull` без владельца, переименование, чужой класс и не-update, документа нет) — проверка: прогон зелёный (9/9).

## 4. Режимы гостя постоянной встречи (P1)

- [x] 4.1 `services/love/src/guests.ts` `/guestInfo`: `after` отдаётся только сессии серии; завершённая сессия постоянной встречи даёт `before`, потому что сама встреча не заканчивается — проверка: `get_errors` чистый.
- [x] 4.2 Тест: `services/love/src/__tests__/guests.test.ts` — «завершённая сессия серии → `after`» и «завершённая сессия постоянной встречи → никогда `after`» — проверка: прогон зелёный (18/18 в файле).

## 5. Навигатор (P2)

- [x] 5.1 `navigateToOfficeDoc` и `backOrOpen` не режут `loc.path` до трёх сегментов — сделано в `audit-pr-428.md` §12 (`plugins/love-resources/src/utils.ts`, `RoomPopup.svelte`).
- [ ] 5.2 Проверить сценарий «в звонке → протокол → назад»: сегмент навигатора сохраняется — ручная проверка на стенде.
- [ ] 5.3 `MeetingMinutesSection`: в шапке одна кнопка настроек — проверка: ручная проверка на стенде.

## 6. Модель: соответствие PR #428

> **Пересмотрено 2026-11-03.** Ранее группа требовала удалить `linkId` и `TMeetingSchedule.meetingId`. Решение
> изменено: следуем концепции PR #428 и модель не переписываем — поля возвращены точно как в PR, а расхождение
> между документацией и поведением закрывается записью `linkId`, а не удалением поля.

- [x] 6.1 Модель и типы приведены к состоянию PR #428: `linkId` в миксине (с `@Index`) и в `PermanentMeeting`, `TMeetingSchedule.meetingId` — на месте; контроль: `git diff 46537cd2c8 -- models/love/src/index.ts plugins/love/src/types.ts` не содержит удалений.
- [x] 6.2 `docs/love.md` §2.5: строка `linkId` возвращена и честно помечена — «не заполняется, открытый вопрос `audit-pr-428.md` §11.3»; §2.6 снова перечисляет `linkId` в составе `PermanentMeeting`.
- [x] 6.3 Пересобраны `model-love`, `love`, `pod-love`, `love-resources` — 0 ошибок.
- [x] 6.4 Запись `linkId` при выдаче ссылки: `WorkspaceClient.setMeetingLinkId(target, shortId)` пишет shortId в документ постоянной встречи или в миксин мастера (`updateMixin`), пропускает запись, если shortId не изменился; `GET /meetingLink` вызывает его после выдачи, и сбой записи только логируется, чтобы не превращать выданную ссылку в ошибку. `MeetingTarget.link` теперь несёт `linkId`. Тест: `workspaceClient.test.ts` — 4 кейса (серия, постоянная встреча, без изменений, после отзыва) — прогон `services/love` 236/236.

## 7. Гость: описание существующего поведения (P3)

- [x] 7.1 Пароль гостя, rate limit, авторизация `/meetingLink`, `/meetingPassword`, `/resolveSession` описаны требованиями (`specs/guest-access`) — проверка: `openspec validate --all --strict`.
- [x] 7.2 Негативные тесты API. Покрыто существующими: отзыв ссылки (`meetingLink.test.ts`), истечение `afterTtl`, rate limit и сброс (`guestPassword.test.ts`). **Решение 2026-11-03: `authorizeTarget` остаётся в `main.ts` как в PR, тест на него не пишем** — вынос гейта в отдельный модуль был рефакторингом файла из слияния без согласования, он откачен (`linkAuth.ts` и его 7 тестов удалены).

## 8. RSVP (P3)

- [x] 8.1 `EventRsvp.svelte`: `pending` считается по id персоны, организатор не попадает в приглашённые — сделано в §12.
- [x] 8.2 Сводка RSVP. Фикс организатора в `EventRsvp.svelte` остался как в слиянии (инлайн-фильтр). **Решение 2026-11-03: хелпер `rsvpInvitedCount` откачен** — вынос был рефакторингом ради теста, он трогал файл из слияния. Покрытие сводки — существующие тесты `plugins/calendar/src/__tests__/rsvp.test.ts` (`collectRsvp`, `rsvpPending`).

## 9. Документация (P3)

- [x] 9.1 `docs/love.md`: обе картинки (`images/love-permanent-create.png`, `images/love-meeting-link.png`) и авторская формулировка `meetingAccess.start` возвращены в виде слияния — **решение 2026-11-03: текст и ссылки PR не переписываем**. Самих файлов картинок в репозитории пока нет — их снимаем со стенда в 11.4.
- [ ] 9.2 `virtual-office.mdx` (en, ru) в `platform-docs`: навигатор, служебный этаж, «Forever», ссылка постоянной встречи. **Заблокировано:** репозитория `platform-docs` нет рядом с рабочим деревом, править нечего.
- [x] 9.3 `docs/features/office-meetings.md`: строка пользовательской документации `communication/invite-link` перепроверена и исправлена в задаче 1.2 — причина `stale` больше не ссылается на `startOnLink`.

## 10. Сквозная проверка (P3)

- [x] 10.1 Unit всех затронутых пакетов на 03.11: `services/love` 232/232, `plugins/love` 45/45, `models/love` 9/9, `calendar` 83/83, `calendar-resources` 7/7, `server-plugins/love-resources` 40/40, `plugins/love-resources` 32/32 — всё зелёное. Сборки `pod-love`, `love-resources`, `model-love` — 0 ошибок.
- [x] 10.2a Сборка фронта падала на `@hcengineering/love-assets`: в `plugins/love-assets/src/index.ts` не было ассетов `Knock` и `StopRecord`, которые PR добавил в `love.icon`. При cherry-pick файл остался в develop-версии (общий спрайт `@hcengineering/ui/assets/icons.svg`). Добавлены `Knock → #door-knock`, `StopRecord → #square-fill` — собственный спрайт `love-assets/assets/icons.svg` из PR в ветке не используется. Проверка: `pnpm build --to @hcengineering/love-assets` — 0 ошибок.
- [ ] 10.2 Sanity: `meetings.recurring`, `meetings.permanent`, `calendar-rsvp`, `meetings.guest`. **Передано CI (решение 2026-11-03):** набор работает с отдельным стендом `tests/docker-compose.yaml` (проект `sanity`, порт 8083, воркспейс `sanity-ws`), которого нет на dev-стенде (8087); подъём требует образов всех 38 подов и ~4–6 ГБ памяти, на машине свободно около 3 ГБ. Проверка фичи до прогона — вручную на dev-стенде (5.2–5.3 и гостевая ссылка).
- [ ] 10.3 Отчёт в `handoff.md`: что сделано, что осталось, состояние стенда.

## 11. Storybook: экраны фичи (P2, запрошено 2026-11-03)

Всё, что относится к встрече и её ссылке, должно быть в Storybook в том же стиле, что там уже есть —
экраны рисуются кодом через `kit.*`, а не вставляются картинками.

- [x] 11.1 Набор экранов `dev/design/screens/love-meeting-link/`: `_meeting.js` (данные + блоки `linkMemberRow`/`linkSetting`/`linkField`), `01-meeting-link-popup.js` (панель события + попап ссылки: кто открывает сессию, срок жизни, `past`, пароль, копирование, отзыв), `02-permanent-meeting.js` (ссылка, участники с защищённым владельцем, прошедшие сессии), `03-guest-lobby.js` (три состояния: пароль, до начала, идёт) — проверка: все три набора исполняются рекордером без ошибок.
- [x] 11.2 Зарегистрировано в `dev/design/design.json` отдельной секцией `"storybook": "Meetings"` (не `Screens`: совпадение названия с `Intabia Screens` давало одинаковый slug и перезаписывало чужие истории); `node dev/storybook/scripts/generate-screens.mjs` создал `meetings-page-1..3.stories.ts` — проверка: `Meetings page-1..3` в выводе генератора.
- [x] 11.3 Стиль: `kit.Workbench`/`kit.PanelHeader`/`kit.Modal`/`kit.Table`/`kit.Switcher`-замена на `kit.Radio`, лобби — на логин-ките (`kit.LoginPage`/`LoginCard`/`LoginField`/`LoginButton`), подписи русские, один фрейм на состояние.
- [x] 11.3a `dev/storybook/scripts/generate-screens.mjs`: абсолютный путь в `await import(...)` заменён на `pathToFileURL(...).href` — на Windows генератор падал с `ERR_UNSUPPORTED_ESM_URL_SCHEME` независимо от наших экранов.
- [ ] 11.4 Скриншоты «как фича выглядит» — снять со стенда на демо-данных (`docs-media`-скилл) после деплоя PR; положить рядом со страницами документации и сослаться из `docs/love.md` вместо удалённых `images/love-*.png`. **Заблокировано:** PR не задеплоен на стенд.
- [x] 11.5 Проверка: `cd dev/storybook && pnpm run build-storybook` собирается, в `storybook-static/index.json` есть «Ссылка на встречу», «Постоянная встреча», «Гостевой лобби» (4 записи: 3 фрейма + кanвас) — проверка пройдена.

## 12. Следующая итерация: планировщик — перенесено

Раздел закрыт 2026-11-06: обе корректировки (постоянная встреча из планировщика и появление встречи из планировщика
во вкладке «Встречи») переехали в изменение `openspec/changes/love-recurring-meetings/`, группы 1–2. Здесь они
больше не ведутся — статус смотреть там.

Факты, которые сюда вернулись из кода ветки (коммиты `a579404650`, `e2da22dca2`), чтобы не искать их заново:

- путь формы события уже создаёт `PermanentMeeting` и хранит связь в `MeetingEventLink.permanentMeeting`;
- миксин `MeetingEventLink` объявлен строго на `calendar.class.Event`, на `calendar.class.Schedule` его не навесить;
- вкладка «Встречи» (`love.viewlet.CalendarScheduledMeetings`) строится по миксину, постоянные встречи — отдельной
  вкладкой `officeView === 'permanent'`.
