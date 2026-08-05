# ansible-role-fluentbit

General Fluent Bit → OpenSearch log shipper. **Single source of truth** for all
projects (TayanchBank, MKB, …) — consume it via `requirements.yml`, do not fork
per-repo copies.

One role, two deployment modes, selected with `fluentbit_mode`:

| mode | what it does | origin |
|---|---|---|
| `package` | Installs the native fluent-bit package (apt/yum), runs under systemd, tails arbitrary files into OpenSearch daily indices | TayanchBank edge-nginx access analytics |
| `swarm` | Deploys the official container as a Docker Swarm **global** service, tails Docker's json-file container logs, enriches with stack/service/image via Lua (no docker.sock), routes to per-project indices | MKB container logging |

Both modes share: OpenSearch output settings (`fluentbit_os_*`), filesystem
buffering (outage ⇒ queue to disk, bounded), optional `/etc/hosts` pinning,
and a non-fatal preflight check of the OpenSearch endpoint.

## Consuming (requirements.yml)

```yaml
roles:
  - name: fluentbit
    version: main            # pin a tag for reproducible runs
    src: git@github.com:anyopsorg/ansible-role-fluentbit.git
    scm: git
```

```sh
ansible-galaxy install -r requirements.yml --force
```

## package mode — file tail (TayanchBank nginx example)

```yaml
- hosts: edge-nginx
  become: true
  roles:
    - role: fluentbit
      vars:
        fluentbit_mode: package
        fluentbit_os_host: 10.77.12.196
        fluentbit_os_port: 9200
        fluentbit_os_tls_verify: "Off"        # self-signed cluster REST certs
        fluentbit_os_user: fluentbit
        fluentbit_os_password: "{{ vault_fluentbit_opensearch_password }}"
        fluentbit_logstash_prefix: nginx-access
        fluentbit_parsers:
          - name: nginx_json
            format: json
            time_key: msec                    # nginx $msec -> ms-precision @timestamp
            time_format: "%s.%L"
            time_keep: "On"
        fluentbit_tail_inputs:
          - tag: nginx.access
            path: /var/log/nginx/json_access.log
            parser: nginx_json
            records:
              log_host: "{{ inventory_hostname }}"
```

## swarm mode — Docker container logs (MKB example)

```yaml
- hosts: swarm-managers
  roles:
    - role: fluentbit
      vars:
        fluentbit_mode: swarm
        fluentbit_image: harbor.mkb.uz/library/fluent/fluent-bit
        fluentbit_version: "4.1.1"
        fluentbit_os_host: opensearch.mkb.uz
        fluentbit_os_port: 443
        fluentbit_os_tls_verify: "On"         # public wildcard via gateway
        fluentbit_os_user: fluentbit
        fluentbit_os_password: "{{ fluentbit_os_password_vault }}"
        fluentbit_environment: prod
        # index: <project>-prod-logs-YYYY.MM.DD, project derived per-container
```

Swarm-mode extras (see `defaults/main.yml` for the full documented list):
Java stack-trace folding (`fluentbit_multiline_java`), JSON app-log lifting
(`fluentbit_parse_json_logs`), ECS mode (`fluentbit_ecs_mode` — read the
comment before enabling), W3C trace-id extraction
(`fluentbit_extract_trace_ids`), project excludes, memory caps.

## Variable migration from the old per-repo roles

| old (TAYANCH `fluent-bit`) | old (MKB `fluentbit`) | now |
|---|---|---|
| `fluentbit_opensearch_host` | `fluentbit_os_host` | `fluentbit_os_host` |
| `fluentbit_opensearch_port` | `fluentbit_os_port` | `fluentbit_os_port` |
| `fluentbit_opensearch_tls` | `fluentbit_os_tls` | `fluentbit_os_tls` |
| `fluentbit_opensearch_tls_verify` | `fluentbit_os_tls_verify` | `fluentbit_os_tls_verify` |
| `fluentbit_opensearch_user` | `fluentbit_os_user` | `fluentbit_os_user` |
| `fluentbit_opensearch_password` | `fluentbit_os_password` | `fluentbit_os_password` |
| `fluentbit_nginx_json_log` | — | `fluentbit_tail_inputs[].path` |
| — | (implicit docker input) | `fluentbit_mode: swarm` |

MKB's variable names are unchanged; TayanchBank playbooks set the
`fluentbit_os_*` names and describe the nginx tail via `fluentbit_tail_inputs`.
