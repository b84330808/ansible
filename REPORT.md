# Cloud report: ansible-core `serial` percentage off-by-one

## Status

- Fix branch: `claude/bold-thompson-zv0pdv` on https://github.com/b84330808/ansible
- Fix commit: `9ea6faca313ccfaf7a965477171cab648e53e48d`
- Base: `upstream/devel` at `efb0665` ("Update AZP container (#87823)"). The branch is one commit on top of it.
- Nothing was opened, edited or commented on upstream.
- **Do not open the PR yet.** ansibot lets a first-time contributor have only one open PR, and https://github.com/ansible/ansible/pull/87817 (IPv6 prefix) is still open. Open this one only after #87817 is merged or closed.

Files changed:

- `lib/ansible/utils/helpers.py`: the fix (one line, plus a one-line comment).
- `test/units/utils/test_helpers.py`: 4 new `pct_to_int` cases.
- `changelogs/fragments/serial-percentage-rounding.yml`: `bugfixes` fragment.

Commit identity and trailers: author and committer are `Wei-Lun Li <b84330808@gmail.com>`, with `Signed-off-by` (DCO, from `git commit -s`), `Assisted-by: Claude Code` and `Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>`.
The commit also has a `Claude-Session:` trailer that links to the cloud session.
Drop it with `git commit --amend` before opening the PR if you don't want that link upstream; the DCO sign-off is unaffected.

## The bug

`serial: "N%"` is turned into a batch size by `ansible.utils.helpers.pct_to_int()`, called from `PlaybookExecutor._get_serialized_batches()`:

```python
return int((value_pct / 100.0) * num_items) or min_value
```

Dividing first makes the intermediate value inexact, so a result that should be a whole number of hosts can land just below it, and `int()` truncates it to one fewer:

```text
>>> (29 / 100.0) * 100
28.999999999999996
```

So `serial: "29%"` with 100 hosts runs batches of 28, not 29.
Exhaustive check over 1-100% and 1-1000 hosts: 49 of 100,000 combinations come out one host short (e.g. 29%, 57% and 58% of 100; 58% of 50; 70% of 90).

Fix: multiply first, `int(value_pct * num_items / 100)`.
The integer product is exact, so a single correctly rounded division can't fall below a whole-number result.
I checked this exhaustively against exact `Fraction` arithmetic over -200..1000% and 0..3000 hosts: the old form was wrong in 5,979 cases and the new form in 0. A further 2,000,000 random samples with up to 10,000,000 hosts also had 0 mismatches.
Truncation, negative-input behavior and `min_value` are unchanged, because `int()` still truncates toward zero.
I avoided `// 100` on purpose, since it floors negative values and would change the (undocumented) behavior for negative percentages.

## PR title

```text
serial - fix percentage batch size lost to float rounding
```

## PR body (Bugfix template)

```markdown
##### SUMMARY

`serial` percentages are turned into a batch size by `pct_to_int()`, which computed `int((value_pct / 100.0) * num_items)`.
Dividing first makes the intermediate value inexact, so a result that should be a whole number of hosts can come out just below it, and `int()` then truncates it to one host fewer:

    >>> (29 / 100.0) * 100
    28.999999999999996

So with 100 hosts, `serial: "29%"` runs batches of 28 hosts instead of 29.
The same happens for 57% and 58% of 100 hosts, 58% of 50, 70% of 90 and so on (49 of the 100,000 combinations of 1-100% and 1-1000 hosts).

This multiplies before dividing, `int(value_pct * num_items / 100)`, so the integer product is exact and a whole-number result can't be rounded below itself.
Otherwise behavior is unchanged: the result is still truncated (this is not the round-to-nearest change proposed in #61853), and `min_value` still applies when the result is 0.

Steps to reproduce, with 100 hosts using `ansible_connection=local`:

    - hosts: all
      gather_facts: false
      serial: "29%"
      tasks:
        - debug:
            msg: "batch size {{ ansible_play_batch | length }}"
          run_once: true

Before: batches of 28, 28, 28, 16.
After: batches of 29, 29, 29, 13.

##### ISSUE TYPE

- Bugfix Pull Request

I used Claude Code to write this change.
```

## Fail-before (fix reverted, new tests kept)

`ansible-test units --venv --python 3.13 test/units/utils/test_helpers.py` with `lib/ansible/utils/helpers.py` restored to `upstream/devel`:

```text
4 workers [12 items]
......F.FF.F                                                             [100%]
=================================== FAILURES ===================================
______________ test_pct_to_int[string_percentage_exact_29_of_100] ______________
value = '29%', num_items = 100, min_value = 1, expected = 29
>       assert pct_to_int(value, num_items, min_value) == expected
E       AssertionError: assert 28 == 29
E        +  where 28 = pct_to_int('29%', 100, 1)
______________ test_pct_to_int[string_percentage_exact_70_of_90] _______________
value = '70%', num_items = 90, min_value = 1, expected = 63
E       AssertionError: assert 62 == 63
E        +  where 62 = pct_to_int('70%', 90, 1)
______________ test_pct_to_int[string_percentage_exact_58_of_50] _______________
value = '58%', num_items = 50, min_value = 1, expected = 29
E       AssertionError: assert 28 == 29
E        +  where 28 = pct_to_int('58%', 50, 1)
______________ test_pct_to_int[string_percentage_exact_57_of_100] ______________
value = '57%', num_items = 100, min_value = 1, expected = 57
E       AssertionError: assert 56 == 57
E        +  where 56 = pct_to_int('57%', 100, 1)
=========================== short test summary info ============================
FAILED test/units/utils/test_helpers.py::test_pct_to_int[string_percentage_exact_29_of_100]
FAILED test/units/utils/test_helpers.py::test_pct_to_int[string_percentage_exact_70_of_90]
FAILED test/units/utils/test_helpers.py::test_pct_to_int[string_percentage_exact_58_of_50]
FAILED test/units/utils/test_helpers.py::test_pct_to_int[string_percentage_exact_57_of_100]
========================= 4 failed, 8 passed in 0.83s ==========================
```

