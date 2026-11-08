---
name: foodics-zendesk
description: Answer questions about Foodics (and Tanar) Zendesk Support data with the zendesk MCP server, using the BI team's definitions. Use whenever someone asks about Foodics tickets, case types, product types, CSAT, CES, FCR, first reply or resolution times, chat / email / internal performance, abandoned chats, training or menu configuration (onboarding), Answer Bot, Zendesk views or queues, or wants to find tickets by F5 number or phone, read ticket comments, or add a note, create or update a ticket, or send a WhatsApp message or template notification, or look up / change Sunshine Conversations (Smooch) webhooks, integrations and switchboards. Triggers include "how many tickets", "CSAT for", "case type of", "tickets for F5", "performance yesterday", "training completion", "what's in the view", "add a note to ticket", "send a WhatsApp", "send the template", "smooch webhook", "escalated tickets", "side conversation", pastes a foodics.zendesk.com link, or asks about Tanar / tanarhelp.
---

# Foodics Zendesk

The `zendesk` MCP server reads foodics.zendesk.com through shared service accounts and calculates support KPIs
with the BI team's Snowflake definitions. It does not query Snowflake: numbers come straight from Zendesk, so
they can be fresher than the dashboards. It also sends WhatsApp messages through Sunshine Conversations
(Smooch), and admins can change Zendesk and Sunshine settings with a rollback for every change.

Roles (`whoami`): **viewer** reads; **writer** also adds notes, creates / updates tickets and sends WhatsApp;
**admin** also changes configuration, users, tickets in bulk and Sunshine settings.

## Instances (which Zendesk)

Every tool takes `instance`. Leave it out for **foodics** (foodics.zendesk.com), the default. Pass
`instance: "tanar"` for **tanarhelp.zendesk.com** when the person asks about Tanar. `whoami` lists the instances
the person has; others are refused, so say who to ask (an MCP admin ticks them on the access page).

- Never mix instances in one answer without saying which numbers come from which Zendesk.
- Ids (tickets, fields, groups, views) belong to one instance: a Tanar ticket number means nothing on Foodics.
- The BI definitions below are **Foodics'** unless a line says Tanar. Tanar has its own (see "Tanar BI");
  `onboarding_performance` and `find_tickets_by_f5` are Foodics-only.
- WhatsApp on Tanar needs `integration_id` / `parent` (from `sunshine_read(kind="integrations", instance="tanar")`);
  there is no OpenCX handover there.
- A rollback always runs on the instance the change was made in.

### Tanar BI (instance "tanar": the Pay Zendesk)

`get_support_attributes`, `support_summary`, `view_tickets(with_support_attributes)` and `channel_performance`
work on Tanar with the BI team's Tanar definitions (Snowflake `BI_TEAM.TANAR_OKRS`: table
FCT_TICKET_CUSTOM_FIELD, FCT_TICKET_CONVERSATION, VW_FCT_SUPPORT_INTERACTION_{EMAIL,INTERNAL,CHAT}, and the same
VW_AGG_SUPPORT_*_PERFORMANCE as Foodics). Checked against Snowflake: 396 sampled tickets match on all 22 columns;
20 Oct–2 Nov email, internal and chat daily numbers match (chat 8 of 9 days, see below).

- Product type → sub-category → case type from the forms' "Product R/C" / "Product Type" conditions. A
  sub-category field with no further conditions gives sub-category **"Inquiry"** and its value as the case type.
- **CSAT** is the "3cx CSAT" text field (1–5; a few `X`). Zendesk satisfaction ratings are empty on Tanar.
  `support_summary` averages only 1–5; the interaction views count any answer as rated.
- **CES** is "CES Rating" free text (e.g. `[5] : محسن بشكل كبير`): its first digit 1–5. Raw text: `ticket_custom_ves`.
- **FCR and outbound are not defined for Tanar** (always empty): don't report FCR for Tanar.
- `channel_name` is `via.source.rel` first, then the channel: `follow_up`, `side_conversation`, `api` (calls),
  `email`, `native_messaging`, `web`. `custom_channel` is `internal` when "Internal ticket" is ticked, else
  the Channel field (Inbound Call, Emails, Chat…). Country ("Country - Pay") is an inactive field, so it's empty, as in BI.
- Extra columns: form and group name, merchant reference (upper-cased), follow-up ids, Pay validation /
  financial / non-financial validation, final decision, caller number, xCALLY id, SAMA CSAT handling /
  resolution, requester / assignee emails. Any other field: `group_by: ["field:<title>"]`.
