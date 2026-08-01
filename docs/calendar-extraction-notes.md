# Calendar Extraction Notes

## Orientation: where the business logic lives

Cal.diy is a Yarn/Turbo monorepo. The calendar-domain logic is concentrated outside the web apps, mainly in:

- `packages/features/*`: feature services and domain orchestration for availability, bookings, busy-time calculation, schedules, calendars, event types, and conferencing.
- `packages/lib/*`: shared calendar/date/time utilities, CalDAV/ICS base calendar service code, builders, and formatters.
- `packages/types/*`: shared calendar, event, and schedule contracts.
- `packages/app-store/*calendar*/lib`: concrete calendar provider adapters such as Google, Office 365, CalDAV, Apple, Exchange, ICS feed, Lark, Feishu, and Zoho.
- `packages/prisma/schema.prisma`: persistence model source for bookings, availability rows, schedules, selected calendars, credentials, and event types. This is schema input, not the engine itself.
- `packages/trpc/*` and `apps/*`: API and frontend surfaces that call the domain layer; useful for integration context, but not the extraction target.

Ignore for Proxy extraction: docs/marketing/web pages, dashboards, UI components, pricing, billing/payment flows, auth pages, organization/team SaaS administration, and most CRUD repositories.

## Providers

| Component | Keep? | Why? | Proxy Equivalent |
| --- | --- | --- | --- |
| Shared calendar provider interface (`Calendar`) | ✅ | Defines the portable provider contract for create/update/delete events, free/busy, timezone-aware availability, cache warmup, and calendar listing. | `CalendarProvider` trait/interface |
| `packages/lib/CalendarService.ts` base CalDAV/ICS service | ✅ | Provides reusable CalDAV event creation/update/deletion, ICS generation, VTIMEZONE injection, travel-time handling, recurrence expansion, and busy extraction. | `CalDavProviderBase`, `ICSParser`, `ICSExporter` |
| Google calendar service | ✅ | First-party concrete adapter with OAuth, free/busy, event CRUD, calendar listing, Google Meet/conference fields, and calendar timezone handling. | `GoogleCalendarProvider` |
| Office 365 calendar service | ✅ | Microsoft Graph adapter for event CRUD, schedule/free-busy lookup, and event translation. | `OutlookCalendarProvider` |
| CalDAV / Apple / Exchange services | ✅ | Non-Google provider implementations; Apple and CalDAV subclass the common CalDAV base. | `CalDavProvider`, `AppleCalendarProvider`, `ExchangeProvider` |
| ICS feed calendar service | ✅ | Read-only feed provider that imports busy intervals from `.ics`/iCalendar sources. | `ICSFeedProvider` |
| Lark / Feishu / Zoho services | ✅ | Additional provider mapping examples if Proxy wants extensible adapter patterns. | Optional provider packs |
| App store UI/settings pages | ❌ | SaaS integration management, not calendar-engine behavior. | None |

## Scheduling

| Component | Keep? | Why? | Proxy Equivalent |
| --- | --- | --- | --- |
| Schedule and working-hours models | ✅ | Express weekly availability as time ranges and working hours in minutes since midnight. | `WorkingHours`, `WeeklySchedule` |
| Schedule-to-working-hours conversion | ✅ | Converts stored UTC availability rows to timezone-relative working-hour windows and handles day overflow. | `ScheduleNormalizer` |
| Schedule services/repositories | ⚠️ Partial | Useful for understanding availability persistence, but database/repository details are Cal.diy-specific. | SQLite structs or local config store |
| Booking pages and schedule editors | ❌ | Frontend SaaS surfaces. | Desktop UI later |

## Availability

| Component | Keep? | Why? | Proxy Equivalent |
| --- | --- | --- | --- |
| User availability aggregation | ✅ | Combines schedule availability, date overrides, busy intervals, seats, limits, and final timezone into the data needed for slot finding. | `AvailabilityEngine` |
| Aggregated/team availability | ✅ | Explains multi-host/team availability composition. | `GroupAvailabilityEngine` |
| Busy-time service | ✅ | Merges existing bookings, buffers, calendars, limits, and video busy times into busy intervals. | `BusyTimeCollector` |
| Booking limit busy-time logic | ✅ | Converts period/duration/team limits into synthetic busy intervals. | `BookingLimitRule` |
| No-slots notifications | ❌ | SaaS notification policy; keep only the event that no slots were found if Proxy needs it. | Optional notification hook |

## Events

| Component | Keep? | Why? | Proxy Equivalent |
| --- | --- | --- | --- |
| `CalendarEvent`, `Person`, `TeamMember`, `RecurringEvent`, `NewCalendarEventType` | ✅ | Core event/attendee/recurrence/provider-result contracts. | `CalendarEvent`, `Attendee`, `Organizer`, `RecurrenceRule`, `ProviderEventId` |
| Calendar event builder/formatter/parser | ✅ | Builds event payloads and rich descriptions for providers and ICS. | `CalendarEventBuilder`, `EventDescriptionRenderer` |
| Booking audit/history/payment/event reports | ❌ | SaaS operations around bookings. | None |
| Event type UI and CRUD | ⚠️ Partial | Contains business rules like buffers, notice, future limits, locations, seats; ignore CRUD/UI. | `MeetingTypePolicy` |

## Timezone

| Component | Keep? | Why? | Proxy Equivalent |
| --- | --- | --- | --- |
| IANA timezone validation and helpers | ✅ | Validates and formats timezone values and offsets. | `TimeZoneId`, `TimeUtils` |
| Timezone-aware working-hours conversion | ✅ | Required to map weekly availability across user/visitor timezones and DST boundaries. | `WorkingHoursProjector` |
| VTIMEZONE generation/injection | ✅ | Required for interoperable ICS/CalDAV events with local times and DST transitions. | `VTimezoneBuilder` |
| Timezone dropdown/UI labeling | ❌ | UI concern. | None |

## Recurrence

| Component | Keep? | Why? | Proxy Equivalent |
| --- | --- | --- | --- |
| `RecurringEvent` model and RRULE fields | ✅ | Captures frequency, interval, count, until, and timezone ID. | `RecurrenceRule` |
| ICS recurrence expansion in calendar services | ✅ | Expands recurring VEVENTs into concrete busy intervals and preserves recurrence identifiers/timezones. | `RecurrenceExpander` |
| Unsupported recurrence telemetry/logging | ⚠️ Partial | Useful edge-case inventory; implementation logging is not core. | Validation warnings |

## ICS

| Component | Keep? | Why? | Proxy Equivalent |
| --- | --- | --- | --- |
| ICS creation/update via `ics` package | ✅ | Generates provider-neutral iCalendar events for CalDAV-like backends. | `ICSExporter` |
| ICS parsing via `ical.js` | ✅ | Extracts busy intervals, all-day events, travel duration, recurrence, timezone metadata. | `ICSImporter` |
| RFC line folding and schedule-agent injection | ✅ | Interoperability details needed by CalDAV clients/servers. | `ICSCompatibility` |

## Conferencing

