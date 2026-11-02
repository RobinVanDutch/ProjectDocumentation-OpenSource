# Proposal

> **Статус: поглощено PR #428 (2026-11-02).** Эта изменение описывает ветку `FUSIO-1307-livekit-shared-link` с полем
> `MeetingMinutes.startOnLink` и перезапуском встречи из `Finished`. В PR #428 (`FUSIO-1307: Recurrent meetings`)
> такой модели нет: статус `MeetingStatus.Scheduled` удалён, перезапуска нет (сессия создаётся входом),
> а право гостя открыть сессию задаётся `meetingAccess.start = 'members' | 'link'`. Если PR вливается в develop,
> ветку и это изменение закрываем; остаток задач — в `audit-pr-428.md` §8.

## Why

Гостевая ссылка на встречу перестаёт работать, как только встреча выходит из окна расписания (15 минут до старта и 4 часа после) или завершается: `/guestJoin` отвечает 403 «Meeting has not started yet» / «Meeting has already finished», а если LiveKit-комната уже пуста — 404. Хост, опоздавший на 10 минут или перезвонивший через час, вынужден создавать новую встречу и рассылать новую ссылку.

## What Changes

- `/guestJoin` больше не отбивает вход по статусу: встреча в `Scheduled` или `Finished` перезапускается (`WorkspaceClient.restartMeeting` - `status: Active`, сброс `meetingEnd`, `transcriptionState`/`recordingState` в `NotStarted`).
- Отсутствующая LiveKit-комната создаётся на входе гостя так же, как это уже делает `/getToken` для участников.
- Гостевой лобби (`GuestMeetingApp.svelte`) вместо тупиковых экранов «Завершена» / «Ждём хоста» показывает форму входа и строку статуса.
- `meetingId` и ссылка не меняются: комната пересоздаётся по требованию, комната LiveKit называется `${workspace}_${meetingId}`.

## Capabilities

### New Capabilities

- `love-meeting-link`: гостевая ссылка на встречу - постоянная, вход в любое время, перезапуск сессии и пересоздание комнаты.

### Modified Capabilities

Нет: `openspec/specs/` этого проекта пока пуст (остальные изменения ведутся для другой ветки).

## Impact

- `services/love/src/guests.ts` - `/guestJoin`: перезапуск встречи и создание комнаты.
- `services/love/src/workspaceClient.ts` - новый `restartMeeting`.
- `plugins/love-resources/src/components/guest/GuestMeetingApp.svelte` - лобби без тупиков.
- `services/love/src/__tests__/guests.test.ts` - ожидания приведены к новому контракту.
- Документация: `docs/love.md` §8.3, `docs/features/office-meetings.md`.
- Участники заходят в ту же сессию без изменений: `createMeetingDocument` переиспользует живую встречу комнаты (`notMatch` + поиск `Active`/`Pending`), поэтому LiveKit-комната у гостя и участника совпадает.