- `channel_performance` on Tanar: **email** = channel `email` and not internal (follow-ups excluded); first reply
  is the first public comment by anyone but an end-user. **Internal** = "Internal ticket" ticked, queue
  `COE Support` by default; first reply is the first comment by neither requester nor submitter. **Chat** = one
  session per chat start on native messaging, from the transcript: first reply / queue time = the end-user's
  first message to the agent's answer, handling = agent first-to-last message per reply window (summed);
  sessions with no agent time are dropped; abandoned = no later replies on the ticket's first session.
- Tanar chat quirks copied from the view (say so when they matter): a chat's transcript holds the customer's
  earlier conversations, so a session can be dated weeks before the chat (by the first agent message in the
  transcript), and recent days change as later chats arrive; a completed session counts on its UTC date.
  Deleted tickets are in BI's email totals but not here (the API can't see them).

## Pick the tool

| Question | Tool |
|---|---|
| How many tickets match X? | `count_tickets` (one call). Use this before pulling rows. |
| Tickets with a product / complaint / country value over time | `count_tickets_by_option`. One value has many tags across fields (RMS: `rms0000`, `rms00`, `foodics_rms`, …; KSA: `ksa1`, `ksa`, `ksa2`, …), so never filter on one tag: pass the wording plus `field_contains` (e.g. `"product"`), and `and_options` / `not_options` (e.g. `[{"wording": "KSA", "field_contains": "country"}]`). A tag counts only on forms that contain its field; for support tickets pass `forms: ["Support Request/Complaint", "Support Form Inquiries"]`. Close to the BI product type (14 Oct: 309 vs 306), not identical; for dashboard-exact numbers use `support_summary`. |
| Which tickets match X? | `search_tickets` (Zendesk search syntax; `updated_after` splits a long range into parallel windows) |
| Tickets for an F5 number | `find_tickets_by_f5` (6 digits) |
| Which F5 / merchant / group / form has the most X (complaints, escalations…) | `support_summary` with `group_by: ["f5"]` (or `"merchant"`, `"group"`, `"form"`; these show names, `group_id` / `ticket_form_id` show ids) and `filters`, e.g. most complaints today: `group_by: ["f5"], filters: {"sub_category": "Complaint"}`. Complaint is a **sub_category** (Inquiry / Request / Complaint / Suggest), not a case type. F5 `000000` is the "unknown" placeholder and counts as none; report the no-F5 bucket separately. Escalations per group: `group_by: ["group", "is_escalated"]`. Over more than a day or two the 5000-ticket read cap is hit (`truncated: true`): narrow with `query: 'form:"Support Request/Complaint Form"'`. Tanar has no F5: use `merchant`. |
| Tickets linked to a merchant (or any lookup record) | `lookup_relationships(field="Merchant", record_id=...)`; find the record with `custom_object_records("merchant", query=...)` |
| Account settings, installed apps, brands, schedules, support addresses, dynamic content, user / organization fields | `admin_config(kind=...)` (admin): `account_settings` (by section; update `{"tickets.comments_public_by_default": false}`), `apps` (list / get settings, enable / disable, change settings; install and remove in Admin Center). Secrets are hidden and never changed. Every change is previewed and can be rolled back |
| Bulk config changes (many triggers, groups, fields… at once) | `admin_config`: `create` with `data` as a list (up to 50), `update` with `ids` + one change, or `data: [{"id": …, …}, …]` for a different change per object; `manage_users` create also takes a list. One preview, one confirm, one `op_id`: `admin_rollback` undoes the whole batch. If a create fails midway it stops, and the op holds exactly what was made |
| Organizations, who is in which group / organization | search: `search_people(kind="organization")`; admin: `admin_config("organizations", search=name)`, `admin_config("group_memberships", search="group:<id>")` or `"user:<id>"`, `organization_memberships` (`"organization:<id>"` / `"user:<id>"`); add someone to a group: `action="create", data={"user_id": …, "group_id": …}` |
| Tickets for a phone number | `find_tickets_by_phone` (any of 05… / 966… / +966…; also matches the requester's profile) |
| One ticket's details | `get_ticket` (named fields, side conversations: who it was escalated to); its comments: `list_ticket_comments` |
| The person pasted a Zendesk link (ticket, view, user, org, trigger, automation, macro, field, form, group, SLA, status, article) | `open_link(url)`: extracts the id and fetches it; a view comes with its live count |
| Escalated tickets | count / search: `tags:escalated` (e.g. `count_tickets("tags:escalated created>=2026-10-01 created<2026-11-01")`); KPIs: `support_summary` with `group_by: ["is_escalated"]` or the `escalated_tickets` column |
| Product type, sub-category, case type, CSAT, CES, FCR flag, reply/resolution times of tickets | `get_support_attributes` (≤500 ids) |
| KPIs over a date range, grouped by anything | `support_summary`. Columns: tickets, FCR-included, CSAT / CES, medians of first reply, first and full resolution, requester / agent wait and on-hold (minutes), reopened, escalated, group-transferred and L2-backlog tickets. Group or filter by product / sub-category / case type, country, channel, CSAT / CES score and reason, `is_escalated`, `is_reopened`, `is_group_transferred`, `is_outbound_ticket`, `is_fcr_included`, `is_suspended_ticket`, `is_followup_ticket`, `is_internal_whatsapp`, `is_l2_backlog`, status, group, assignee, form, time, or `field:<any field>` |
| Daily chat / email / internal performance | `channel_performance`; per agent / queue / sub-channel: `group_by: "assignee"` (with names), `"queue"`, `"sub_channel"`; for a whole period instead of per day: `per: "range"` (e.g. each agent's week) |
| Training and menu configuration performance | `onboarding_performance` |
| Answer Bot article suggestions | `answer_bot_summary` |
| What is in a Zendesk view / queue right now | `list_views`, `count_views`, `view_tickets` |
| Who changed a field, and when | one ticket: `get_ticket_history`; all tickets in a time range: `account_audits` |
| How a form's conditions lead to a field | `trace_field_conditions` (gives ready `search` strings); one ticket: `trace_case_type`; all forms: `list_forms` |
| Groups, statuses, brands, custom objects, deleted tickets | `list_reference`, `custom_object_records` |
| Users / organizations | `search_people` |
| Help center articles | `search_help_center` |
| WhatsApp conversation, its messages, a Sunshine user, approved templates, webhooks / integrations / switchboards | `sunshine_read` |
| Send a WhatsApp template notification or a message in a conversation | `sunshine_send` (writers; see WhatsApp below) |
| Change Sunshine webhooks, integrations, switchboards | `sunshine_config` (admins) |
| Can I write? Am I an admin? | `whoami` |

For a ticket's case type, use `get_support_attributes`: that is the BI definition. `trace_case_type` shows the
full chain of form selections, which is useful to explain *why*, not as the KPI value.

## Definitions (BI team's, as implemented)

- **Product type → sub-category → case type:** on the ticket's form, the product field (title contains
  "product") shows a sub-category field (title contains "sub"). That shows the case-type field
  (complaint / request / inquiry dropdown). Values are option names, e.g. `RMS` → `Complaint` →
  `RMS Cashier::Release Issue`.
- **CSAT score (1–5):** the `3cx CSAT` field first (yes / نعم / جيد / good → 5, no / لا / سيء / bad → 1, or a
  digit 1–5). Then the tags `whatsappgoodrating` (5) / `whatsappbadrating` (1). Then Zendesk's native rating
  (good 5 / bad 1). **Good CSAT = 4 or 5.** Report good ÷ rated, never good ÷ all tickets.
- **CES:** the `CES Rating` field: the highest digit 5..1 that appears in it.
- **Escalated:** the ticket has a side conversation, counted by the `escalated` tag everywhere (`get_ticket`
  also lists the side conversations themselves, including older untagged ones). The trigger "Side Ticket Created -> Tagger (Escalated)"
  adds the tag `escalated` when one is created, so it is searchable (`tags:escalated`). The trigger exists since
  2024-01-29: older tickets aren't tagged, so say so for earlier periods. `get_ticket` shows each side
  conversation's subject, state and participants (people and groups, e.g. `Integration L3`).
- **Other flags (BI view):** suspended = a tag contains `suspended_ticket`; follow-up = a tag like
  `follow_up`; internal WhatsApp = the internal-WhatsApp field is set; L2 backlog = the L2 backlog field is
  `yes_l2_coe`; reopened = reopens > 0; group-transferred = more than one group on the ticket.
- **xCALLY id:** `id-` + the field's digits (dots removed); NULL when the value isn't numeric.
- **FCR-included:** the requester is not @foodics, the custom channel is one of native_messaging, whatsapp, email,
  telegram, social_media or inbound_call, the ticket is not outbound, and the case type is not an automated or
  cc'd one. This flag marks tickets *eligible* for FCR; it is not the FCR rate.
- **Custom channel:** inbound_call when an xCALLY id is set; twitter / facebook / instagram → social_media;
  native_messaging / whatsapp / email / telegram as is; otherwise the "Channel" field, else Zendesk's channel.
- **Ticket metrics** are in **seconds** (`*_business_second` = business hours, `*_calendar_second` = wall clock).
  `support_summary` reports medians in **minutes**.
- **Chat performance** (per chat session, not per ticket):
  - **Abandoned:** the ticket has no messaging reply.
  - **Answered within threshold:** queue time ≤ 30 s.
  - **Averages:** `AVG(DISTINCT …)`, as in the view.
  - **First reply:** the assignee's first message from 5 s before the session started. No value means the session was not replied to.
  - **Unique answered:** answered sessions that are the requester's only session of that status, sub-channel and day.
    It is worked out over all sessions before any `group_by` split, so splitting doesn't change it.
  - **Messages:** average customer and agent (last assignee) messages per answered session, counted in the
    session's own window (the view counts the whole stored transcript, earlier sessions included).
- **Email / internal performance:**
  - **Queue time:** created → first assigned.
  - **First reply:** first assigned → first agent comment. Email counts public comments by agents (an author whose role is unknown doesn't count, as in the SQL); internal counts any comment by someone other than the requester or submitter.
  - **Answered within threshold:** first reply ≤ 30 min.
  - **Scope:** internal = the Channel field `internal_communication` (also when the ticket arrived by email /
    WhatsApp) in the `COE Support` group. Email sub-channel = the custom channel (e.g. `inbound_call`).
    A ticket that came in by email with the Channel field set to internal counts in both email and internal,
    as in the BI views.
- **Days** are local **Asia/Riyadh** dates.
- **Training / menu:** tickets on the "Training or Menu Configuration" form, split by `Onboarding - Task Type`.
  Training outcomes come from `Onboarding - Training session completion status`. The completion,
  no-show and reschedule rates are **provisional**, not yet matched to the Metabase dashboards. Say so when you
  report them.
- **Answer Bot:** suggestions = `AutomaticAnswerSend`, viewed = `AutomaticAnswerViewed`, solved by Answer Bot =
  `AutomaticAnswerSolve`. "Marked helpful" is the `ar_marked_helpful` tag.

## Working habits

- **Count before you list.** `count_tickets` costs one call; listing 1000 tickets costs 10+.
- **One summary call, not many.** For "by hour / by day" use `group_by: ["created_hour"]` or `["created_day"]`
  (Asia/Riyadh local) in a single `support_summary` call. Never loop `count_tickets` or `support_summary` over
  hourly windows. Asking again with another `group_by` or `filters` on the same dates is cheap: unchanged
  tickets are reused.
- **Range limit:** `support_summary` and `onboarding_performance` read at most 5000 tickets (a busy day is
  ~3000); if `truncated` is true, narrow the dates or add a `query` and say the result was partial.
- **`channel_performance` and `answer_bot_summary` may return `complete: false` with `pending_days`.** They read
  a day of audits once (≈22 calls) into a shared cache. Call again with the same arguments until `complete` is
  true. Tell the user it's loading; don't present partial numbers.
- **Group by any ticket field** in `support_summary` / `onboarding_performance` with `field:<field title>`, e.g.
  `field:handled by Training`, `field:Onboarding - Area of Training`.
- **Search syntax:** custom fields are `custom_field_<id>:<tag>` (tags, not display names). Get them from
  `trace_field_conditions` or `get_ticket`. Forms: `form:"Support Request/Complaint Form"`.
- **Budget:** each person has 60 Zendesk calls a minute. If a tool says the budget is used up, wait the seconds
  it gives or narrow the question. Don't loop. Identical questions within a minute (any user) are free.
- **Freshness:** results are live. They can differ from Snowflake dashboards, which run behind, and where
  the views were wrong before 2026-11-06 (email queue time +3 h, negative chat first replies, truncated xCALLY ids).
- **Known limits:**
  - Account-wide audits don't include archived tickets (closed more than ~120 days ago), so performance for older periods is incomplete.
  - OpenCX AI-agent sessions are not connected.

## Admin (admins only)

`admin_config` (triggers, automations, macros, views, ticket fields and forms, custom statuses, SLA policies,
groups), `manage_users`, `bulk_tickets` and `admin_rollback` exist only for admins (`whoami` → role `admin`).
People's access (viewer / writer / admin) is set on the access page, `<MCP server URL>/admin`, not through tools.

- **Find before changing:** `admin_config(kind, "list", search=...)`, `get`; bulk tickets by `ids` or a `query`
  (count it first with `count_tickets`). Up to 50 config objects or users, 1000 tickets per call.
- **Every change is two calls.** The first returns a preview and a `confirm` token. Show the person the preview
  (what changes, how many, the sample) and wait for an explicit yes; only then call again with the same
  arguments plus `confirm`. Never pass a token the person hasn't approved. A changed argument or object needs
  a new preview.
- **Every change can be rolled back.** Each one returns an `op_id`. `admin_rollback()` lists recent operations
  (who, when, what); `admin_rollback(op_id)` previews the undo, then `confirm` applies it, with the same
  approval rule. Values someone changed again since are left as is and listed under `skipped`: tell the person.
  Deleted config objects come back with new ids; deleted tickets within Zendesk's 30 days. Access changes made
  on the access page are operations too.
- **Not offered, because they can't be rolled back:** deleting or merging users (suspend with
  `{"suspended": true}`), deleting ticket fields or custom statuses (deactivate with `{"active": false}`),
  closing tickets, writing CSAT / CES.

## WhatsApp (Sunshine Conversations)

- **Numbers (integrations):**

  | Number | `integration_id` |
  |---|---|
  | Whatsapp_KSA (main, the default) | `65f9d73f5c36afb5d1765823` |
  | Whatsapp Egypt | `644feaef4367668867111dff` |
  | Whatsapp_Menu | `67b59f920f8764b921d4a79f` |

  Others (Instagram, Messenger, web, custom): `sunshine_read(kind="integrations")`.
- **Find the template first:** `sunshine_read(kind="templates", template="<part of the name>")` (add
  `parent=<integration_id>` for another number) shows each template's language, status and body with its
  `{{1}}`, `{{2}}` variables. `body_params` must fill all of them, in order. Languages are like `ar` and
  `en_US`; `language="en"` finds `en_US`. Only approved templates send.
- **Conversation messages** go into an existing conversation: find it with `sunshine_read(kind="user_conversations",
  id=<Sunshine user id or external id>)`, read it with `kind="messages"`. A plain message reaches WhatsApp only
  within 24 hours of the customer's last message; outside that window send a template notification.
- **Every send is two calls.** The first returns the exact text, the recipient and a `confirm` token. Show it to
  the person and wait for an explicit yes; only then call again with the same arguments plus `confirm`.
- **A sent message can't be taken back.** Sends are recorded in the operation log (admins see them in
  `admin_rollback()`), but there is no rollback. One recipient per call; never loop over a list of numbers
  without the person approving each send. Bulk / campaign sending isn't offered.
