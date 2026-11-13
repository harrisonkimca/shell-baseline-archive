# Requirement tests

Every row in [spec/auth.md](../spec/auth.md) is proved by its required
verification kinds, identified by its requirement ID.

1. Read the row. *Acceptance* is what must be proved; *Verification*
   is where it is proved:
   - unit test: on fakes, mirroring the module;
   - integration suite: against the real stack, with real Ory from stage 1;
   - configuration check: a test of settings, startup or grants, or
     an inspection of the deployed environment;
   - hands-on check or load test: owner-run verification.
2. Write the automated test, see it fail for the right reason, then build.
3. Tag it. Pytest uses `req` markers; other runners associate their test
   names with the requirement IDs in test metadata. Results identify the
   ID, verification kind and any named verification part.

```python
@pytest.mark.req("AUTH-CODE-004")
async def test_resend_within_60_seconds_sends_nothing(sender, clock, outbox) -> None:
    await sender.send_code(to=ADDRESS, purpose="sign_in", code="123456", locale="en")
    clock.advance(seconds=59)
    await sender.send_code(to=ADDRESS, purpose="sign_in", code="654321", locale="en")
    assert outbox.count(to=ADDRESS) == 1
```

- Assert the Acceptance line's exact values. For elapsed-time checks,
  use an injected clock in Shell or controlled provider time. If the
  provider's time cannot be controlled, check its exact production
  settings and exercise the same behaviour with shorter test-only
  lifetimes against the real provider.
- One behaviour per test, named for the behaviour, not the ID.
- Synthetic data only (NFR-010).
- Name separately verified parts, including those due in later stages.
  Requirements covering a growing set are checked as members are added.
- Automated verification counts only executed, passing tests, not
  collection, skips, expected failures or a green job alone.
- Owner-run checks are recorded as described in the workflow runbook,
  with their ID or exit criterion and evidence.
- Never skip or weaken a tagged test to turn CI green.
