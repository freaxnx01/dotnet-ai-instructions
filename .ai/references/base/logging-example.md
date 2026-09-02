# Logging — action + outcome example

Every operation logs an **attempt** message, then an **outcome** message, both
as plain sentences a human can read without decoding an event name or code.

```text
[Information] Trying to log in user "jdoe" with Windows Integrated Authentication...
[Information] Successfully logged in user "jdoe" and acquired token.

[Information] Trying to send order confirmation email to "jdoe@example.com"...
[Warning] Failed to send order confirmation email to "jdoe@example.com": SMTP timeout after 5s.
```

- The attempt message names the action and its key inputs (user, resource, target).
- The outcome message restates the action and states success or failure — on
  failure, include the reason (exception message, status code, etc.), not just
  "failed".
- Prefer `Debug` for the attempt message on high-frequency or purely internal
  steps where an `Information`-level attempt would flood the log.
