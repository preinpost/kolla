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

## Running it

From the Actions tab: **eol-images** → **Run workflow**, or from the CLI:

```bash
gh workflow run eol-images.yml \
  --repo preinpost/kolla \
  -f release=zed \
  -f kolla_ref=zed-eol \
  -f base_distro=rocky \
  -f base_distro_version=9 \
  -f image_tag=zed-rocky-9 \
  -f groups=all \
  -f push_images=true \
  -f registry=ghcr.io \
  -f namespace=preinpost/kolla
```

`groups` accepts `all`, `base`, or a comma separated subset of the group names
(`base`, `infra`, `keystone-glance-placement-horizon`, `nova`, `neutron`). Use
`groups=base` to validate the platform before spending time on the services.

Inputs:

| input | meaning |
| --- | --- |
| `release` | kolla-build `--openstack-release` (drives the source tarball branch, e.g. `stable-zed`) |
| `kolla_ref` | git ref of `preinpost/kolla` to build from; use the release's last tag or `zed-eol` |
| `base_distro` / `base_distro_version` | kolla-build `--base` / `--base-tag` |
| `image_tag` | tag applied to every image; must match what kolla-ansible computes as `openstack_tag` |
| `registry` / `namespace` | push destination, e.g. `ghcr.io` + `preinpost/kolla` |
| `push_images` | set to `false` to build without pushing, useful for debugging |
| `public_packages` | try to flip the pushed GHCR packages to public after pushing |

Each job logs in with the workflow's `GITHUB_TOKEN`
(`permissions: packages: write`), so no extra secret is needed for pushing.

## Image naming and kolla-ansible wiring

Images are pushed as `<registry>/<namespace>/<service>:<image_tag>`, for example
`ghcr.io/preinpost/kolla/keystone:zed-rocky-9`. Because the tag is chosen to be
exactly what kolla-ansible computes for the release, a deployment only has to
override the registry and namespace:

```yaml
# /etc/kolla/globals.yml
docker_registry: "ghcr.io"
docker_namespace: "preinpost/kolla"
```

For zed / rocky 9 those values give `openstack_tag=zed-rocky-9`
(`{{ openstack_release }}-{{ kolla_base_distro }}-{{ kolla_base_distro_version }}`),
`docker_namespace` defaults to `openstack.kolla` and `kolla_base_distro` to
`rocky`, so nothing else needs to change. Verify with:

```bash
grep -E '^(openstack_tag|kolla_base_distro|docker_registry|docker_namespace)' \
  /usr/share/kolla-ansible/ansible/group_vars/all.yml
ansible -i /etc/kolla/all-in-one -m debug -a "var=openstack_tag" all
```

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
crane manifest ghcr.io/preinpost/kolla/keystone:zed-rocky-9
```

## Published image set

The zed / rocky 9 set was built from `kolla_ref=zed-eol`, `image_tag=zed-rocky-9`
([run 42768163458](https://github.com/preinpost/kolla/actions/runs/42768163458),
about 18 minutes wall clock, 38 images, ~42 GB on disk in total):

| | |
| --- | --- |
| parents | `base` 655MB, `openstack-base` 1.22GB |
| infra | `kolla-toolbox` 1.16GB, `cron` 668MB, `fluentd` 909MB, `haproxy` 684MB, `keepalived` 705MB, `memcached` 697MB, `rabbitmq` 733MB |
| database | `mariadb-base` 779MB, `mariadb-server` 1.08GB, `mariadb-clustercheck` 779MB |
| network | `openvswitch-base` / `-db-server` / `-vswitchd` 754MB each |
| identity | `keystone-base` 1.29GB, `keystone` / `-fernet` / `-ssh` 1.31GB each |
| image | `glance-base` 1.37GB, `glance-api` 1.4GB |
| placement | `placement-base` / `-api` 1.23GB |
| dashboard | `horizon` 1.45GB |
| compute | `nova-base` / `-api` / `-conductor` / `-scheduler` 1.52GB, `nova-novncproxy` / `-ssh` 1.54GB, `nova-libvirt` 1.51GB, `nova-compute` 2.12GB |
| networking | `neutron-base` / `-server` / `-openvswitch-agent` / `-dhcp-agent` / `-metadata-agent` 1.46GB, `neutron-l3-agent` 1.49GB |

That covers a core deployment with keystone, glance, placement, nova,
neutron+openvswitch and horizon. Images for services the deployment does not
enable (cinder, heat, ceilometer, barbican, octavia, ironic, swift, ...) are not
built; add them to a group in `groups.json` when needed. `proxysql` is skipped
because kolla-ansible 15.6.0 defaults to `enable_proxysql: no` (haproxy + the
mariadb images are used instead).

Verify from a clean shell (no credentials needed once the packages are public):

```bash
# any of the 38 images
curl -s "https://ghcr.io/token?scope=repository:preinpost/kolla/keystone:pull&service=ghcr.io" \
  | python3 -c 'import sys,json;print(json.load(sys.stdin)["token"])' > /tmp/t
curl -s -o /dev/null -w '%{http_code}\n' -H "Authorization: Bearer $(cat /tmp/t)" \
  -H 'Accept: application/vnd.docker.distribution.manifest.v2+json' \
  https://ghcr.io/v2/preinpost/kolla/keystone/manifests/zed-rocky-9
```

## Release notes / gotchas

**Zed (2022.2), rocky 9** — the base image installs
`centos-release-openstack-zed`. On Rocky Linux 9 that package now comes from
Rocky's own `extras` repository (it is still published there, which is what
makes this rebuild possible), and the `CentOS-OpenStack-zed` repo it configures
is still served from `mirror.stream.centos.org`. The source tarballs are taken
from `https://tarballs.opendev.org/openstack/<project>/<project>-stable-zed.tar.gz`
and the pins come from `requirements-stable-zed.tar.gz`, both of which are still
available, so builds are reproducible.

No template overrides beyond `.github/kolla/template-override.j2` were needed
for zed; that file carries two blocks:

* `labels` — adds `org.opencontainers.image.*` to every image so that GHCR
  links each package back to this repository.
* `openstack_base_footer` — pins `pip==24.0` inside the OpenStack virtualenv.
  pip >= 25.1 builds sdists that only ship a `setup.py` (every `XStatic-*`
  package, which horizon needs) in a PEP 517 isolated environment, where
  present-day setuptools no longer provides `pkg_resources`, so horizon fails
  with `ModuleNotFoundError: No module named 'pkg_resources'`. pip 24.0 keeps
  using the legacy `setup.py` path together with the `setuptools==67.2.*` that
  openstack-base already pins, which is how the release was built originally.

Everything else (repositories, constraints, source tarballs) still works
unmodified, because Rocky's `extras` keeps publishing
`centos-release-openstack-zed`, `mirror.stream.centos.org` still serves the
`CentOS-OpenStack-zed` repository, and `tarballs.opendev.org` still serves the
`stable-zed` tarballs that carry the release's `upper-constraints.txt`.

## Extending to another release

1. Add an entry to `.github/kolla/groups.json` keyed by the release name, with
   the same phase 0/1 split and the image names that release uses (kolla renamed
   several images over time — check `git show <tag>:docker` in kolla).
2. Run the workflow with the matching `release`, `kolla_ref`, `base_distro`,
   `base_distro_version` and `image_tag`.
3. If a repository used by the release has been retired, either point
   `--repos-yaml` at a modified copy of `kolla/template/repos.yaml` or add a
   `--template-override` that rewrites the offending `dnf`/`apt` block.
