# pingrules — Alerts by rules, with a fallback if they break

**Paid tool: $99 once (plus sales tax where it applies), then a private GitHub repository you're invited to, with every update. Buy: https://dominares.org/tools/pingrules**

Commercial license v1.3 — see https://dominares.org/license · Support: support@dominares.org

pingrules sends your app's events to Telegram and email by a rules file you edit by hand. It reports each delivery leg by leg, and a fallback destination still alerts someone when the rules file itself breaks.

## Get it

Buy pingrules at https://dominares.org/tools/pingrules: one payment of $99, plus sales tax where it applies, and every future update. After checkout you are invited to a private GitHub repository with the source and every version published to it. The repository includes the Python package and the pingrules command, an example rules file, and the full test suite.

Requirements: Python 3.10 or newer, standard library only. A Telegram bot token and a Resend API key or an SMTP server to send for real; the demo needs none. tzdata is optional, for systems without a time zone database.

This page documents version 1.0.1. Install commands and file paths in the documentation below run inside the private repository you are invited to; the source is not in this repository.

## Documentation

### What it does

- **Rules in one file.** A recipient is a Telegram chat or an email address. An event is something your code
  fires by name (`payment_failed`), with the fields its payload carries. A rule says *when this event fires, and
  these conditions hold, tell these recipients*. The file is read on every fire, so an edit takes effect at once
  and a file that has broken is noticed at once.
- **Conditions on the payload.** `{field, operator, value}` rows with ten operators (`equals`, `contains`,
  `starts_with`, `greater_than`, `is_empty`, ...), combined with `all` or `any`. Fields can be dotted paths into
  a nested payload (`pingrules check` accepts one when its first segment is a declared field). There is no code and
  no regular expression in a condition. Numbers are compared as numbers: the first number in the value is used
  (`$1,250.50` is 1250.5, `-$40` is -40), and a value with no number in it never satisfies `greater_than` or
  `less_than`.
- **Quiet hours.** A window such as 21:00 to 07:00 in a time zone you name. Telegram is held during it, but only
  when another leg that is not held is set up to send (an email sender is actually configured, or, if you hold
  email instead, a Telegram token), so a Telegram-only event, or a setup with no email sender, is never silenced.
  If the email then fails at send time, the failure sends the fallback like any failed fire. Email is not held
  unless you list it in `mute_channels`; listing both channels holds nothing, and `pingrules check` says so. An event in an `urgent` category, a fire passed
  `urgent=True`, and a test fire go through. A held ping is recorded as held, not as a failure.
- **Test fire.** `pingrules test-fire --send` (or `Router.test_fire`) sends a **real** message marked `[TEST FIRE]`,
  built from the event's sample values, ignoring conditions and quiet hours. Narrow it to one rule or one recipient to answer "does this reach
  that person?".
- **A delivery log that holds nothing personal.** One line per fire: the event key, the rule ids, per leg whether
  it worked, how long it took, and a short error. Never an address, a chat id, a message or a payload. Error text
  is scrubbed of email addresses, phone-like numbers and bot tokens first. The log is capped (500 entries by
  default) so it cannot grow without bound, and reading an unreadable log says so instead of showing it as empty.
- **A fallback that works without the rules.** Set a Telegram chat and/or an email address in the environment. The
  router uses it when the rules cannot be read, and when every delivery attempt for a fire failed. The fallback
  message names the reason and carries the payload's plain values. With nothing configured, a fallback send is a
  failure with a reason. `pingrules status` says `delivering` or `NOT delivering (reason)`.
- **Messages that Telegram will not reject.** Text over 4096 characters or HTML that does not parse makes Telegram
  answer 400 and drop the whole message. Messages are shortened without cutting a tag in half and with every open
  tag Telegram accepts closed, payload values and the labels and descriptions of events and fields are escaped, and `pingrules check` tells you in words why a template you wrote would be
  refused (`<br>`, an unclosed `<b>`, a stray `<`). Telegram's own error text is kept instead of a bare "HTTP 400".
- **"What changed" blocks.** Pass `changes` and/or `details` in the payload and the message ends with a
  `What changed:` or `Details:` block (and, in email, a small card).
- **Retry, when you ask for it.** `retries=N` repeats a Telegram call that got no answer, was rate limited (waiting
  as long as Telegram says, at most 30 seconds) or hit a 5xx. A 4xx is never retried. The router and the fallback
  retry once by default; a bare `telegram.call` does not retry.

### Quickstart

