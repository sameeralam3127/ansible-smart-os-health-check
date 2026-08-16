## What this changes

<!-- One or two sentences. Link the issue it closes: Closes #123 -->

## Why

<!-- The problem, not the patch. If it fixes a bug, what was the wrong
     behaviour and what triggered it? -->

## How it was verified

<!-- Delete what does not apply. CI runs all of these, but say what you ran
     locally and against what. -->

- [ ] `ansible-lint roles/ playbooks/ molecule/`
- [ ] `ansible-playbook playbooks/healthcheck.yml --syntax-check`
- [ ] `pytest -q`
- [ ] `molecule test -s <scenario>` (which distributions?)
- [ ] Ran against a real host / fleet (which distributions?)

## Checklist

- [ ] `CHANGELOG.md` updated
- [ ] Docs updated (`docs/`, role `README.md`) if behaviour or variables changed
- [ ] New variables documented in `docs/variable-reference.md`
- [ ] No secrets, hostnames, or customer data in fixtures, tests, or examples