| Component | Keep? | Why? | Proxy Equivalent |
| --- | --- | --- | --- |
| Conference data in calendar event model | ✅ | Calendar events can carry generated meeting links and provider conference metadata. | `ConferenceDetails` |
| Video provider integrations | ⚠️ Partial | Keep abstraction/attachment points; provider-specific SaaS marketplace code can become optional packs. | `ConferencingProvider` capability pack |
| Video busy times | ✅ | A meeting provider can contribute busy intervals even when calendar is not the source. | `ExternalBusyProvider` |

## Notifications

| Component | Keep? | Why? | Proxy Equivalent |
| --- | --- | --- | --- |
| Calendar invitation behavior | ✅ | Provider event creation/update/deletion often determines attendee notifications. | Provider notification options |
| Email templates and org admin notifications | ❌ | Product/SaaS behavior, not calendar extraction. | Optional email capability pack |

## Utilities

| Component | Keep? | Why? | Proxy Equivalent |
| --- | --- | --- | --- |
| Date/time utilities using `dayjs`, native `Date`, and date-fns-style operations | ✅ | Supports timezone projection, interval math, sorting, formatting, and validation. | `utils/time` |
| Conflict checker | ✅ | Small calendar-engine algorithm for detecting interval overlap against busy times, with seat exceptions. | `ConflictDetector` |
| PII-free logging helpers | ⚠️ Partial | Security practice worth preserving, implementation is app-specific. | Redaction utility |
| Stripe/payment/auth utilities | ❌ | SaaS concerns. | None |

## Database

| Component | Keep? | Why? | Proxy Equivalent |
| --- | --- | --- | --- |
| Prisma models for `Booking`, `Availability`, `Schedule`, `SelectedCalendar`, `DestinationCalendar`, `Credential`, `EventType` | ⚠️ Partial | Important for data shape discovery, but not the extraction target. | SQLite/domain structs |
| Prisma repositories | ❌ | Persistence mechanics and SaaS data access. | Repository adapters only if needed |

## API

| Component | Keep? | Why? | Proxy Equivalent |
| --- | --- | --- | --- |
| tRPC/API route contracts for slots, bookings, calendars, schedules | ⚠️ Partial | Useful to infer inputs/outputs for tools, but transport is not core. | Tool schemas / command inputs |
| Next.js pages and API plumbing | ❌ | Product implementation detail. | None |

## Frontend

| Component | Keep? | Why? | Proxy Equivalent |
| --- | --- | --- | --- |
| Weekly calendar overlap layout utilities | ⚠️ Partial | UI-oriented, but overlap grouping can inspire display later. Not required for engine extraction. | Desktop UI layout helper later |
| Booker, dashboards, React components, Tailwind styles | ❌ | SaaS/web UI. | None |

## Core Models to Extract

- `CalendarAccount`: connected provider credential metadata without secrets, selected calendars, destination calendar, provider type.
- `Calendar`: id, external id, name, provider integration, read-only/primary flags.
- `CalendarEvent`: title, type, start/end, organizer, attendees, location, description, conferencing data, recurrence, sequence/UID/iCalUID, visibility flags.
- `Attendee` / `Organizer`: name, email, timezone, locale/language, optional phone/user id.
- `BusyTime`: start, end, source, timezone, optional title/user id.
- `Availability`: weekly day/time availability and date overrides.
- `WorkingHours`: day indexes plus start/end minutes since midnight, optionally per user.
- `Schedule`: weekly `TimeRange[][]` plus travel schedule/date override concepts.
- `RecurrenceRule`: freq, interval, count, until, dtstart, tzid.
- `MeetingTypePolicy`: duration, buffers, seats, minimum notice, maximum future booking, period/duration/team booking limits.

## Core Algorithms to Extract

### Availability Engine

Purpose: find candidate meeting slots.

Inputs:

- Participants/accounts.
- Duration.
- Date range.
- Weekly working hours and date overrides.
- Visitor/organizer timezone.
- Busy intervals from calendars, existing bookings, limits, seats, and video providers.
- Event policy: buffers, minimum notice, maximum future booking, seats, recurring restrictions.

Outputs:

- Candidate available time slots.
- Diagnostic busy/blocked intervals and reasons.

Dependencies:

- `CalendarProvider` free/busy calls.
- `WorkingHoursProjector`.
- `BusyTimeCollector`.
- `ConflictDetector`.
- Recurrence and timezone utilities.

### Busy-Time Collection

Purpose: normalize every block on a calendar into comparable busy intervals.

Inputs: selected calendars, local bookings, buffers, reschedule UID, booking/video/limit constraints.

Outputs: `BusyTime[]` sorted/normalized by UTC instants with source metadata.

### Conflict Detection

Purpose: decide whether a proposed slot overlaps busy intervals.

Inputs: proposed start, duration, busy intervals, optional seat availability.

Outputs: boolean conflict decision and, in Proxy, ideally a conflict reason.

### ICS Import/Export

Purpose: bridge provider-neutral events to iCalendar.

Inputs: `CalendarEvent`, timezone, recurrence, attendee/organizer data, provider calendar object.

Outputs: ICS payloads for create/update and parsed busy intervals/provider events.

### Recurrence Expansion

Purpose: turn recurring events into concrete intervals over a query window.

Inputs: recurrence rule, DTSTART/DTEND, VTIMEZONE/TZID, query range.

Outputs: generated occurrences with recurrence IDs and timezone-correct start/end.

## Provider Layer Mapping for Proxy

| Cal.diy Concept | Proxy Equivalent |
| --- | --- |
| `Calendar` interface | `CalendarProvider` trait |
| `BaseCalendarService` | `CalDavProviderBase` plus `ICSImporter`/`ICSExporter` |
| Google CalendarService | `GoogleCalendarProvider` |
| Office365 CalendarService | `OutlookCalendarProvider` |
| CalDAV/Apple/Exchange services | `CalDavProvider`, `AppleCalendarProvider`, `ExchangeProvider` |
| ICS feed service | `ICSFeedProvider` |
| `getUserAvailability` | `FindAvailabilityTool` / `AvailabilityEngine` |
| `getBusyTimesService` | `BusyTimeCollector` |
| `checkForConflicts` | `ConflictDetector` |
| `getWorkingHours` | `WorkingHoursProjector` |
| Calendar event builders/parsers | `CalendarEventBuilder`, `EventDescriptionRenderer` |
| Notifications/emails | Email capability pack, optional |

## Extraction Inventory