Nothing here sends anything real: without `--send` the command prints what each channel would receive, says
`(preview, nothing sent)` beside the status, and flags any `--log` row as a preview. The preview does not pretend
a fallback exists: it uses only the destinations your environment names (`PINGRULES_FALLBACK_CHAT`,
`PINGRULES_FALLBACK_EMAIL`). Run these in an empty folder, one after another.

<!-- quickstart:start -->
```bash
pingrules init                                   # writes rules.json, a small working example
pingrules check rules.json                       # validates it, in words
pingrules fire rules.json payment_failed --set customer=acme --set amount=250 --log delivery.jsonl
pingrules fire rules.json payment_failed --set customer=acme --set amount=250 --at 2026-01-15T23:30:00Z --log delivery.jsonl
pingrules fire rules.json payment_failed --set customer=acme --set amount=20 --log delivery.jsonl
pingrules test-fire rules.json payment_failed --recipient ops-mail
echo '{oops' > broken.json
PINGRULES_FALLBACK_CHAT=999000111 pingrules fire broken.json server_down --set host=web-1 --log delivery.jsonl
pingrules log delivery.jsonl
```
<!-- quickstart:end -->

What you will see: the first fire reaches both recipients. The second is the same payment at 23:30 UTC, inside
quiet hours, so Telegram is `held` and the email still goes. The third has an amount of 20, below the rule's
condition, so nobody is bothered (`skipped: conditions-not-met`). The test fire reaches only `ops-mail`, marked
`[TEST FIRE]`. The broken file, with a fallback chat named in the environment for that one line, produces `status: fallback`,
with the reason, and exits 1 so a script notices (leave the variable off and the preview prints `rules-unreadable`
and `FAILED (no fallback destination is configured ...)`, which is what a real send would do). `log` lists all of it
newest first, each preview row marked `[preview]`.

### The rules file

`pingrules init` writes a complete one. In short:

```json
{
  "channels":    {"telegram": {"enabled": true}, "email": {"enabled": true}},
  "quiet_hours": {"enabled": true, "start": "21:00", "end": "07:00", "timezone": "America/New_York",
                  "urgent_categories": ["urgent"], "mute_channels": ["telegram"]},
  "recipients":  [{"id": "oncall", "label": "On call", "channel": "telegram", "address": "123456789"},
                  {"id": "ops", "label": "Ops inbox", "channel": "email", "address": "ops@example.com"}],
  "events":      [{"key": "payment_failed", "label": "A payment failed", "category": "billing",
                   "fields": [{"name": "amount", "type": "number", "sample": "250"}]}],
  "rules":       [{"id": "big-failures", "event": "payment_failed", "recipients": ["oncall", "ops"],
                   "match": "all", "when": [{"field": "amount", "operator": "greater_than", "value": "100"}],
                   "template": {"telegram": {"title": "Payment failed", "body": "{{customer}}: {{amount}}"}}}]
}
```

- A **Telegram address** is a chat id (digits, `-100...` for groups) or `@channelname`. Create a bot with
  @BotFather, send it a message, and read your chat id from `getUpdates`.
- **Templates** use `{{name}}` markers (dotted names reach into nested payloads; unknown names render empty).
  A template can set `title`/`body` for Telegram and `subject`/`body` for email, on a rule or on an event; a rule's
  template beats the event's, which beats the default built from the event's label, description and fields.
  `{{change_list}}`, `{{detail_list}}` and `{{context_block}}` place the "what changed" blocks yourself; otherwise
  they are appended.
- A row that cannot be understood is **dropped and reported**, never silently: `pingrules check` lists it and
  exits 1. That covers a recipient without an id, an event without a key, a rule without an id or event, a
  condition row without a field and an operator (a typo such as `op` for `operator`), a `when` that is not a list,
  a template under a channel other than `telegram` or `email`, and a template part other than `title`, `subject`
  and `body`. A dropped condition makes its rule match more events than written, so `check` always names it.
- `pingrules check` also catches: a rule naming an undeclared event or a missing recipient, a rule that would alert
  nobody, a condition on a field the event does not declare, an unknown operator, a bad address, duplicate ids,
  Telegram text that would be refused, and a quiet-hours window without a time zone.

### From Python

<!-- python-example:start -->
```python
from pingrules import Router, Fallback, DeliveryLog
from pingrules.mail import ResendSender

router = Router(
    "rules.json",                                     # a path, a Rules object, or a function returning one
    telegram_token="123456:bot-token",
    email_sender=ResendSender("re_api_key", "alerts@example.com"),
    log=DeliveryLog("delivery.jsonl"),
    fallback=Fallback(telegram_token="123456:bot-token", telegram_chat="123456789"),
)
result = router.fire("payment_failed", {"customer": "acme", "amount": "250"})
print(result.status)     # delivered | partial | fallback | failed | skipped | unknown-event | rules-unreadable
```
<!-- python-example:end -->