- On the main number (Whatsapp_KSA) the conversation is passed to OpenCX after a notification, so the
  customer's reply reaches OpenCX, not Answer Bot. Say so in the preview; `hand_to_opencx=False` only if asked.
- Phone numbers in international form (`9665…` or `009665…`); `05…` is refused, not guessed.
- **One yes, one send.** A confirm token works once. If a send errors with "MAY have been sent", read the
  conversation (`sunshine_read(kind="messages")`) before asking to send again. Named template variables
  (`{{name}}`) aren't supported; numbered ones are.
- `sunshine_config` (admins) works like `admin_config`: preview, confirm, `op_id`, rollback. Deleting an
  integration or the switchboard isn't offered; a deleted webhook comes back with a new id and a new secret.
  Secrets are always hidden.

## Writing (writers only)

`add_internal_note`, `create_ticket`, `update_ticket` and `sunshine_send` exist only for writers and admins;
`whoami` tells you the role. For a viewer, say they have view access and who to ask. Don't look for workarounds.

- **Confirm before every write:** show the ticket, the exact change or note text and whether it is public, and
  wait for a yes.
- **Internal by default.** Make a comment public only when the user explicitly asks to reply to the customer.
- **Not possible through the MCP:**
  - closing tickets (set `solved`)
  - writing CSAT / CES fields
  - overwriting a change someone made meanwhile (the update fails; re-read the ticket and ask again)
- **Every write is signed** with the writer's email in an internal note. Don't add your own signature.
- **Field values:** use option names as shown in Zendesk ("Training"). The tool maps them to tags and lists
  the valid options if a value is wrong.