| Component | Keep? | Why? | Proxy Equivalent |
| --- | --- | --- | --- |
| Calendar Provider | ✅ | Core abstraction over providers. | `CalendarProvider` |
| Availability Engine | ✅ | Finds bookable meeting slots from schedules and busy intervals. | `FindAvailabilityTool` / `AvailabilityEngine` |
| Busy-Time Collector | ✅ | Normalizes booking, calendar, limit, and video busy blocks. | `BusyTimeCollector` |
| Conflict Checker | ✅ | Detects interval overlap, with seats exception. | `ConflictDetector` |
| Timezone Utils | ✅ | Universal date/time conversion and validation. | `utils/time` |
| Working Hours | ✅ | Core availability constraint. | `WorkingHours` / `ScheduleNormalizer` |
| Recurrence / RRULE | ✅ | Calendar-native repeated event behavior. | `RecurrenceRule`, `RecurrenceExpander` |
| ICS / CalDAV | ✅ | Provider-neutral import/export and interoperability. | `ICSImporter`, `ICSExporter`, `CalDavProviderBase` |
| Google/Outlook/CalDAV/Apple providers | ✅ | Concrete adapters. | Provider packs |
| Conferencing hooks | ⚠️ Partial | Calendar event metadata and busy contribution only. | `ConferencingProvider` pack |
| Prisma Models | ⚠️ Partial | Data-shape reference; not runtime architecture. | SQLite structs/domain store |
| Booking Pages | ❌ | Web UI/product flow. | None |
| Stripe Billing | ❌ | SaaS monetization. | Polar/payment pack if needed |
| Auth/Organizations/Dashboards | ❌ | SaaS concerns. | Runtime permissions or none |
| React Components | ❌ | Frontend implementation. | Desktop UI later |

## Recommended Proxy Architecture

1. `core/models`: calendar accounts, calendars, events, attendees, busy intervals, working hours, schedules, recurrence rules, meeting policies.
2. `core/time`: timezone validation/projection, interval math, DST-safe local/UTC conversions.
3. `core/ics`: ICS import/export, VTIMEZONE, recurrence expansion, CalDAV compatibility helpers.
4. `core/availability`: working-hours normalization, busy-time merge, conflict detection, slot search, group availability.
5. `providers`: Google, Outlook, CalDAV, Apple, Exchange, ICS feed; each implements `CalendarProvider`.
6. `tools`: `FindAvailabilityTool`, `ScheduleMeetingTool`, `UpdateMeetingTool`, `CancelMeetingTool`, `ImportCalendarTool`.
7. `packs`: optional conferencing and email/notification packs.


---

# Detailed Calendar Engine Extraction

## 1. Core Domain Models

The calendar engine's reusable domain model is defined by shared type contracts plus several persistence-backed shapes that should be re-modeled independently of Prisma.

| Model | Responsibility | Key fields | Relationships | Used by |
| --- | --- | --- | --- | --- |
| `Person` | Represents an organizer, attendee, host, or team member participating in an event. | `name`, `email`, `timeZone`, `language`, optional `id`, `username`, `locale`, `timeFormat`, `phoneNumber`, `bookingSeat`. | Embedded in `CalendarEvent.organizer`, `CalendarEvent.attendees`, and team members. | Event builders, provider translation, emails, calendar descriptions. |
| `TeamMember` | Lightweight host participant for team events. | `id`, `name`, `email`, `phoneNumber`, `timeZone`, `language`. | Nested in `CalendarEvent.team.members`. | Team calendar event generation and host metadata. |
| `CalendarEvent` | Provider-neutral event payload used for meeting creation, update, cancellation, conferencing, recurrence, and attendee details. | `type`, `title`, `startTime`, `endTime`, `organizer`, `attendees`, `location`, `description`, `uid`, `bookingId`, `destinationCalendar`, `recurrence`, `recurringEvent`, `iCalUID`, `iCalSequence`, `conferenceData`, `videoCallData`, `responses`, seats flags, cancellation/reschedule flags. | Contains `Person`, `TeamMember`, `DestinationCalendar`, `RecurringEvent`, `ConferenceData`, `VideoCallData`, app status, booking metadata. | `EventManager`, `CalendarManager`, every provider `CalendarService`, ICS generation, booking creation/reschedule/cancel flows. |
| `CalendarServiceEvent` | Calendar-provider event with rendered calendar description. | All `CalendarEvent` fields plus `calendarDescription`. | Produced by `processEvent()` before provider calls. | Provider `createEvent` / `updateEvent`. |
| `NewCalendarEventType` | Normalized provider result after creating or updating a calendar event. | `uid`, `id`, `type`, `password`, `url`, `additionalInfo`, optional `iCalUID`, `location`, `hangoutLink`, `conferenceData`, `thirdPartyRecurringEventId`, `delegatedToId`. | Becomes `EventResult.createdEvent` / `updatedEvent`, then booking references. | `CalendarManager`, `EventManager`, booking reference persistence. |
| `EventBusyDate` / `EventBusyDetails` | Busy interval returned by calendar providers and internal busy-time collectors. | `start`, `end`, optional `source`, `timeZone`; details add optional `title`, mandatory `source`, optional `userId`. | Subtracted from `DateRange[]`; merged with booking-limit and team-limit busy intervals. | `BusyTimesService`, `UserAvailabilityService`, `Calendar.getAvailability`, conflict checker. |
| `RecurringEvent` | Structured recurrence rule fields. | `dtstart`, `interval`, `count`, `freq`, `until`, `tzid`. | Referenced by `CalendarEvent.recurringEvent`; raw RRULE string may be on `CalendarEvent.recurrence`. | Event creation, provider translation, recurrence expansion/parsing. |
| `IntegrationCalendar` / `SelectedCalendar` | Calendar account/calendar selection shape used for free-busy queries and destination selection. | `externalId`, `integration`, optional `primary`, `name`, `readOnly`, `email`, `credentialId`, `customCalendarReminder`, selected-calendar event type ids. | Tied to credentials; passed to providers in `GetAvailabilityParams`. | Calendar listing, busy-time fetch, destination calendar selection. |
| `GetAvailabilityParams` | Provider-level free/busy request. | `dateFrom`, `dateTo`, `selectedCalendars`, `mode`, optional `fallbackToPrimary`. | Consumed by every provider's `getAvailability`. | `CalendarManager.getCalendarsEvents`, provider implementations. |
| `DateRange` | Available or unavailable interval in the availability engine. | `start`, `end` as `Dayjs`. | Source ranges are subtracted by busy ranges and intersected across users. | `buildDateRanges`, `subtract`, `intersect`, `getAggregatedAvailability`, slot generation. |
| `WorkingHours` | Weekly availability rule from stored availability rows. | `days`, `startTime`, `endTime`; in shared schedule type, times are minutes since midnight; in schedule lib they are `Date` times. | Converted from availability rows and projected into concrete `DateRange`s. | `getWorkingHours`, `processWorkingHours`, `buildDateRanges`. |
| `DateOverride` | One-off availability override. | `date`, `startTime`, `endTime`. | Overrides/augments weekly working hours for specific dates. | `processDateOverride`, `buildDateRanges`, `UserAvailabilityService`. |
| `TravelSchedule` | Temporary timezone override for a user's default schedule. | `id`, `timeZone`, `userId`, `startDate`, `endDate`, `prevTimeZone`. | Adjusts timezone used when building date ranges. | `getAdjustedTimezone`, `processWorkingHours`, `processDateOverride`. |
| `GetSlots` / slot result | Turns available ranges into bookable slot start times. | Inputs: `inviteeDate`, `frequency`, `dateRanges`, `minimumBookingNotice`, `eventLength`, optional `offsetStart`, OOO data, optimized slots flag. Output: `time`, optional `userIds`, OOO metadata. | Consumes `DateRange[]` after busy subtraction and team aggregation. | `packages/features/schedules/lib/slots.ts`. |
| `EventResult<T>` | Result envelope for calendar/video/CRM provider operations. | `type`, `appName`, `success`, `uid`, optional `iCalUID`, `createdEvent`, `updatedEvent`, `originalEvent`, `calError`, `calWarnings`, credential/destination ids. | Converted to `PartialReference`. | `EventManager`, booking flow. |
| `PartialReference` | Provider reference stored on bookings. | `type`, `uid`, optional `meetingId`, `meetingPassword`, `meetingUrl`, `thirdPartyRecurringEventId`, `externalCalendarId`, `credentialId`, `delegationCredentialId`. | Points from a booking to third-party provider records. | Reschedule, update, cancel, calendar deletion. |
| `MeetingTypePolicy` (derived from `EventType`) | Booking policy object to extract from app-specific event-type schema. | Duration, buffers, seats per slot, selected calendars mode, booking/duration limits, minimum booking notice, minimum reschedule notice, max future booking, recurrence. | Feeds availability, conflict checks, and booking creation. | `UserAvailabilityService`, `BusyTimesService`, `RegularBookingService`, limit utilities. |