`fire` never raises into your code. `result.delivered` is true only when somebody was actually reached (a rule's
recipient or the fallback), `result.legs` lists every attempt with its error, and `result.skipped` says why
anything was passed over. `fire` always sends: it has no dry-run switch. To see messages without sending, build the
router with stand-in transports and `preview=True` (what the command line does); `result.preview` and the log's
`preview` field then say nothing real went out, and `result.test` / `test` mark a `test_fire`. `fire_async` does the
same as `fire` on a daemon thread so a request handler is not held.
`router.health(probe=True)` answers "can an alert reach anyone right now?": a channel counts only if it has both a
working credential and a recipient on an active rule, and `health()["delivery"]["warnings"]` names every channel
that has recipients but cannot deliver, even while another channel works. `SmtpSender` sends through any SMTP server instead of
Resend. Any function `(to, subject, text, html) -> (ok, error)` works as an email sender, and any function with the
signature of `pingrules.telegram.Transport` can replace the network in your own tests.

### Command line

| Command | What it does |
| --- | --- |
| `pingrules demo` | A built-in tour. Sends nothing. |
| `pingrules init [PATH] [--force]` | Writes a sample rules file. |
| `pingrules check RULES` | Validates the file; exit 1 if anything is wrong. |
| `pingrules fire RULES EVENT [--set K=V] [--payload JSON] [--urgent] [--at ISO] [--log FILE] [--send]` | Routes one event. |
| `pingrules test-fire RULES EVENT [--rule ID] [--recipient ID] [--set K=V] [--send]` | A marked test of a rule. |
| `pingrules log FILE [--limit N] [--json]` | Shows the delivery log, newest first. |
| `pingrules status RULES [--probe]` | Can an alert be delivered with this file and this environment? Names every channel that cannot, even if another can; exit 1 then. |

Exit codes: 0 fine; 1 a problem (a failed, partial or fallback delivery, an invalid file, a channel that cannot
deliver); 2 an unknown event or a usage error.

With `--send`, the real transports are read from the environment:

| Variable | Meaning |
| --- | --- |
| `PINGRULES_TELEGRAM_TOKEN` | Bot token for rule recipients and the fallback. |
| `PINGRULES_RESEND_KEY`, `PINGRULES_EMAIL_FROM` | Email through Resend. |
| `PINGRULES_SMTP_HOST`, `PINGRULES_EMAIL_FROM`, optional `PINGRULES_SMTP_PORT`, `_USER`, `_PASSWORD`, `PINGRULES_SMTP_SSL=1` | Email through SMTP (STARTTLS on 587 by default). |
| `PINGRULES_FALLBACK_CHAT`, `PINGRULES_FALLBACK_EMAIL` | The fallback destination. |

A time zone database is needed for quiet hours in a named zone. Python's `zoneinfo` uses the system's; on Windows or
a slim container install the extra: `pip install '.[tz]'`. An unknown zone is read as UTC, and `pingrules check` says so.

### Limits

- **Quiet hours hold, they do not queue.** A held Telegram ping is not sent later. The email leg (or the next fire)
  is what reaches the person. If you need "deliver at 07:00", that is a queue, which this is not.
- **Only Telegram and email.** No SMS, Slack, webhooks or push. Two channels, done carefully.
- **One process, no scheduler, no digests, no deduplication.** pingrules routes a fire when your code calls it. It
  does not watch anything, batch alerts, throttle repeats or page on a schedule.
- **The fallback sends the payload's plain values** (strings, numbers and booleans at the top level, 20 at most) to
  the fallback destination. Do not put secrets in an event payload.
- **`status` without `--probe` checks that a token is set, not that it works.** Add `--probe` to ask Telegram.
  Neither proves the person reads their phone.
- **The delivery log is locked across processes on POSIX** (`fcntl`). On Windows only threads in one process are
  serialised.
- **The rules file is edited by hand or by your code** (`load_rules`, `save_rules`). There is no editing interface.
- **A bare `telegram.call` or `send_message` does not retry**; the router and the fallback retry once by default
  (`Router(retries=...)`, `Fallback(retries=...)`).

> **pingrules is a tool. It cannot make anyone read an alert.** It makes sure that what was sent, what
> failed and what was held is never in doubt, and that a broken setup is itself reported.

---

Docs mirror of the private repository Dominares-Tools/pingrules; updated with each release.
