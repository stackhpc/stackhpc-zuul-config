# stackhpc-zuul-config

Configuration for StackHPC's Zuul (<https://zuul.stackhpc.com>):

* `zuul.d/`, `playbooks/`, `roles/`: the tenant's config project (pipelines,
  jobs, providers).
* `ansible/`: the deployment of Zuul itself.

## Deployment

Zuul runs as podman containers managed by systemd
([quadlets](https://docs.podman.io/en/latest/markdown/podman-systemd.unit.5.html)),
on a single Ubuntu 26.04 VM by default.

| Service | Image | Unit |
|---|---|---|
| ZooKeeper (TLS, port 2281) | `docker.io/library/zookeeper` | `zookeeper` |
| MariaDB | `docker.io/library/mariadb` | `mariadb` |
| Zuul scheduler, web, fingergw, merger, executor, launcher | `quay.io/zuul-ci/zuul-*` | `zuul-<component>` |
| Nodepool launcher | `quay.io/zuul-ci/nodepool-launcher` | `nodepool-launcher` |
| Caddy (TLS via Let's Encrypt, proxy to zuul-web) | `docker.io/library/caddy` | `caddy` |

Layout:

* `ansible/site.yml`: deploys everything, one play per inventory group.
* `ansible/inventory/`: production. `hosts` maps components to hosts;
  `group_vars/all/` holds the Zuul settings, tenant config (`zuul-tenant-config`),
  Nodepool config (`nodepool`), `clouds-yaml` and the vault-encrypted `secrets`
  (see `ansible/secrets.example.yml` for the variables it must define).
* `ansible/roles/`: one role per service. `podman_service` writes a quadlet and
  (re)starts the unit; `zuul_service` wraps it for Zuul components. Services
  restart when their configuration changes. A tenant config change triggers a
  `smart-reconfigure` instead of a restart.
* `ansible/ca.yml`: generates the ZooKeeper CA (run once).
* `ansible/inventory-test/`, `ansible/test-prepare.yml`, `ansible/test.yml`:
  the test deployment run on every pull request.

Versions are pinned in `ansible/inventory/group_vars/all/zuul` (`zuul_version`,
`nodepool_version`) and in role defaults (ZooKeeper, MariaDB, Caddy). Bumping
one restarts the affected services on the next deployment.

### Splitting across VMs

Every component is in its own inventory group (`zookeeper`, `zuul_db`,
`zuul_scheduler`, `zuul_web`, `zuul_fingergw`, `zuul_merger`, `zuul_executor`,
`zuul_launcher`, `nodepool_launcher`, `zuul_proxy`). To move one, add the new
host to `ansible/inventory/hosts` with `ansible_host` and `zuul_internal_address`
and put it in the group instead of `zuul01`. Addresses in `zuul.conf`,
`zoo.cfg`, `nodepool.yaml` and the Caddyfile follow the groups. With more than
one host, ZooKeeper, MariaDB and zuul-web listen on all interfaces, so restrict
ports 2281, 2888, 3888, 3306, 7900 (executor log streaming) and 9000 to the Zuul
hosts. For a ZooKeeper ensemble, put three hosts in `zookeeper`.

### Deploying

Merges are deployed by `.github/workflows/merge.yml` on a self-hosted GitHub
Actions runner on the Zuul VM (`ansible_connection=local`). Until cutover it
only runs on `workflow_dispatch`. The runner is installed by hand:

* runner labels: `self-hosted`, `zuul-deploy`;
* the runner user needs passwordless sudo, and `ansible-core` on its `PATH`;
* create a `production` environment in the repository settings holding the
  `ANSIBLE_VAULT_PASSWORD` secret, restricted to `master`;
* set "Require approval for all external contributors" for fork pull request
  workflows. No PR-triggered workflow targets the `zuul-deploy` label, and
  this is a second guard.

To run by hand on the VM:

```
ansible-galaxy collection install -r ansible/requirements.yml
ansible-playbook -i ansible/inventory ansible/site.yml --ask-vault-pass
```

### Operating

```
systemctl status 'zuul-*' zookeeper mariadb nodepool-launcher caddy
journalctl -u zuul-scheduler -f
podman exec zuul-scheduler zuul-admin --help
podman exec zuul-scheduler zuul-scheduler smart-reconfigure
```

Backups, as systemd timers:

* `mariadb-backup.timer`: database dumps in `/var/backups/mariadb`;
* `zuul-keys-backup.timer`: `zuul-admin export-keys` in
  `/var/lib/zuul/backup`. These are encrypted with the keystore password.

Copy both off the VM.

### Testing

Pull requests run `.github/workflows/pr.yml` on GitHub-hosted `ubuntu-26.04`
runners. The test job:

1. deploys everything to the runner with `ansible/inventory-test`, using
   generated secrets, a tenant built from a local git repository
   (`ansible/test/config`) and a Nodepool static node pointing at the runner
   itself;
2. checks that the services are running, the tenant loads without errors,
   and that jobs succeed on the executor and on the node (`ansible/test.yml`);
3. deploys again and fails if anything changed;
4. adds a tenant and checks that it appears (smart-reconfigure).

On failure the journal and configuration are uploaded as an artifact.

## Migrating from the zuul-operator deployment

1. Populate `ansible/inventory/group_vars/all/secrets` (see
   `ansible/secrets.example.yml`). Generate the ZooKeeper CA with
   `ansible-playbook ansible/ca.yml -e zk_ca_output=...`. Take the keystore
   password from the existing deployment's keystore Secret: the exported keys
   are encrypted with it.
2. Export the project keys from the running deployment:
   `kubectl -n zuul exec zuul-scheduler-0 -- zuul-admin export-keys /tmp/keys.json`
   and copy the file out.
3. Build the VM, install the runner, and deploy with the scheduler stopped:
   `ansible-playbook ... -e zuul_scheduler_start=false`.
4. Copy the export to `/var/lib/zuul/keys.json`, then import it with a
   one-off container (the scheduler is not running yet):
   `podman run --rm --network host -v /etc/zuul:/etc/zuul:ro -v /var/lib/zuul:/var/lib/zuul quay.io/zuul-ci/zuul-scheduler:<version> zuul-admin import-keys /var/lib/zuul/keys.json`
5. Stop the Kubernetes scheduler, then deploy normally to start this one.
6. Point DNS for `zuul.stackhpc.com` at the VM (Caddy then obtains a
   certificate) and update the GitHub app webhook URL.
7. Enable the `push` trigger in `merge.yml` and remove the Kubernetes
   deployment.