## 2. Availability Calculation Algorithms and Execution Flow

### Files involved

- `packages/features/availability/lib/getUserAvailability.ts`
- `packages/features/availability/lib/getAggregatedAvailability/getAggregatedAvailability.ts`
- `packages/features/availability/lib/getAggregatedAvailability/date-range-utils/mergeOverlappingDateRanges.ts`
- `packages/features/availability/lib/getAggregatedAvailability/date-range-utils/filterRedundantDateRanges.ts`
- `packages/features/availability/lib/detectEventTypeScheduleForUser.ts`
- `packages/features/availability/lib/calculateHolidayBlockedDates.ts`
- `packages/features/availability/lib/findUsersForAvailabilityCheck.ts`
- `packages/features/busyTimes/services/getBusyTimes.ts`
- `packages/features/busyTimes/lib/getBusyTimesFromLimits.ts`
- `packages/features/schedules/lib/date-ranges.ts`
- `packages/features/schedules/lib/slots.ts`
- `packages/lib/availability.ts`
- `packages/features/bookings/lib/conflictChecker/checkForConflicts.ts`
- `packages/features/calendars/lib/CalendarManager.ts`
- Provider `CalendarService.ts` files under calendar app-store packages.

### Complete flow: inputs to outputs

1. **Input normalization**: availability starts with user/event inputs: user identity, date range, event type id, duration, buffers, selected calendar mode, reschedule UID, return-date-overrides flag, calendar-fetch mode, and failure behavior.
2. **Event type and seats**: the service loads the event type if not provided, parses metadata, and fetches current seat counts when `seatsPerTimeSlot` is enabled.
3. **Schedule selection**: `detectEventTypeScheduleForUser` chooses whether to use event-type schedule, event-type availability, user schedule, or user availability.
4. **Timezone resolution**: if schedule timezone is not set, delegated Google/Outlook calendar credentials may be queried for the user's main calendar timezone and cached; otherwise the schedule timezone is used.
5. **Working-hours projection**: `getWorkingHours` converts stored availability rows into timezone-relative weekly working hours; `buildDateRanges` expands weekly rules and date overrides into concrete `DateRange[]` over the request window, with travel-schedule timezone adjustments and out-of-office exclusions.
6. **Out-of-office and holidays**: OOO ranges and holiday blocked dates are converted into unavailable day ranges and excluded from the returned OOO-aware ranges.
7. **Booking and duration limits**: limit schemas are parsed, then `getBusyTimesFromLimits` and team-limit logic convert reached limits into synthetic busy intervals.
8. **Busy-time collection**: `BusyTimesService.getBusyTimes` collects existing Cal bookings, buffer intervals, connected calendar free/busy results, and optional video busy times.
9. **Busy subtraction**: detailed busy intervals from calendars/bookings/limits are converted to `Dayjs` ranges and subtracted from available date ranges via `subtract`.
10. **Aggregation for team events**: `getAggregatedAvailability` intersects fixed/collective hosts and round-robin groups, merges overlapping ranges, and filters redundant ranges.
11. **Slot generation**: `getSlots` turns available `DateRange[]` into candidate slot start times by applying frequency, minimum booking notice, event duration, offset, timezone-local rounding, and OOO annotations.
12. **Conflict validation**: booking creation re-checks requested slots against busy data and booking constraints before persisting and creating provider events.

### Outputs

- Per-user availability result: busy details, final timezone, available date ranges, OOO-excluded ranges, working hours, date overrides, current seats, and OOO metadata.
- Aggregated availability result: merged/intersected date ranges for the event/team strategy.
- Slots result: ordered unique candidate slot start times with optional OOO metadata.

### Key dependencies

- Data repositories: event type, OOO, booking, holiday.
- Calendar provider free/busy abstraction: `Calendar.getAvailability` through `CalendarManager.getBusyCalendarTimes`.
- Date utilities: dayjs timezone/UTC helpers, `buildDateRanges`, `subtract`, `intersect`.
- Policy utilities: booking limits, duration limits, buffers, seats, minimum notice, reschedule UID.

## 3. Provider Abstraction Layer

The provider abstraction is the `Calendar` interface in `packages/types/Calendar.d.ts`. Concrete services implement this interface and are dynamically loaded through `CalendarServiceMap` by `packages/app-store/_utils/getCalendar.ts`.

### Mandatory methods

| Method | Responsibility | Implemented by |
| --- | --- | --- |
| `createEvent(event, credentialId, externalCalendarId?)` | Create a third-party calendar event and return normalized `NewCalendarEventType`. | Google, Office365, CalDAV base subclasses, Exchange variants, ICS feed stub, Lark/Feishu/Zoho. |
| `updateEvent(uid, event, externalCalendarId?)` | Update an existing provider event; may return one event or multiple events. | Google, Office365, CalDAV base subclasses, Exchange variants, ICS feed stub, Lark/Feishu/Zoho. |
| `deleteEvent(uid, event, externalCalendarId?)` | Delete/cancel a provider event. | Google, Office365, CalDAV base subclasses, Exchange variants, ICS feed stub, Lark/Feishu/Zoho. |
| `getAvailability(params)` | Return busy intervals for selected calendars in the requested range. | Google, Office365, CalDAV base subclasses, Exchange variants, ICS feed. |
| `listCalendars(event?)` | Return calendars selectable by users or destination-calendar logic. | Provider services. |

### Optional methods

| Method | Responsibility | Provider-specific behavior |
| --- | --- | --- |
| `getCredentialId()` | Identify credential backing the provider. | Used where adapters need credential metadata. |
| `getAvailabilityWithTimeZones(params)` | Return busy data with timezone details for OOO/timezone calibration. | Primarily Google/Outlook-style delegated calendar flows. |
| `fetchAvailabilityAndSetCache(selectedCalendars)` | Warm subscription/cache-backed availability. | Calendar subscription/cache wrapper aware providers. |
| `testDelegationCredentialSetup()` | Validate domain-wide delegation setup. | Delegated Google/Outlook credential flows. |
| Provider-specific extras like `getMainTimeZone()` | Not in the interface, but checked dynamically by availability code. | Delegated credentials can supply a main calendar timezone. |

