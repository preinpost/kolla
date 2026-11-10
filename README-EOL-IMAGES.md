# Building kolla images for end-of-life OpenStack releases

This fork is a utility fork of [openstack/kolla](https://github.com/openstack/kolla).
It exists for one purpose: rebuild the kolla container images for OpenStack
releases that are no longer published to a public registry, and push them to
GitHub Container Registry (GHCR) so that an existing deployment (for example a
kolla-ansible 15.6.0 "zed" cluster) can keep pulling its images.

Upstream kolla stops publishing once a release becomes unmaintained, so
`quay.io/openstack.kolla/<image>:<release>-<distro>-<version>` starts returning
`manifest unknown` after a while. The Dockerfile templates are still in git,
therefore the images can be rebuilt — the only thing that disappeared is the
published artifact.

## What gets built

`.github/kolla/groups.json` is the single source of truth for the image list.
It is keyed by OpenStack release and each group has a `phase`:

* `phase: 0` — parent images (`base`, `openstack-base`). They are built and
  pushed first, because everything else derives from them.
* `phase: 1` — leaf services. These jobs pull the phase 0 images back from the
  registry and build on top of them (`--skip-existing`), which keeps each job
  inside the disk budget of a free GitHub-hosted runner.

Image selection is done with kolla-build positional regexes, so a group is built
with exactly the regexes listed in `groups.json` (for example `^nova-base$` and
`^nova-api$`).

The set covers a core deployment (keystone, glance, placement, nova, neutron,
horizon, mariadb, rabbitmq, memcached, haproxy/keepalived, openvswitch) plus
`proxysql`, which `kolla-ansible` 2024.2 turns on by default and which is built
for 2024.1 and 2024.2 alike so both image sets stay interchangeable. Images for
services a deployment does not enable (cinder, heat, ceilometer, barbican,
octavia, ironic, swift, ...) are not built; add them to a group in
`groups.json` when needed.

## Running it

From the Actions tab: **eol-images** → **Run workflow**, or from the CLI:

```bash
gh workflow run eol-images.yml \
  --repo preinpost/kolla \
  -f release=2024.1 \
  -f kolla_ref=unmaintained/2024.1 \
  -f base_distro=rocky \
  -f base_distro_version=9 \
  -f image_tag=2024.1-rocky-9 \
  -f groups=all \
  -f push_images=true \
  -f registry=ghcr.io \
  -f namespace=preinpost/kolla
```

The three published sets were built with these inputs:

| release | `kolla_ref` | kolla version shipped | `image_tag` |
| --- | --- | --- | --- |
| zed (2022.2) | `zed-eol` (tag) | 15.6.0 | `zed-rocky-9` |
| 2024.1 (Caracal) | `unmaintained/2024.1` (branch head, `ed8a1ef`) | 18.8.1 | `2024.1-rocky-9` |
| 2024.2 (Dalmatian) | `2024.2-eol` (tag, `a77ec9d`) | 19.8.0 | `2024.2-rocky-9` |

Prefer the last surviving commit of the series: the `unmaintained/<release>`
branch when it still exists, otherwise the `-eol`/`-eom` tag. 2024.2 only has
the tag; 2024.1 kept an `unmaintained` branch, so its head is used.

`groups` accepts `all`, `base`, or a comma separated subset of the group names
(`base`, `infra`, `keystone-glance-placement-horizon`, `nova`, `neutron`). Use
`groups=base` to validate the platform before spending time on the services.

Inputs:

| input | meaning |
| --- | --- |
| `release` | kolla-build `--openstack-release` (drives the source tarball branch, e.g. `stable-2024.1`) |
| `kolla_ref` | git ref of `preinpost/kolla` to build from; use the release's last tag or `unmaintained/<release>` |
| `base_distro` / `base_distro_version` | kolla-build `--base` / `--base-tag` |
| `image_tag` | tag applied to every image; must match what kolla-ansible computes as `openstack_tag` |
| `registry` / `namespace` | push destination, e.g. `ghcr.io` + `preinpost/kolla` |
| `push_images` | set to `false` to build without pushing, useful for debugging |
| `public_packages` | try to flip the pushed GHCR packages to public after pushing |

Each job logs in with the workflow's `GITHUB_TOKEN`
(`permissions: packages: write`), so no extra secret is needed for pushing.

## Image naming and kolla-ansible wiring

Images are pushed as `<registry>/<namespace>/<service>:<image_tag>`, for example
`ghcr.io/preinpost/kolla/keystone:2024.1-rocky-9`. Because the tag is chosen to
be exactly what kolla-ansible computes for the release, a deployment only has to
override the registry and namespace:

```yaml
# /etc/kolla/globals.yml
docker_registry: "ghcr.io"
docker_namespace: "preinpost/kolla"
```

For all three releases that yields `openstack_tag=<release>-rocky-9`
(`{{ openstack_release }}-{{ kolla_base_distro }}-{{ kolla_base_distro_version }}`).
`docker_namespace` defaults to `openstack.kolla`, `kolla_base_distro` to
`rocky` and `kolla_base_distro_version` to `9` for 2024.1 and 2024.2 (zed maps
rocky to `9` as well), so nothing else needs to change. Keep
`openstack_tag_suffix` at its empty default. Verify with:

```bash
grep -E '^(openstack_release|openstack_tag|kolla_base_distro|docker_registry|docker_namespace)' \
  /usr/share/kolla-ansible/ansible/group_vars/all.yml
ansible -i /etc/kolla/all-in-one -m debug -a "var=openstack_tag" all
```

Things a stock deployment would otherwise ask for that this set does **not**
contain:

* **`enable_heat`** — `kolla-ansible` 2024.1 and 2024.2 default it to
  `enable_openstack_core`, so a default `deploy` also wants `heat-api`,
  `heat-engine` and `heat-api-cfn`. Those images are not built; set
  `enable_heat: "no"` in `globals.yml` (or add a `heat` group to
  `groups.json` and rebuild).
* Everything else that defaults to on is covered: `proxysql` (2024.2 default,
  built for both), `fluentd` (`enable_fluentd: yes`), `haproxy`/`keepalived`,
  `openvswitch`, `mariadb-clustercheck`, `cron` and `kolla-toolbox` (both part
  of the `common` role).
* Defaults that stay off and need nothing: `enable_central_logging` /
  `enable_opensearch`, `enable_letsencrypt`, `enable_redis`/`enable_etcd`,
  `enable_prometheus` (the Prometheus exporters have `enable_prometheus_*`
  defaults that follow mariadb/memcached/haproxy, but their play is
  intersected with the `enable_prometheus_True` group, so nothing is deployed
  while Prometheus is off), `enable_cinder`, `enable_masakari`,
  `enable_octavia`, `enable_ironic`, `enable_swift`.

## Registry visibility

New GHCR packages are **private by default**. Either authenticate the nodes
(`docker login ghcr.io`) on the cluster, or make the packages public.

The build workflow tries to do that automatically (`public_packages`, using
`PATCH /user/packages/container/<name>` with the workflow's `GITHUB_TOKEN`). If
GitHub rejects that token for package administration, the step logs a warning
and the build still succeeds. In that case do it by hand — a user-owned package
needs `write:packages` on a personal token, or the UI:

`https://github.com/users/preinpost/packages/container/<image-name>/settings`
→ **Danger Zone** → **Change visibility** → **Public**.

`gh auth refresh -s write:packages` adds the scope, after which the whole
namespace can be flipped from a shell:

```bash
for pkg in $(gh api '/user/packages?package_type=container&per_page=100' --jq '.[].name'); do
  gh api -X PATCH "/user/packages/container/$(python3 -c 'import sys,urllib.parse;print(urllib.parse.quote(sys.argv[1],safe=""))' "$pkg")" \
    -f visibility=public
done
```

Note that GHCR visibility is per package, so every image needs it. Once a
package is public, anonymous pulls work:

```bash
crane manifest ghcr.io/preinpost/kolla/keystone:2024.1-rocky-9
```

All images in the three published sets below were verified this way (anonymous
token against the registry HTTP API, no credentials involved).

## Published image sets

| release | run | wall clock | images |
| --- | --- | --- | --- |
| zed | [42768163458](https://github.com/preinpost/kolla/actions/runs/42768163458) | 13m52s | 38 |
| 2024.1 | [42781172784](https://github.com/preinpost/kolla/actions/runs/42781172784) | 14m18s | 39 |
| 2024.2 | [42783107664](https://github.com/preinpost/kolla/actions/runs/42783107664) | 13m29s | 39 |

Sizes are `docker image ls` values from those runs (`—` means the image is not
part of that set; only 2024.1/2024.2 carry `proxysql`).

| image | zed | 2024.1 | 2024.2 |
| --- | --- | --- | --- |
| `base` | 655MB | 648MB | 648MB |
| `openstack-base` | 1.22GB | 1.2GB | 1.19GB |
| `kolla-toolbox` | 1.16GB | 1.22GB | 1.2GB |
| `cron` | 668MB | 661MB | 661MB |
| `fluentd` | 909MB | 905MB | 905MB |
| `haproxy` | 684MB | 677MB | 677MB |
| `keepalived` | 705MB | 697MB | 697MB |
| `memcached` | 697MB | 689MB | 689MB |
| `rabbitmq` | 733MB | 720MB | 720MB |
| `mariadb-base` | 779MB | 786MB | 786MB |
| `mariadb-server` | 1.08GB | 1.1GB | 1.1GB |
| `mariadb-clustercheck` | 779MB | 786MB | 786MB |
| `proxysql` | — | 789MB | 790MB |
| `openvswitch-base` | 754MB | 751MB | 751MB |
| `openvswitch-db-server` | 754MB | 751MB | 751MB |
| `openvswitch-vswitchd` | 754MB | 751MB | 751MB |
| `keystone-base` | 1.29GB | 1.28GB | 1.26GB |
| `keystone` | 1.31GB | 1.32GB | 1.31GB |
| `keystone-fernet` | 1.31GB | 1.3GB | 1.28GB |
| `keystone-ssh` | 1.31GB | 1.3GB | 1.28GB |
| `glance-base` | 1.37GB | 1.32GB | 1.31GB |
| `glance-api` | 1.4GB | 1.36GB | 1.34GB |
| `placement-base` | 1.23GB | 1.21GB | 1.2GB |
| `placement-api` | 1.23GB | 1.21GB | 1.2GB |
| `horizon` | 1.45GB | 1.44GB | 1.42GB |
| `nova-base` | 1.52GB | 1.51GB | 1.5GB |
| `nova-api` | 1.52GB | 1.51GB | 1.5GB |
| `nova-conductor` | 1.52GB | 1.51GB | 1.5GB |
| `nova-scheduler` | 1.52GB | 1.51GB | 1.5GB |
| `nova-novncproxy` | 1.54GB | 1.64GB | 1.63GB |
| `nova-compute` | 2.12GB | 2.14GB | 2.1GB |
| `nova-libvirt` | 1.51GB | 1.53GB | 1.5GB |
| `nova-ssh` | 1.54GB | 1.53GB | 1.52GB |
| `neutron-base` | 1.46GB | 1.46GB | 1.44GB |
| `neutron-server` | 1.46GB | 1.47GB | 1.45GB |
| `neutron-openvswitch-agent` | 1.46GB | 1.46GB | 1.44GB |
| `neutron-l3-agent` | 1.49GB | 1.49GB | 1.47GB |
| `neutron-dhcp-agent` | 1.46GB | 1.46GB | 1.44GB |
| `neutron-metadata-agent` | 1.46GB | 1.46GB | 1.44GB |

Every image is `linux/amd64` only; kolla-build defaults to `amd64` and the
workflow does not override `--arch`.

Verify from a clean shell (no credentials needed once the packages are public):

```bash
# any of the images, for any of the tags
tag=2024.1-rocky-9
image=keystone
curl -s "https://ghcr.io/token?scope=repository:preinpost/kolla/${image}:pull&service=ghcr.io" \
  | python3 -c 'import sys,json;print(json.load(sys.stdin)["token"])' > /tmp/t
curl -s -o /dev/null -w '%{http_code}\n' -H "Authorization: Bearer $(cat /tmp/t)" \
  -H 'Accept: application/vnd.docker.distribution.manifest.v2+json' \
  "https://ghcr.io/v2/preinpost/kolla/${image}/manifests/${tag}"
```

## Release notes / gotchas

**Common to every release**

No template overrides beyond `.github/kolla/template-override.j2` were needed.
That file carries two blocks:

* `labels` — adds `org.opencontainers.image.*` to every image so that GHCR
  links each package back to this repository.
* `openstack_base_footer` — pins `pip==24.0` inside the OpenStack virtualenv.
  pip 26 (and later) build sdists that only ship a `setup.py` (every
  `XStatic-*` package, which horizon needs) through PEP 517 build isolation,
  where present-day setuptools no longer provides `pkg_resources`, so the
  install fails with `ModuleNotFoundError: No module named 'pkg_resources'`.
  pip 24.0 keeps using the legacy `setup.py` path together with the
  `setuptools==67.2.*` that openstack-base already pins, which is how the
  releases were built originally.

That pin is still needed for 2024.1 and 2024.2 and is not harmful: those two
horizon templates force `pip==25.2.*` for themselves, overriding the base pin,
and pip 25.2 still takes the legacy `setup.py` path, so the sdist-only XStatic
packages build. Verified inside the pushed images — `keystone:2024.1-rocky-9`
has `pip 24.0` in `/var/lib/kolla/venv`, `horizon:2024.1-rocky-9` has
`pip 25.2`. Raising the base pin past 25.2 would break those images again.

**Zed (2022.2), rocky 9** — the base image installs
`centos-release-openstack-zed`. On Rocky Linux 9 that package now comes from
Rocky's own `extras` repository (it is still published there, which is what
makes this rebuild possible), and the `CentOS-OpenStack-zed` repository it
configures is still served from `mirror.stream.centos.org`. The source tarballs
are taken from
`https://tarballs.opendev.org/openstack/<project>/<project>-stable-zed.tar.gz`
and the pins come from `requirements-stable-zed.tar.gz`, both of which are still
available, so builds are reproducible.

**2024.1 (Caracal) / 2024.2 (Dalmatian), rocky 9** — same picture:

* `centos-release-openstack-caracal` and `centos-release-openstack-dalmatian`
  are still in Rocky 9 `extras`; they pull `centos-release-cloud`,
  `centos-release-nfv-openvswitch`, `centos-release-ceph-reef` and
  `centos-release-rabbitmq-38` along, so the OVS and repo files the other
  templates enable exist.
* The `CentOS-OpenStack-caracal` / `-dalmatian` repositories are served from
  `https://mirror.stream.centos.org/SIGs/9-stream/cloud/x86_64/openstack-<release>/`
  (the `SIGs/` part is required; the `9-stream/cloud/x86_64/` root that the repo
  name suggests returns 404).
* Source tarballs and `upper-constraints.txt` come from
  `stable/2024.1` and `stable/2024.2` on `tarballs.opendev.org`, both alive, so
  the images match what the release shipped.
* 2024.2 dropped `centos-release-nfv-openvswitch` from the base image's
  `centos_yum_repo_packages`; the dependency of
  `centos-release-openstack-dalmatian` still installs it, so
  `openvswitch-base` works unmodified.

Build-side workaround that 2024.1/2024.2 needed — the workflow now installs the
docker SDK itself when the checked-out kolla does not:

* 2024.1 moved the docker SDK from `requirements.txt` to
  `test-requirements.txt`, so `pip install ./kolla-src` leaves kolla-build
  unable to reach the daemon (`Error, you have set Docker as container engine,
  but the Python library is not found`).
* Simply installing the latest docker-py is not enough: docker-py 6.x with
  `requests >= 2.32` fails with
  `Error while fetching server API version: Not supported URL scheme http+docker`
  (2024.1 also dropped `requests<2.32.0` from its requirements).

The reusable workflow therefore installs `docker>=3.0.0,<7.0.0` together with
`requests<2.32.0` when `import docker` fails, which is exactly the pair zed
declares and was built with; refs that already declare the SDK (zed) are
untouched.

**2024.2 and proxysql** — `enable_proxysql` defaults to `yes` in 2024.2 (it is
`no` in 2024.1 and zed), and with it `enable_loadbalancer` and
`mariadb_loadbalancer` switch to proxysql. The image is therefore part of both
new sets, so a default 2024.2 deployment finds it; 2024.1 deployments keep
using haproxy unless they opt in.

## Extending to another release

1. Add an entry to `.github/kolla/groups.json` keyed by the release name, with
   the same phase 0/1 split and the image names that release uses (kolla renamed
   several images over time — check `git show <tag>:docker` in kolla).
2. Run the workflow with the matching `release`, `kolla_ref`, `base_distro`,
   `base_distro_version` and `image_tag`.
3. If a repository used by the release has been retired, either point
   `--repos-yaml` at a modified copy of `kolla/template/repos.yaml` or add a
   `--template-override` that rewrites the offending `dnf`/`apt` block.