## Pass-after (fix applied)

```text
$ ansible-test units --venv --python 3.13 test/units/utils/test_helpers.py
============================== 12 passed in 0.84s ==============================

$ ansible-test units --venv --python 3.13 test/units/executor/test_playbook_executor.py test/units/utils/test_helpers.py
============================== 13 passed in 0.91s ==============================
```

`test_playbook_executor.py::test_get_serialized_batches` already covers `serial: 20%` and `[1, "20%", -1]`, and it still passes.

End to end, with real `ansible-playbook`, 100 local hosts and the playbook from the PR body:

```text
== before (upstream/devel)
    "msg": "batch size 28"
    "msg": "batch size 28"
    "msg": "batch size 28"
    "msg": "batch size 16"
== after (fix)
    "msg": "batch size 29"
    "msg": "batch size 29"
    "msg": "batch size 29"
    "msg": "batch size 13"
```

## Sanity

`ansible-test sanity --python 3.13 lib/ansible/utils/helpers.py test/units/utils/test_helpers.py changelogs/fragments/serial-percentage-rounding.yml`: exit 0, 40 tests run, none failed.
A verbose re-run confirmed that pep8, pylint, mypy, changelog and boilerplate actually ran against these files.
`validate-modules` warned that no base commit was given, which doesn't matter here because no modules changed.

How this was run: `ci-files.testing.ansible.com` (ansible-test's pip bootstrap host) is blocked in the cloud sandbox.
I saved a local stand-in at `~/.ansible/test/cache/get_pip_26_2_2.py` that runs the stdlib `ensurepip` and then `pip install`s the requested packages from PyPI.
The test requirements themselves came from PyPI as normal. Nothing in the repo was changed for this.

## Duplicate search (ansible/ansible, all states)

| Query | Result |
| --- | --- |
| PRs: `pct_to_int` | https://github.com/ansible/ansible/pull/61853 (closed, 2019, "round before int casting"): a **feature** PR to switch from truncation to round-to-nearest (2.6 -> 3). Labeled `needs_revision`, never merged. A different change: this fix keeps truncation and only removes the float error. Also https://github.com/ansible/ansible/pull/1777 (2012, parsing serial from extra_vars), unrelated. |
| Issues: `pct_to_int` | none |
| PRs: `serial percentage` | 12 results, none about float truncation. Open ones: https://github.com/ansible/ansible/pull/86620 (linear strategy `max_fail_percentage` calculation, different code) and https://github.com/ansible/ansible/pull/84914 (warnings), both unrelated. |
| PRs: `serial batch rounding`, `utils helpers percentage` | only #61853 |
| PRs: `_get_serialized_batches` | 8 old results (2012-2018), none about this |
| Issues: serial percentage batch size wrong / off by one / float | nothing matching. https://github.com/ansible/ansible/issues/14344 (2016, "Serial percentage does not work") and https://github.com/ansible/ansible/issues/68661 (2020, `0%`) are closed and about different problems. |

`max_fail_percentage` (`lib/ansible/plugins/strategy/linear.py`) was checked too. It compares `failed / batch_size > pct / 100.0`, which doesn't have this problem because both sides are single correctly rounded divisions.

## Candidates found but skipped

Each of these is real, but it's already reported upstream by someone else, or it was a weaker candidate:

- `ImmutableDict.__eq__` compares hashes, so `ImmutableDict(a=-1) == ImmutableDict(a=-2)` is True. Taken: issue #87397 and PR #87407 (dwin-gharibi, bot-closed).
- `regex_replace` `mandatory_count` error reports `count` instead of the real match count. Taken: issue #87399 and PR #87409 (same contributor).
- `camel_dict_to_snake_dict` drops `ignore_list` when it recurses. Taken: issue #87394 and PR #87404.
- `snake_dict_to_camel_dict` raises `TypeError` for any value whose type can't be built with no arguments (e.g. `datetime.date`), because `camelize()` calls `type(x)()` before checking for dict/list. No duplicate found. Not chosen because module params are JSON types, so it's unlikely in practice. It's a possible follow-up.
- `human_to_bytes`: the unit names are misspelled `zetabyte`/`zetabit`, so `1 zettabyte` is rejected. Low impact.
- galaxy `meets_requirements('1.5.0', '>=1.0.0, <2.0.0')` (space after the comma) raises `ValueError`. Debatable whether that's supported syntax.