### Provider-specific behaviors

- **Google**: handles OAuth/delegation, Google Meet/hangout links, Google freebusy/calendar APIs, event insert/update/delete, selected calendar handling, and main-calendar timezone lookup.
- **Office 365 / Outlook**: maps events to Microsoft Graph, supports Teams behavior when Teams is represented through Outlook calendar event creation, and returns schedule availability.
- **CalDAV / Apple**: Apple and generic CalDAV subclass `BaseCalendarService`; they use CalDAV discovery, ICS creation/update, VTIMEZONE injection, and DAV object synchronization.
- **Exchange**: Exchange 2013/2016/generic adapters implement the same `Calendar` interface with Exchange-specific event and availability APIs.
- **ICS feed**: read-only provider. Create/update/delete are stubs/non-operational, while `getAvailability` parses external `.ics` feeds into busy intervals.
- **Wrappers**: `getCalendar` may wrap providers with cache and telemetry wrappers for slot-fetch mode without changing the `Calendar` contract.

## 4. Recurrence Engine

### Representation

- Structured recurrence is `RecurringEvent`: `dtstart`, `interval`, `count`, `freq`, optional `until`, optional `tzid`.
- Raw recurrence can also be carried as `CalendarEvent.recurrence`, usually an RRULE string consumed by provider translation or ICS generation.
- Provider results may store `thirdPartyRecurringEventId` and `iCalUID` for recurring series identity.
- Booking references preserve `thirdPartyRecurringEventId` so cancellation can target the whole series or specific booking occurrence as needed.

### Expansion and conversion

- ICS/CalDAV parsing in `BaseCalendarService.getAvailability` reads VEVENTs via `ical.js`, detects recurrence types, obtains `vtimezone` data, and expands recurring events across the requested window.
- Recurrence expansion is timezone-sensitive: the code builds/uses an `ICAL.Timezone` from VTIMEZONE where present, keeps recurrence IDs, and avoids mixing CalDAV/iCalendar timezone semantics.
- For create/update, calendar services generate provider payloads from `CalendarEvent.recurringEvent` or `CalendarEvent.recurrence`; for CalDAV-style providers, this is emitted into ICS payloads.
- Unsupported recurrence types are warned/skipped or handled conservatively to avoid returning incorrect busy intervals.

### Validation

- RRULE shape validation is mostly implicit through type-level contracts, provider APIs, and parser behavior rather than a dedicated recurrence validation service.
- Proxy should extract this into an explicit `RecurrenceRule` parser/validator with provider compatibility warnings.

### Relevant files

- `packages/types/Calendar.d.ts`
- `packages/lib/CalendarService.ts`
- `packages/lib/CalEventParser.ts`
- `packages/features/bookings/lib/EventManager.ts`
- `packages/features/bookings/lib/service/RegularBookingService.ts`
- Provider services: `packages/app-store/googlecalendar/lib/CalendarService.ts`, `packages/app-store/office365calendar/lib/CalendarService.ts`, `packages/app-store/caldavcalendar/lib/CalendarService.ts`, `packages/app-store/applecalendar/lib/CalendarService.ts`, `packages/app-store/exchangecalendar/lib/CalendarService.ts`, `packages/app-store/exchange2013calendar/lib/CalendarService.ts`, `packages/app-store/exchange2016calendar/lib/CalendarService.ts`, `packages/app-store/ics-feedcalendar/lib/CalendarService.ts`.

## 5. Timezone System

### Concepts

- **Stored weekly availability** is stored as UTC-time `Date` values plus day indexes, then projected into a schedule/user timezone.
- **Working hours** are normalized to minutes since midnight or concrete `DateRange`s depending on layer.
- **Date ranges** are expanded in organizer timezone but compared/subtracted as absolute instants.
- **Slot display/generation** rounds in invitee timezone so half-hour offset timezones align correctly.
- **ICS/CalDAV events** require VTIMEZONE blocks and local DTSTART/DTEND conversion for interoperability and DST correctness.

### Execution details

1. `UserAvailabilityService` picks final timezone from the schedule or delegated calendar lookup.
2. `getWorkingHours` maps UTC start/end values to timezone-relative minute ranges, splitting overflow into previous/next days where needed.
3. `buildDateRanges` expands working hours day-by-day. It adjusts for travel schedules, DST transitions, OOO days, and date overrides.
4. `getSlots` converts candidate starts to the invitee timezone before applying interval rounding and minimum notice.
5. `BaseCalendarService` builds VTIMEZONE components by detecting DST transitions and rewrites DTSTART/DTEND from UTC into timezone-local ICS properties.
6. Provider availability parsers normalize provider-native date/time and timezone data into `EventBusyDate` intervals.

### Relevant files

- `packages/lib/availability.ts`
- `packages/features/schedules/lib/date-ranges.ts`
- `packages/features/schedules/lib/slots.ts`
- `packages/features/availability/lib/getUserAvailability.ts`
- `packages/lib/CalendarService.ts`
- `packages/lib/dayjs/index.ts`
- `packages/lib/dayjs/timeZone.schema.ts`
- `packages/lib/timezone.ts`
- `packages/lib/timeZones.ts`
- `packages/lib/timezoneConstants.ts`
- `packages/lib/timeShift.ts`
- Provider `CalendarService.ts` implementations.

## 6. Busy-Time Calculation Pipeline

### Files involved

- `packages/features/busyTimes/services/getBusyTimes.ts`
- `packages/features/busyTimes/lib/getBusyTimesFromLimits.ts`
- `packages/features/calendars/lib/CalendarManager.ts`
- `packages/features/calendars/lib/getCalendarsEvents.ts`
- `packages/features/availability/lib/getUserAvailability.ts`
- `packages/features/schedules/lib/date-ranges.ts`
- `packages/features/eventtypes/lib/getDefinedBufferTimes.ts`
- Calendar provider services.

### Flow from connected calendars to returned intervals

1. **Inputs**: credentials, selected calendars, user identity, event type, date range, before/after buffers, reschedule UID, duration, seated-event flag, bypass-calendar flag, and mode.
2. **Local booking query**: existing accepted bookings owned by or attended by the user are fetched over an expanded range that includes the maximum possible buffer.
3. **Booking buffer application**: each booking interval is expanded by the existing event type's buffers plus the candidate event's opposing buffers. For example, existing `beforeEventBuffer` plus candidate `afterEventBuffer` blocks time before the existing event.
4. **Seated-event handling**: partially filled seated bookings do not block the event slot itself for the same event type, but their before/after buffers still block; full seated slots block normally.
5. **Reschedule exclusion**: the original booking UID is ignored so rescheduling to the same time can remain possible; its range can also be treated as an open seat interval for calendar subtraction.
6. **Connected-calendar fetch**: `getBusyCalendarTimes` deduplicates credentials by selected calendars, expands the date range by -11/+14 hours for UTC offset safety, and calls each provider's `getAvailability` through `getCalendarsEvents`.
7. **Open-seat subtraction**: calendar busy intervals are subtracted by partially open seated-booking ranges to avoid double-blocking slots that still have capacity.
8. **Calendar buffer application**: connected-calendar busy intervals are expanded with candidate after/before buffers.
9. **Limit busy intervals**: booking, duration, and team limits are converted into synthetic busy intervals and appended by `UserAvailabilityService`.
10. **Return**: a flat `EventBusyDetails[]` list with normalized start/end, title/source metadata, and optional user id.

