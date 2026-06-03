## Scope
This file owns only:
- Required verification steps before an agent says work is complete.

This file must not contain:
- Permanent architecture rules, test-writing guidance, or domain acceptance criteria.

Source of truth: mandatory

# Verification

Required checklist before finishing any change. Verify the smallest meaningful scope that proves the change. Report any check you could not run and why.

How to run tests and checks: `docs/workflow/testing.md`. How to check API/OpenAPI: `docs/workflow/openapi-checks.md`.

## Checklist

- [ ] Code compiles / typechecks.
- [ ] Relevant tests pass (see `docs/workflow/testing.md`).
- [ ] API contract checked when an endpoint or schema changed (see `docs/workflow/openapi-checks.md`; CI drift baseline details in `docs/workflow/api-drift-ci.md`).
- [ ] Migration generated and reviewed when the DB schema changed.
- [ ] Frontend flow checked in a browser when UI changed, or final response explains why not.
- [ ] Docs updated when behavior, rules, API, or DB changed (see below).
- [ ] No unrelated changes included.
- [ ] No hidden fallback or legacy-compatibility behavior added.

## When docs must be updated

- Behavior changed.
- API contract changed.
- DB / schema changed.
- Permission / security rule changed.
- Domain / business rule changed.
- Workflow / run command changed.

## When OpenAPI must be checked

See `docs/workflow/openapi-checks.md`. Required when:

- Request / response schema changed.
- Endpoint added, removed, or renamed.
- Status codes or error shape changed.
- Auth / permission behavior changed.

If the changed schema is consumed by the frontend, also update the committed generated type baseline and the manual frontend feature contracts in the same change set, or explicitly report why the frontend is out of scope.
