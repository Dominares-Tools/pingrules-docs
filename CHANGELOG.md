# Changelog

## 1.0.1 (2026-10-05)

Fixes from the review of 1.0.0. Anything marked (breaking) changes a call or a log field that 1.0.0 had.

- Quiet hours no longer hold Telegram when no other leg can send. A leg now counts as "will reach someone"
  only if its channel is set up (a Telegram token; an email sender that is configured). With the rules from
  `pingrules init` and only a Telegram token, the Telegram ping is sent and the email leg is reported as failed.
- `pingrules status` and `Router.health()["delivery"]["warnings"]` name every channel that has recipients on an
  active rule but cannot deliver, even while another channel works; `status` exits 1 then.
- A condition row that cannot be understood (a typo such as `op`), a `when` that is not a list, a template under
  an unknown channel and an unknown template part are now reported in `Rules.problems`, so `pingrules check`
  lists them and exits 1. Before, they vanished and the rule matched every event.
- Event labels, descriptions and field labels are HTML-escaped in the default Telegram message, so a `<` in a
  label no longer makes Telegram refuse the alert.
- Preview fires (no `--send`) are labelled: the status line says `(preview, nothing sent)`, `--log` rows carry
  `"preview": true` and `pingrules log` shows `[preview]`. The preview fallback uses only the destinations the
  environment names instead of an always-successful stand-in.
- (breaking) `Router.fire` no longer takes `dry_run`: it never was a dry run, it sent for real. A router built
  with `preview=True` marks its results; `test_fire` marks real test sends. `FireResult.dry_run` and the log
  field `dry_run` are now `test`; both results and rows also carry `preview`. `render_for_channel(dry_run=)` is
  `test=`.
- `truncate_html` closes every tag Telegram accepts (`del`, `ins`, `strike`, `span`, `tg-spoiler`, `tg-emoji`,
  as well as the ones it knew), so a cut message always parses.
- `to_float` reads `-$1,250` as -1250 (a refund is no longer a large positive amount).
- `pingrules check` warns when `mute_channels` lists every channel or an unknown one.
- The `held` fire status could never occur and is gone from the docs and the code.

## 1.0.0 (2026-10-05)

First release.

- Rules in one JSON file: recipients (Telegram chat or email address), events with declared fields, and
  rules with conditions, recipients and templates. Read on every fire. A row that cannot be understood is
  dropped and reported (`Rules.problems`), never silently. `validate` and `pingrules check` list every
  problem in words, including Telegram text that would be refused.
- Conditions on the payload: ten operators, `all`/`any`, dotted paths into nested payloads, numeric
  comparison that fails closed. A condition value of `0` is a value (it is not read as "missing").
- Quiet hours with a named time zone: Telegram is held in the window only when another leg that is not
  held will still reach someone; urgent categories, `urgent=True` and test fires go through; a held leg
  is recorded as held, not as a failure.
- `Router.fire`, `fire_async`, `test_fire` and `health(probe=...)`. A fire returns per-leg results and a
  status (`delivered`, `partial`, `fallback`, `failed`, `held`, `skipped`, `unknown-event`,
  `rules-unreadable`); it never raises, and a failure is never reported as success.
- Fallback destination independent of the rules store, used when the rules cannot be read or every
  delivery attempt failed; `Fallback.status()` says `delivering` or `NOT delivering (reason)`; the last
  attempt can be kept in a file shared between processes.
- Delivery log: capped JSONL, no addresses, chat ids, bodies or payloads, errors scrubbed of emails, phone
  numbers and bot tokens, atomic rewrites, a lock across processes on POSIX, and a read that distinguishes
  "unreadable" from "empty".
- Telegram transport: HTML-safe truncation (never exceeds the limit including closing tags), word-level
  explanations of HTML Telegram would refuse, Telegram's own error text, optional retry (no answer, 429
  with the wait Telegram asks for, 5xx).
- Email: Resend over HTTP and SMTP (STARTTLS or SSL), standard library only; "What changed" and "Details"
  blocks in text and as an email card.
- Command line: `demo`, `init`, `check`, `fire`, `test-fire`, `log`, `status`. Nothing is sent without
  `--send`.