## 7. Conflict Detection Logic

### Direct overlap checker

`checkForConflicts` receives busy intervals, a proposed slot start, event length, and optional current seats. Rules:

1. If the busy list is missing or empty, there is no conflict.
2. If `currentSeats` contains the exact proposed start, there is no conflict because the seated slot still has capacity.
3. Convert proposed slot to `[slotStart, slotEnd)` in UTC milliseconds.
4. Sort busy intervals by start time.
5. Stop scanning when busy start is at/after slot end.
6. Skip busy intervals ending at/before slot start.
7. Any remaining overlap is a conflict.

### Other conflict-producing rules

- Existing accepted bookings block slots, expanded by before/after buffers.
- Connected calendar busy periods block slots, expanded by candidate buffers.
- Booking and duration limits create synthetic busy intervals when limits are reached.
- Team booking limits create per-user/team synthetic busy intervals.
- OOO and holiday ranges remove availability before slot generation.
- Minimum booking notice removes slots before `now + minimumBookingNotice`.
- Event duration must fit inside a date range; slots whose end exceeds the range are not emitted.
- Reschedule UID prevents self-conflict with the booking being moved.
- Seated events may allow conflicts with partially filled slots while retaining buffer conflicts.

### Relevant files

- `packages/features/bookings/lib/conflictChecker/checkForConflicts.ts`
- `packages/features/busyTimes/services/getBusyTimes.ts`
- `packages/features/busyTimes/lib/getBusyTimesFromLimits.ts`
- `packages/features/availability/lib/getUserAvailability.ts`
- `packages/features/schedules/lib/date-ranges.ts`
- `packages/features/schedules/lib/slots.ts`
- `packages/features/bookings/lib/checkBookingLimits.ts`
- `packages/features/bookings/lib/checkDurationLimits.ts`
- `packages/features/bookings/lib/handleNewBooking/checkBookingAndDurationLimits.ts`
- `packages/features/bookings/lib/service/RegularBookingService.ts`

## 8. ICS Subsystem

### Generation

- CalDAV-style providers use `BaseCalendarService.createEvent` and `updateEvent` to call `ics.createEvent`, convert dates to iCalendar date arrays, inject attendees/organizer/details, and send the generated ICS to the DAV server.
- The subsystem folds long lines according to RFC 5545, injects `SCHEDULE-AGENT=CLIENT` for organizer/attendee properties to avoid duplicate server-side invitation emails, and removes `METHOD:PUBLISH` where CalDAV requires it.
- VTIMEZONE injection rewrites UTC `DTSTART`/`DTEND` to organizer-local `TZID` fields and inserts a generated VTIMEZONE block based on DST transition calculations.

### Parsing and synchronization

- `BaseCalendarService.getAvailability` fetches calendar objects from CalDAV/ICS sources, parses them with `ical.js`, and returns busy intervals.
- It handles all-day events, missing VTIMEZONE blocks, TZID extraction, UTC `Z` handling, Apple travel duration, recurrence IDs, and recurrence expansion over the query range.
- Create/update/delete synchronization is performed through `tsdav` (`createCalendarObject`, `updateCalendarObject`, `deleteCalendarObject`, `fetchCalendarObjects`, `fetchCalendars`).
- ICS feed provider is read-only: it parses external calendar feeds and returns busy intervals rather than synchronizing writable events.

### Relevant files

- `packages/lib/CalendarService.ts`
- `packages/app-store/caldavcalendar/lib/CalendarService.ts`
- `packages/app-store/applecalendar/lib/CalendarService.ts`
- `packages/app-store/ics-feedcalendar/lib/CalendarService.ts`
- `packages/app-store/exchangecalendar/lib/CalendarService.ts`
- `packages/app-store/exchange2013calendar/lib/CalendarService.ts`
- `packages/app-store/exchange2016calendar/lib/CalendarService.ts`
- `packages/types/ical.d.ts`
- `packages/types/Calendar.d.ts`

## 9. Scheduling Workflow: Create Meeting Request to Calendar Event Created

Transport/UI are ignored here; this starts at the booking service boundary.

1. **Booking service entry**: `RegularBookingService.createBooking` delegates to its handler with `CreateRegularBookingData` and metadata.
2. **Early policy validation**: the handler validates reschedule restrictions, booker requirements, booking/duration limits, attendee data, event type data, payment/confirmation flags, and organizer/team assignment.
3. **Availability/conflict validation**: it computes or reuses availability/busy data, checks candidate time against current bookings, limits, buffers, seats, reschedule UID, and minimum notice rules.
4. **Event construction**: `CalendarEventBuilder` constructs a provider-neutral `CalendarEvent` containing organizer, attendees, time, location, destination calendars, recurrence/iCal sequence, responses, conferencing preferences, and booking metadata.
5. **Booking persistence**: the booking row is created/updated before external side effects; duplicate conflicts map to booking conflict errors.
6. **Credential refresh**: calendar/video credentials are refreshed after booking persistence.
7. **EventManager creation**: `EventManager` splits app credentials into calendar, video, and CRM credentials and sorts calendar credentials for delegation/latest fallback behavior.
8. **Video/conferencing handling**: `EventManager.create` may create a dedicated video meeting first or let Google/Outlook calendar creation provide Meet/Teams links.
9. **Destination calendar selection**: `createAllCalendarEvents` chooses explicit destination calendars, deduplicates multiple Google destinations, validates CalDAV server URL matches, or falls back to the first calendar credential.
10. **Provider event creation**: `CalendarManager.createEvent` formats the event, renders calendar description, hides notes if requested, obtains a provider with `getCalendar(..., "booking")`, and calls provider `createEvent`.
11. **Reference construction**: `EventManager` maps provider `EventResult`s into `PartialReference`s with UID, URL, password, third-party recurrence id, external calendar id, and credential id.
12. **Post-create updates**: booking references, metadata, iCalUID, conference data, notifications/webhooks/tasks, and app statuses are persisted or dispatched by the booking service.

### Files involved

- `packages/features/bookings/lib/service/RegularBookingService.ts`
- `packages/lib/builders/CalendarEvent/builder.ts`
- `packages/lib/builders/CalendarEvent/director.ts`
- `packages/features/bookings/lib/EventManager.ts`
- `packages/features/calendars/lib/CalendarManager.ts`
- `packages/app-store/_utils/getCalendar.ts`
- `packages/lib/formatCalendarEvent.ts`
- `packages/lib/CalEventParser.ts`
- Provider `CalendarService.ts` implementations.

## 10. Calendar Engine Dependency Graph

