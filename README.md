# Expector - verify, remediate, report.

Modular expected-state checker based on Ansible:
1. Reads a YAML checklist
2. Compares actual host state to what you declared
3. Optionally remediates failures
4. Generates HTML and JSON reports
5. Send email and/or webhook notifications with reports

Users can add their own checks (See [Add a check type](#add-a-check-type)) and their own notification systems (See [Add a notification type](#add-a-notification-type) ). 

Available checks are :

* `services` :
  * Check expected state of systemd units (system or user)
  * Remediation: restart if `expected_state` is `running`, stop if `stopped`; also applies `expected_enabled` when set. After a fix, state **and** enablement are re-checked.

* `podman_containers` :
  * Check expected state of Podman containers (`running`, `stopped`, `present`, `absent`)
  * Remediation: `podman start` or `podman stop`, then re-check.

* `podman_secrets` :
  * Check that named Podman secrets are `present` or `absent`
  * Remediation: none.

* `podman_volumes` :
  * Check that named Podman volumes are `present` or `absent`
  * Remediation: none.

* `command` :
  * Run shell commands and compare exit code (`expected_rc`) and optional stdout+stderr regex (`expected_regex`)
  * Remediation: run `remediation_command` (per entry or check-level), then re-run the original command with the same expectations. Success only if that retest passes.

* `api` :
  * Call HTTP URLs and compare status (`expected_status`) and JSON body keys (`expected_body`). Usually `hosts: localhost`.
  * Remediation: none.

## Requirements

- Ansible (ansible-core 2.14+; developed against 2.21)
- SSH to hosts in `inventory`; Python 3 on the controller (`localhost` is `ansible_connection=local`)
- `community.general` — required only for email (`ansible.builtin.uri` is used for webhooks; no extra collection)

```bash
ansible-galaxy collection install -r requirements.yml -p collections
```

Podman checks use the `podman` CLI on the target. Service checks use systemd.

## Getting started

Work from the project root. You need Ansible (ansible-core 2.14+), Python 3 on the controller, and SSH access to the hosts you will check.

### 1. Install collections

Email notifications need `community.general`. Webhook notifications use `ansible.builtin.uri` (no extra collection). Install the collection next to the playbook and point Ansible at that path:

```bash
ansible-galaxy collection install -r requirements.yml -p collections
```

Add this under `[defaults]` in `ansible.cfg` if it is not already there:

```ini
collections_path = collections
```

Skip this step if you will only write reports or only use webhook notifications (`expector_notify: false`, `--skip-tags notify`, or no email notificator).

### 2. Create an inventory

Edit `inventory`. Keep `localhost` with `ansible_connection=local` (API checks run there). Replace the sample groups with your hosts. Do **not** put hosts in `[expector_reachable]` — the playbook fills that after ping.

```ini
localhost ansible_connection=local

[group2]
my-machine2.example.com
my-machine3.example.com

[expector_reachable]
```

Confirm SSH works: `ansible all -m ping`.

### 3. Edit settings

`settings.yml` is the runner config (paths, report names, history, notifications). Extra vars (`-e`) override it for one run.

At minimum:

| What | Keys |
|---|---|
| Where reports go | `expector_report_dir` (default `reports/`) |
| Report filenames | `expector_report_file_prefix` / `expector_json_file_prefix` (empty JSON prefix copies the HTML prefix) |
| How many archives to keep | `expector_report_keep` (default `10`; `all` or `-1` keeps everything) |
| Force or disable all fixes | `expector_remediate`: `""` = per-check, `true` / `false` = override |

Notifications are optional. Either:

- Set `expector_notify: false`, or run with `--skip-tags notify`, **or**
- Fill `notificators` and `notifications`. Email needs `smtp_host`, `smtp_port`, `smtp_username`, `smtp_password`, `smtp_secure`, `from`, and each message’s `to`. Webhook needs `url` (and `token` / basic auth / `headers` if the endpoint requires them). Encrypt a real SMTP password or webhook token: `ansible-vault encrypt settings.yml`.

To use another settings file: `-e settings_file=/path/to/other-settings.yml`.

### 4. Edit the checklist

`checklist.yml` is the list of checks, in order. Each item needs `name`, `type`, and `hosts` (an inventory group or hostname). Start from the sample: point `hosts` at your groups, set `remediate` the way you want, and replace unit/container/URL names with yours.

- API checks usually use `hosts: localhost`.
- Empty `podman_user` / `service_user` means “the SSH user”.
- Full keys for each `type` are in [Checklist](#checklist-checklistyml).

Another checklist: `-e checklist_file=/path/to/other-checklist.yml`.

### 5. Run the playbook

```bash
ansible-playbook expector.yml
```

First run without notifications:

```bash
ansible-playbook expector.yml --skip-tags notify
```

With a vaulted `settings.yml`:

```bash
ansible-playbook expector.yml --ask-vault-pass
```

Open `reports/<prefix>.html` (default `reports/expector.html`). JSON is `reports/<prefix>.json`. Timestamped copies sit beside them; older archives are pruned per `expector_report_keep`.

Failed checks do not abort the play. Unreachable hosts are skipped and listed as UNREACHABLE in the report.

## How it works

1. Load `settings.yml`, then `checklist.yml`, and build a map of inventory hosts to the checks that target them.
2. Ping every targeted host. Unreachable hosts are excluded from the checklist and recorded with `error_flag: UNREACHABLE`.
3. Walk the checklist **in order** on reachable hosts only. A host is skipped when it is not in that item’s `hosts` group or hostname.
4. For each targeted reachable host, include `tasks/checks/<type>.yml` (Verify → Remediate → Report).
5. Append a standard `check_report` dict to `host_check_results` on that host.
6. Aggregate on localhost: write JSON, render `templates/report.html.j2`, and — only when at least one check failed — render `templates/report-errors.html.j2`. Prune timestamped archives per `expector_report_keep`.
7. Dispatch each enabled notification from `settings.yml`. Notification failures do not abort the playbook.


## Run

```bash
ansible-playbook expector.yml
ansible-playbook expector.yml -e settings_file=/path/to/other-settings.yml
ansible-playbook expector.yml -e expector_remediate=false
ansible-playbook expector.yml --limit group2
ansible-playbook expector.yml --skip-tags notify
```

Extra vars override `settings.yml` for that run.

| Extra var | Effect |
|---|---|
| `settings_file` | Another settings file (default `settings.yml`) |
| `checklist_file` | Another checklist file |
| `expector_remediate=true` / `false` | Force or disable all remediation |
| `expector_report_dir` | Report directory |
| `expector_report_file_prefix` | HTML (and errors) filename prefix |
| `expector_json_file_prefix` | JSON filename prefix (empty → HTML prefix) |
| `expector_report_keep` | Timestamped runs to keep (`all` / `-1` = unlimited) |
| `expector_notify=false` | Skip every notification |

Encrypt `settings.yml` if it contains a real SMTP password or webhook token: `ansible-vault encrypt settings.yml`, then `--ask-vault-pass`.

## Settings (`settings.yml`)

Loaded on every play that needs it. Empty optional strings are treated as unset.

| Key | Default | Meaning |
|---|---|---|
| `checklist_file` | `{{ playbook_dir }}/checklist.yml` | Checklist path |
| `expector_report_dir` | `{{ playbook_dir }}/reports` | HTML, JSON, and errors output directory |
| `expector_report_file_prefix` | `expector` | Prefix for HTML and errors files. Timestamp is appended to archives. |
| `expector_json_file_prefix` | `expector` | Prefix for JSON files. Empty or null → same as HTML prefix. |
| `expector_remediate` | `""` | `""` → honor each check’s `remediate`. `true` / `false` override every check. |
| `expector_report_keep` | `10` | Keep last N timestamped **runs** (HTML+JSON+errors together). Values below 1 count as 1. `all` or `-1` disables pruning. Latest `<prefix>.html` / `.json` / `-errors.html` are never deleted. |
| `expector_notify` | `true` | `false` skips all notifications (same as `--skip-tags notify`) |
| `notification_defaults.enabled` | `true` | Fallback per-system switch |
| `notification_defaults.notify_on` | `[failed, remediated]` | Fallback trigger |

## Inventory

`hosts` on a checklist item is an inventory **group** or **hostname** (`localhost`, `my-machine1.example.com`, or a comma-separated list). Keep `localhost` for API checks that run on the controller. `[expector_reachable]` is filled at runtime after ping — do not list hosts there.

AAP containerized installs typically use **rootless Podman** and **systemd user units** (`become: false`, `user_service: true`).

| Situation | What to set |
|---|---|
| Rootless Podman as the SSH user | `become: false` (default for podman types) |
| Rootless Podman as another account | `podman_user: aap` |
| Rootful Podman | `become: true` |
| systemd user units | `user_service: true` |
| systemd user units of another account | `user_service: true` and `service_user: aap` |
| systemd system units | `user_service: false` and usually `become: true` |

Empty `podman_user` / `service_user` / `url_username` are treated as unset.

## Reports

Directory: `expector_report_dir`. Names use the prefixes above (`<prefix>` = `expector` by default):

| File | When |
|---|---|
| `<prefix>.html` / `<prefix>.json` | Latest full reports (overwritten each run) |
| `<prefix>-<YYYYMMDD-HHMMSS>.html` / `.json` | Archives |
| `<prefix>-errors.html` | Latest errors-only HTML (only when something failed; removed on all-pass) |
| `<prefix>-errors-<YYYYMMDD-HHMMSS>.html` | Errors archive |

HTML: dashboard (KPIs, pass rate, host cards) at the top; one table per host. **Detection** holds the Pass/Fail/Unreachable pill; **Remediation** holds the remediation pill. Failed items (including each unmatched API body key) are extra Item / Expected / Actual rows with empty Detection and Remediation cells.

JSON (every run):

| Key | Content |
|---|---|
| `generator` | `"expector"` |
| `generated_at` | ISO-8601 |
| `summary` | `hosts_checked`, `hosts_unreachable`, `checks_total`, `checks_passed`, `checks_failed` |
| `failures` | `{host, check_name, check_type, check_error, error_flag, check_details}` for checks that did not pass. `check_details` is the failed `details` row from `results` (`result: fail`), as a dict. |
| `host_stats` | Per-host `{passed, failed, total, unreachable}` |
| `results` | Per-host list of `check_report` dicts |

Errors HTML lists only failures: host, message, expected vs actual, remediation (Succeeded / Failed / Not attempted).

Unreachable ping targets skip every checklist item. They appear once as UNREACHABLE (`expected` reachable / `actual` unreachable), count as a failed check, and are included in the errors report and failure notifications.

## Notifications

Configured in `settings.yml`. Run after reports; cannot fail checks or reports. Skip with `--skip-tags notify` or `expector_notify: false`. Empty username/password/token/from and similar fields are `omit`. The send task uses `no_log: true`; a failed send is a debug line and the play continues.

### `notificators`

Shared transports. A notification sets `notificator: <name>`.

| Key | Default | Meaning |
|---|---|---|
| `name` | required | Referenced by `notificator` |
| `type` | required | `smtp` (email) or `webhook` |

#### SMTP (`type: smtp`)

| Key | Default | Meaning |
|---|---|---|
| `smtp_host` | required to send | `host` accepted as alias |
| `smtp_port` | `587` | `25` unencrypted, `587` STARTTLS, `465` implicit TLS |
| `smtp_username` / `smtp_password` | unset | Auth |
| `smtp_secure` | `starttls` | `always` · `never` · `starttls` · `try` |
| `smtp_timeout` | `20` | Seconds |
| `smtp_ehlohost` | unset | EHLO name |
| `from` | unset (module default `root`) | Envelope sender (`sender` alias) |
| `message_id_domain` | unset | `Message-ID` domain |

#### Webhook (`type: webhook`)

| Key | Default | Meaning |
|---|---|---|
| `url` | required to send | Endpoint (`endpoint` accepted as alias) |
| `method` | `POST` | `POST` · `PUT` · `PATCH` |
| `timeout` | `30` | Seconds |
| `validate_certs` | `true` | TLS certificate verification |
| `follow_redirects` | `safe` | Passed to `ansible.builtin.uri` |
| `use_proxy` | `true` | Honor `http(s)_proxy` |
| `status_code` | `[200, 201, 202, 204]` | HTTP statuses treated as success |
| `token` | unset | Bearer token; sets `Authorization: Bearer …` unless that header is already present |
| `url_username` / `url_password` | unset | HTTP basic auth (`username` / `password` aliases) |
| `force_basic_auth` | `false` | Send basic auth without waiting for 401 |
| `headers` | `{}` | Extra headers (dict) |
| `ca_path` / `client_cert` / `client_key` | unset | TLS client/CA files |

### `notifications`

`type` must match `tasks/notify/<type>.yml`.

| Key | Default | Meaning |
|---|---|---|
| `name` | required | Label in logs |
| `type` | required | Task file stem (`email`, `webhook`, …) |
| `enabled` | `notification_defaults.enabled` | Per-system switch |
| `notificator` | required for `email`; for `webhook` required unless `url` is set on the notification | Name of a notificator |
| `charset` | `utf-8` | Email: shared unless a message overrides |
| `headers` | email: `[]` · webhook: `{}` | Email: `Header=value` strings. Webhook: dict merged on top of the notificator |
| `inline` | `[]` | Email: `{path, cid, mime_type}` |
| `attach` | `[]` | Email: extra files on both messages |

`notify_on` (string or list): `failed` (any check failed), `passed` (all passed), `remediated` (any remediation attempted), `always`, `never`.

### `email`

`community.general.mail` from localhost. Two independent messages:

| Block | `enabled` default | Default body |
|---|---|---|
| `errors_report` | `true` if the block exists | `templates/notify/email.html.j2` |
| `full_report` | `false` | `templates/notify/email-full.html.j2` |

Each block has its own `to` / `subject` / `body` / `notify_on` / attachments.

| Key | Default | Meaning |
|---|---|---|
| `enabled` | see above | Send this message |
| `notify_on` | `notification_defaults.notify_on` | When to send |
| `to` | required to send | List or comma-separated string |
| `cc` / `bcc` | `[]` | |
| `reply_to` | unset | `Reply-To` header |
| `from` | notificator `from` | Per-message sender |
| `subject` | auto | Empty → errors: `Expector: N failure(s)` / `remediation executed` / `all checks passed`; full: `Expector: full report` |
| `subtype` | `html` | `html` or `plain` |
| `body` | empty | Empty → default template. `report` → generated full HTML as body. `errors_report` → generated errors HTML as body. A `.j2` path is rendered. Any other string is sent literally. |
| `attach_report` | errors: `false` · full: `true` | Attach full HTML report |
| `attach_errors_report` | errors: `true` · full: `false` | Attach errors HTML when it exists |
| `attach` / `charset` / `headers` / `inline` / `message_id_domain` | parent or type default | Per-message overrides |

### `webhook`

`ansible.builtin.uri` from localhost (no extra collection). One POST (or `method`) per notification. There is no `errors_report` / `full_report` split; `notify_on` on the notification decides when to send.

Notification-level `url`, auth, headers, timeout, and TLS keys override the named notificator when set (empty string = unset / inherit). `notificator` may be omitted when `url` is set on the notification.

| Key | Default | Meaning |
|---|---|---|
| `notify_on` | `notification_defaults.notify_on` | When to POST |
| `url` | notificator `url` | Endpoint override (`endpoint` alias) |
| `method` | notificator / `POST` | `POST` · `PUT` · `PATCH` |
| `timeout` | notificator / `30` | Seconds |
| `validate_certs` | notificator / `true` | TLS certificate verification |
| `follow_redirects` | notificator / `safe` | Passed to `uri` |
| `use_proxy` | notificator / `true` | Honor `http(s)_proxy` |
| `status_code` | notificator / `[200, 201, 202, 204]` | HTTP statuses treated as success |
| `token` | notificator | Bearer token; sets `Authorization: Bearer …` unless that header is already present |
| `url_username` / `url_password` | notificator | HTTP basic auth (`username` / `password` aliases) |
| `force_basic_auth` | notificator / `false` | Send basic auth without waiting for 401 |
| `headers` | `{}` | Dict merged on top of notificator headers |
| `body` | empty | Empty, `json_report`, or `report` → generated JSON report (`generator`, `generated_at`, `summary`, `failures`, `host_stats`, `results`). A `.j2` path is rendered. Any other string is sent literally. |
| `body_format` | `json` | `json` · `raw` · `form-urlencoded`. For the JSON report, `json` sends a JSON object and sets `Content-Type: application/json` (encoded once). `raw` sends the same JSON string; `Content-Type: application/json` is added if you did not set it. |
| `ca_path` / `client_cert` / `client_key` | notificator | TLS client/CA files |

The JSON report is read from the in-memory `expector_json_report` fact, or from `expector_json_file` if that fact is missing. If neither exists, the webhook is skipped.

### Add a notification type

Do not edit `expector.yml` or `run_notify.yml`.

1. Create `tasks/notify/<type>.yml` (example: `xmatter.yml` for `type: xmatter`).
2. Read `current_notify`. Flags already set: `_notify_failed`, `_notify_passed`, `_notify_remediated`, `_notify_events` (`{host, check}`).
3. Wrap the send in `block` / `rescue`. Log failures; do not fail the Ansible task.
4. `no_log: true` on anything with secrets.
5. Add a fully specified entry to `settings.yml`. Confirm disabled / unmatched `notify_on` is skipped, and a send error does not abort the play.

## Checklist (`checklist.yml`)

Top-level key `checklist` — a list, executed in order.

### Common keys

| Key | Default | Meaning |
|---|---|---|
| `name` | required | Label in the report |
| `type` | required | Must match `tasks/checks/<type>.yml` |
| `hosts` | required | Group, hostname, or comma-separated list |
| `report` | `true` | Print `check_report` with `debug` |
| `remediate` | `false` | Run Remediate when Verify failed |
| `become` | type-specific | Privilege escalation for that check |

### `services`

Systemd units (system or user). Remediation: restart if `expected_state` is `running`, stop if `stopped`; also applies `expected_enabled` when set. After a fix, state **and** enablement are re-checked.

| Key | Default | Meaning |
|---|---|---|
| `services` | required | Unit names |
| `expected_state` | `running` | `running` or `stopped` |
| `expected_enabled` | unset (not checked) | `true` / `false` |
| `user_service` | `false` | `true` → `systemctl --user` |
| `service_user` | unset | Username for the user systemd instance |
| `become` | `true` unless `user_service: true` | |

### `podman_containers`

Uses `podman ps -a` / `podman ps`. Remediation: `podman start` or `podman stop`, then re-check.

| Key | Default | Meaning |
|---|---|---|
| `containers` | required | Names |
| `expected_state` | `running` | `running`, `stopped`, `present`, `absent` |
| `become` | `false` | Rootless as the SSH user |
| `podman_user` | unset | Another account’s rootless Podman |

### `podman_secrets`

`podman secret ls`. No remediation.

| Key | Default | Meaning |
|---|---|---|
| `secrets` | required | Names |
| `expected_state` | `present` | `present` or `absent` |
| `become` | `false` | |
| `podman_user` | unset | |

### `podman_volumes`

`podman volume ls`. No remediation.

| Key | Default | Meaning |
|---|---|---|
| `volumes` | required | Names |
| `expected_state` | `present` | `present` or `absent` |
| `become` | `false` | |
| `podman_user` | unset | |

### `command`

A single top-level `command` is accepted if `commands` is omitted. Remediation runs only when `remediate: true` and a `remediation_command` is non-empty (per entry, else check-level). After the fix, the original command is re-run and scored with the same `expected_rc` / `expected_regex`. Success only if that retest passes.

| Key | Default | Meaning |
|---|---|---|
| `commands` | required | List of command entries |
| `become` | `true` | |
| `remediation_command` | `""` | Fallback fix for every failed command |

Each `commands` entry:

| Key | Default | Meaning |
|---|---|---|
| `command` | required | Shell command |
| `expected_rc` | `0` | |
| `expected_regex` | unset | Applied to stdout+stderr |
| `remediation_command` | check-level or `""` | |

### `api`

Usually `hosts: localhost`. No remediation. A single top-level `url` is accepted if `api` is omitted. Each unmatched JSON body key is its own report row (`details[].mismatches`).

| Key | Default | Meaning |
|---|---|---|
| `api` | required | List of HTTP calls |
| `validate_certs` | `false` | Default for each URL |
| `url_username` / `url_password` | unset | Basic auth fallback |

Each `api` entry:

| Key | Default | Meaning |
|---|---|---|
| `url` | required | |
| `method` | `GET` | |
| `expected_status` | `200` | |
| `expected_body` | `[]` | `{key, value}` (dot-path keys allowed) |
| `timeout` | `30` | Seconds |
| `validate_certs` | check-level or `false` | |
| `headers` | `{}` | |
| `body` / `body_format` | unset | Omitted when empty |
| `url_username` / `url_password` | check-level or unset | |
| `force_basic_auth` | `false` | |
| `follow_redirects` | `safe` | |

## Add a check type

The dispatcher loads `tasks/checks/{{ current_check.type }}.yml`. Do **not** edit `expector.yml` or `run_check.yml`. Copy a small type (`podman_secrets.yml`) or the skeleton below. Filename stem must match `type:` (e.g. `tasks/checks/disk_space.yml` ↔ `type: disk_space`).

### Sections (in order)

1. **Verify** — compare state; wrap in `block` / `rescue`; `failed_when: false` on commands that may fail.
2. **Remediate** — when `_do_remediate` and Verify failed (or a no-op). Retest the **same** Verify criteria; `_remediate_success` is true only if the retest passes.
3. **Report** — always include `_report.yml`.

```yaml
---
- name: Disk space | Verify
  block:
    - name: Disk space | Verify | Require spec
      when: current_check.paths is not defined or current_check.paths | length == 0
      ansible.builtin.set_fact:
        _spec_invalid: true
        check_success: false
        check_msg: "Check '{{ current_check.name }}' is missing required field 'paths'"
        check_error: "Required field 'paths' is missing or empty"
    # gather data; append _check_details; then:
    - name: Disk space | Verify | Set check status
      when: not (_spec_invalid | bool)
      ansible.builtin.set_fact:
        check_success: "{{ _failures | length == 0 }}"
        check_msg: "{{ 'ok' if (_failures | length == 0) else (_failures | length | string ~ ' paths below threshold') }}"
        check_error: "{{ _failures | map(attribute='error') | reject('equalto', '') | list | join('; ') }}"
      vars:
        _failures: "{{ _check_details | selectattr('result', 'equalto', 'fail') | list }}"
  rescue:
    - ansible.builtin.set_fact:
        check_success: false
        check_msg: "Disk space verification raised an unexpected error"
        check_error: "{{ ansible_failed_result.msg | default(ansible_failed_result | string) }}"

- name: Disk space | Remediate
  when: _do_remediate | bool
  ansible.builtin.set_fact:
    _remediate_attempted: false
    _remediate_success: false
    _remediate_msg: "No remediation is available for check type 'disk_space'"
    _remediate_error: ""

- name: Disk space | Report
  ansible.builtin.include_tasks:
    file: "{{ playbook_dir }}/tasks/checks/_report.yml"
```

If remediation **is** implemented, gate with `_do_remediate`, `not check_success`, `not _spec_invalid`; retest Verify criteria; `rescue` must set `_remediate_attempted: true` and `_remediate_success: false`.

Use `current_check.become | default(...)`. Do not assume play-level `become`. Do not fail the Ansible task to signal a failed check.

### Dispatcher variables (`run_check.yml` resets these)

| Variable | Meaning |
|---|---|
| `current_check` | Current checklist item |
| `_do_remediate` | Whether to attempt a fix |
| `_check_details` | Start `[]`; one dict per item |
| `_spec_invalid` | Set `true` when required fields are missing |
| `check_success` / `check_msg` / `check_error` | Set in Verify |
| `_remediate_attempted` / `_remediate_success` / `_remediate_msg` / `_remediate_error` | Set in Remediate |

### Facts before Report

| Fact | Required | Meaning |
|---|---|---|
| `check_success` | yes | Verify passed |
| `check_msg` | yes | Short summary |
| `check_error` | yes | Failure text, or `""` |
| `_check_details` | recommended | Item / Expected / Actual rows |

`check_failed` is `not check_success` in `_report.yml`. `check_report` also includes `check_name`, `check_type`, `check_host`, `check_unreachable`, `error_flag`, remediation fields, and `details`. If `check_report.check_name` is missing, the dispatcher includes `_report.yml` once as a fallback.

### `_check_details` keys (HTML Item / Expected / Actual)

Each dict: `result: pass` or `fail`.

| Key | Used when |
|---|---|
| `name` | Services, podman, generic |
| `command` | `command` type |
| `url` | `api` type |
| `expected_state` / `actual_state` | State checks |
| `expected_enabled` / `actual_enabled` | Services enablement |
| `expected_rc` / `actual_rc` | Commands |
| `expected_regex` / `stdout` | Command output |
| `expected_status` / `actual_status` | API HTTP status |
| `mismatches` | API: `{kind, key, url, method, expected, actual}` per unmatched body key, status, or request error |
| `error` | Fallback when no expected/actual pair exists |

```yaml
- name: Disk space | Verify | Evaluate each path
  when: not (_spec_invalid | bool)
  ansible.builtin.set_fact:
    _check_details: "{{ _check_details + [_item] }}"
  loop: "{{ current_check.paths }}"
  vars:
    _item:
      name: "{{ item }}"
      expected_state: "{{ current_check.expected_min_free_percent }}% free"
      actual_state: "{{ _actual_free }}% free"
      result: "{{ 'pass' if (_ok | bool) else 'fail' }}"
      error: "{{ '' if (_ok | bool) else (item ~ ' is below the threshold') }}"
```

List every option for the type in `checklist.yml`, using defaults for unused ones. Do not edit `expector.yml`, `run_check.yml`, `compile_host_checks.yml`, or `report.yml`. Edit report templates only for new expected/actual field names; edit `_report.yml` only for new `check_report` keys.

```bash
ansible-playbook expector.yml --syntax-check
ansible-playbook expector.yml --limit <target-group>
```

Confirm: non-targeted hosts skip; pass → `check_success: true` and empty `check_error`; fail → precise `check_error` and `details[].result: fail`; `remediate: false` skips or no-ops Remediate; HTML Detection / Remediation look right; unknown `type` yields `Unknown check type` instead of aborting.