```text
RegularBookingService.createBooking
└─ booking handler in RegularBookingService
   ├─ validateRescheduleRestrictions / checkBookingAndDurationLimits
   │  ├─ CheckBookingLimitsService / checkDurationLimits
   │  └─ BusyTimesService.getBusyTimesForLimitChecks
   ├─ UserAvailabilityService.getUserAvailability / getUserAvailabilityIncludingBusyTimesFromLimits
   │  ├─ detectEventTypeScheduleForUser
   │  ├─ getWorkingHours
   │  ├─ buildDateRanges
   │  │  ├─ processWorkingHours
   │  │  ├─ processDateOverride
   │  │  ├─ subtract
   │  │  └─ intersect
   │  ├─ calculateOutOfOfficeRanges / calculateHolidayBlockedDates
   │  ├─ getBusyTimesFromLimits / getBusyTimesFromTeamLimits
   │  └─ BusyTimesService.getBusyTimes
   │     ├─ BookingRepository.findAllExistingBookingsForEventTypeBetween
   │     ├─ getDefinedBufferTimes
   │     ├─ CalendarManager.getBusyCalendarTimes
   │     │  ├─ deduplicateCredentialsBasedOnSelectedCalendars
   │     │  ├─ getCalendarsEvents
   │     │  │  ├─ getCalendar(credential, "slots")
   │     │  │  │  ├─ CalendarServiceMap dynamic provider import
   │     │  │  │  ├─ optional CalendarCacheWrapper
   │     │  │  │  └─ optional CalendarTelemetryWrapper
   │     │  │  └─ provider.getAvailability(GetAvailabilityParams)
   │     │  └─ provider implementations
   │     │     ├─ GoogleCalendarService
   │     │     ├─ Office365CalendarService
   │     │     ├─ BaseCalendarService
   │     │     │  ├─ CalDavCalendarService
   │     │     │  └─ AppleCalendarService
   │     │     ├─ ExchangeCalendarService variants
   │     │     └─ ICSFeedCalendarService
   │     └─ subtract(calendarBusyTimes, openSeatRanges)
   ├─ getAggregatedAvailability
   │  ├─ intersect
   │  ├─ mergeOverlappingDateRanges
   │  └─ filterRedundantDateRanges
   ├─ getSlots
   ├─ checkForConflicts
   ├─ CalendarEventBuilder
   └─ EventManager.create / EventManager.reschedule
      ├─ processLocation
      ├─ createVideoEvent / updateVideoEvent (optional conferencing)
      ├─ createAllCalendarEvents / updateAllCalendarEvents
      │  └─ CalendarManager.createEvent / updateEvent / deleteEvent
      │     ├─ formatCalEvent
      │     ├─ processEvent
      │     ├─ getCalendar(credential, "booking")
      │     └─ provider.createEvent / updateEvent / deleteEvent
      └─ referencesToCreate
```

---

# Subsystem Extraction Roadmap

This document has enough repository inventory to stop broad searching. The extraction should now proceed subsystem by subsystem, recreating the calendar engine in a clean architecture instead of copying Cal.diy implementation details.

For every subsystem below, answer four questions before writing code:

1. What problem does this solve?
2. What are its inputs?
3. What are its outputs?
4. What other modules does it depend on?

Skip completely while extracting: React components, Next.js pages, dashboards, authentication, billing, organizations, teams administration, marketing pages, pricing, admin UI, permissions, and SaaS account management.

## Phase 1 — Domain Models (Foundation)

Do not copy logic yet. Recreate only the domain representation.

| Model | Extraction target | Notes |
| --- | --- | --- |
| `CalendarAccount` | Provider account metadata without secrets. | Represents a connected account, credential id/reference, provider kind, owner, selected calendars, and destination calendar defaults. |
| `Calendar` | Calendar resource metadata. | Represents external id, provider integration, display name, read-only state, primary flag, account relationship, and custom reminders. |
| `CalendarEvent` | Provider-neutral event object. | Contains title, type, start/end, organizer, attendees, location, description, UID/iCalUID, recurrence, conference data, and destination calendars. |
| `Attendee` | Invitee/person attending. | Extract from `Person`; keep name, email, timezone, locale/language, phone, optional booking seat. |
| `Organizer` | Host/owner of event. | Extract from `Person`; same core identity plus username/user id/time format. |
| `BusyTime` | Normalized unavailable interval. | Keep start, end, source, optional title, timezone, user id. Everything that blocks availability should become this. |
| `Availability` | Stored user/event availability rule. | Weekly day/time windows and date overrides. Do not couple to Prisma. |
| `WorkingHours` | Normalized weekly availability. | Days plus start/end minutes since midnight; may be per user. |
| `Schedule` | Collection of availability rules. | Weekly schedule, timezone, date overrides, travel schedule inputs. |
| `RecurrenceRule` | Structured recurring event rule. | Frequency, interval, count, until, DTSTART, TZID, and provider compatibility metadata. |
| `MeetingTypePolicy` | Booking constraints. | Duration, buffers, minimum notice, booking window, seat limits, booking/duration limits, recurring restrictions. |
| `ConferenceDetails` | Meeting link metadata. | URL, provider type, conference data, entry points, access code/password if needed. |

**Problem solved:** establishes a provider-neutral vocabulary for the rest of the engine.

**Inputs:** Cal.diy type shapes and Prisma-backed domain concepts.

**Outputs:** implementation-independent model definitions.

**Dependencies:** none beyond shared date/time primitives.

## Phase 2 — Provider Contract

Extract the `Calendar` interface before reading provider internals. This is the boundary every provider must satisfy.

| Capability | Contract shape | Required? |
| --- | --- | --- |
| Create event | `createEvent(event, account/calendar context) -> ProviderEvent` | Yes |
| Update event | `updateEvent(providerEventId, event, calendar context) -> ProviderEvent | ProviderEvent[]` | Yes |
| Delete event | `deleteEvent(providerEventId, event, calendar context) -> void/result` | Yes |
| Free/busy | `getAvailability({ dateFrom, dateTo, selectedCalendars, mode }) -> BusyTime[]` | Yes |
| List calendars | `listCalendars() -> Calendar[]` | Yes |
| List events | `listEvents(range, calendars) -> CalendarEvent[]` | Optional; inferred from availability/event-overlay paths. |
| Watch/sync | `watchCalendar` / `syncCalendar` / cache warmup | Optional; present around calendar subscription/cache wrappers rather than the base contract. |
| Timezone lookup | `getMainTimeZone()` | Optional provider-specific extension. |
| Delegation validation | `testDelegationCredentialSetup()` | Optional provider-specific extension. |

Ignore provider implementations during this phase. The output should be a provider trait/interface plus request/result types.

## Phase 3 — Availability Engine

Extract the most valuable algorithm as an engine with explicit steps.

```text
FindAvailability
├─ normalize request and policy
├─ resolve participants and schedules
├─ resolve timezone
├─ project working hours + date overrides into DateRange[]
├─ collect out-of-office / holiday blocked ranges
├─ collect BusyTime[] from bookings, providers, limits, buffers, recurrence
├─ subtract BusyTime[] from DateRange[]
├─ aggregate participant/team availability
├─ generate slots with frequency, duration, minimum notice, and timezone rounding
└─ return slots plus diagnostics
```

**Problem solved:** finds candidate meeting slots that satisfy participant availability and meeting policy.

**Inputs:** participants, date range, duration, schedules, meeting policy, selected calendars, reschedule context, timezone.

**Outputs:** candidate slots and diagnostic blocked/busy ranges.

**Dependencies:** domain models, time utilities, busy-time collector, interval utilities, provider contract.

## Phase 4 — Busy Time

Extract every producer of unavailable intervals and normalize them into `BusyTime[]`.

| Source | How it becomes busy |
| --- | --- |
| Existing bookings | Accepted bookings become intervals, expanded by stored and candidate buffers. |
| Calendar providers | Provider free/busy or parsed events become intervals with provider source metadata. |
| Travel time | Apple/ICS travel duration can extend the effective busy window before an event. |
| Buffers | Before/after event buffers become busy time around bookings and calendar events. |
| Booking limits | Reached per-day/week/month/year booking limits become synthetic busy intervals. |
| Duration limits | Reached duration limits become synthetic busy intervals. |
| Team limits | Team-level limits become per-user/team synthetic busy intervals. |
| Recurring events | Expanded occurrences become concrete busy intervals. |
| Seated events | Partially filled slots may remain bookable while their buffers still block adjacent time. |
| Rescheduling | The booking being moved is excluded to avoid self-conflict. |

**Important abstraction:** every source must end as `BusyTime[]`. Source-specific behavior belongs before normalization.

## Phase 5 — Time

Extract time as its own subsystem before porting availability or providers.

| Capability | Responsibility |
| --- | --- |
| UTC conversion | Convert all comparisons to absolute instants. |
| Local conversion | Present and round slots in invitee/organizer local timezone. |
| DST handling | Preserve correct wall-clock availability over DST boundaries. |
| Timezone validation | Accept only valid IANA timezone ids or explicit supported fallbacks. |
| Timezone projection | Project stored weekly rules into concrete dates in a timezone. |
| Working-hour projection | Convert weekly availability rules into date ranges. |
| Travel timezone | Override schedule timezone during travel schedule windows. |
| ICS timezone | Build/read VTIMEZONE and convert DTSTART/DTEND safely. |

Calendar systems fail when time is an incidental helper. In Proxy, make `core/time` a first-class package used by availability, recurrence, providers, and ICS.

## Phase 6 — Recurrence

Extract recurrence after time primitives are reliable.

| Capability | Responsibility |
| --- | --- |
| RRULE parsing | Parse raw RRULE strings into `RecurrenceRule`. |
| Structured representation | Store frequency, interval, count, until, DTSTART, TZID. |
| Expansion | Generate concrete occurrences within a requested date range. |
| Recurrence IDs | Preserve occurrence identity for updates/cancellations. |
| Exceptions | Model skipped/modified occurrences explicitly, even if Cal.diy handles them implicitly/provider-side. |
| Validation | Reject unsupported combinations or return compatibility warnings. |

Do not port analytics/logging. Keep warnings as domain-level validation output.

## Phase 7 — ICS

Extract ICS as import/export/sync primitives.

| Capability | Responsibility |
| --- | --- |
| ICS parser | Parse VEVENTs into `CalendarEvent` / `BusyTime`. |
| ICS exporter | Serialize `CalendarEvent` into iCalendar text. |
| VTIMEZONE | Generate and consume timezone definitions. |
| VEVENT generation | Write DTSTART, DTEND, UID, SUMMARY, DESCRIPTION, LOCATION, RRULE. |
| Attendee serialization | Serialize attendee identities and participation metadata. |
| Organizer serialization | Serialize organizer identity. |
| Recurrence support | Emit/read RRULE and expand recurring busy intervals. |
| CalDAV sync | Create/update/delete/fetch calendar objects for CalDAV-like providers. |

Ignore email templates. Calendar invitations are provider/ICS behavior; product notification emails are not part of the engine.

## Phase 8 — Providers

Only after the contract, time, busy-time, recurrence, and ICS abstractions are clear, read provider implementations. Start with:

1. Google.
2. Outlook / Office 365.
3. CalDAV.

For each provider ask: **How does this provider satisfy the `CalendarProvider` contract?**

| Provider | Read for |
| --- | --- |
| Google | OAuth/delegation, freebusy API, event CRUD, Meet links, selected calendars, main timezone. |
| Outlook / Office 365 | Microsoft Graph event CRUD, schedule availability, Teams behavior. |
| CalDAV | ICS payloads, DAV object sync, VTIMEZONE, generic CalDAV semantics. |
| Apple | CalDAV subclass behavior and Apple travel-time quirks. |
| Exchange | Legacy Exchange implementation of the same contract. |
| ICS feed | Read-only provider that only satisfies busy-time import/listing behavior. |

Everything else can be added later as optional provider packs.

## Phase 9 — Meeting Policies

Extract policy as data and pure validators.

| Policy | Engine behavior |
| --- | --- |
| Minimum notice | Removes slots before `now + minimumBookingNotice`. |
| Booking window | Limits how far into the future slots can be offered. |
| Buffers | Expands busy intervals before/after events. |
| Seat limits | Allows a slot while capacity remains; blocks once full. |
| Booking limits | Blocks periods when count limits are reached. |
| Duration limits | Blocks periods when booked-duration limits are reached. |
| Recurring restrictions | Validates recurrence compatibility and future occurrences. |
| Meeting duration | Requires every generated slot to fit entirely inside availability. |

Ignore CRUD and storage. The model should be usable with any persistence backend.

## Phase 10 — Utilities

Extract small algorithms last, when their consumers are clear.

| Utility | Used by |
| --- | --- |
| Interval overlap | Conflict detection. |
| Interval merge | Aggregating availability and busy windows. |
| Interval subtraction | Removing busy windows from working ranges. |
| Interval intersection | Multi-participant/team availability. |
| Sorting/deduplication | Provider results, slots, ranges. |
| Date formatting/parsing | Provider payloads, ICS, diagnostics. |
| Validation | Timezone, recurrence, provider payloads, policy rules. |

These should be pure functions with targeted tests.

## Extraction Principle

Do not think in terms of Cal.diy classes. Think in terms of capabilities:

| Capability | Inputs | Outputs | Depends on |
| --- | --- | --- | --- |
| Availability Engine | Participants, schedules, policy, calendars, range | Candidate slots, diagnostics | Time, busy time, interval utils, providers |
| Busy-Time Collector | Accounts, bookings, limits, range | `BusyTime[]` | Providers, recurrence, ICS, policy |
| Provider Contract | Provider credentials/context, event/range requests | Events, calendars, busy intervals | Domain models, time |
| Recurrence Engine | Rule, DTSTART, timezone, query range | Occurrences | Time |
| ICS Subsystem | Calendar events or ICS text | ICS text, events, busy intervals | Time, recurrence, models |
| Scheduling Workflow | Meeting request, participants, policy | Booking + provider references | Availability, providers, conference packs |

The end state should be a clean calendar engine with domain models, provider interfaces, availability/busy-time algorithms, recurrence/time/ICS subsystems, meeting policies, and pure utilities—without inheriting the surrounding SaaS product.
