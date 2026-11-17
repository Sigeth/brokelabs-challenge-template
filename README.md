# BrokeLabs challenge template

Start here to write a challenge for BrokeLabs. A challenge is a directory with a `challenge.yaml`: either a repository of its own (this template), or one directory of a repository holding several challenges, like the [official challenges](https://github.com/Sigeth/brokelabs-challenges). The platform installs it from a catalog, runs it in an isolated environment for each student, and checks the tasks automatically.

## Layout

```
challenge.yaml        manifest: machines, access, tasks and their checks
images/<machine>/     Dockerfile of each machine image (minimal, unprivileged user)
instructions/en.md    what the student reads (one file per language)
setup/                scripts run once when an instance starts (break things here)
solution/             reference solution, for testing and for teachers (never shown to students)
```

The first line of `challenge.yaml` links the JSON Schema: editors using the YAML language server (VS Code, Neovim, JetBrains…) then complete and validate every field.

## Runtimes

Pick the lightest runtime that can host the challenge. The cluster admin sees what it needs before installing.

| Runtime | What the student gets | Use it for |
|---|---|---|
| `pod` | A container, no privileges | Files, permissions, config, app debugging |
| `sysbox` | A system container with systemd, Docker or Podman inside | Services, container engines |
| `vcluster` | Admin of a virtual Kubernetes cluster | RBAC, NetworkPolicy, Pod Security, workloads |
| `vm` | A full virtual machine (KubeVirt) | Kernel, firewall, sshd, auditd, container escapes, kubeadm |

Only `pod` runs today; the other runtimes come with later platform versions.

**`pod` machines** run as UID/GID 1000 without any privilege (Pod Security "restricted"): no root, no `sudo`, no package installs. `setup/` scripts and checks run as that user too. The container gets a TTY, so an image whose default command is a shell stays up; services can be started in the background by a setup script.

## Machine images

Each machine image is built from `images/<machine>/Dockerfile` and published with the challenge. Students see how it is built, so make it an example of good practice:

- **Start from a [brokelabs-images](https://github.com/Sigeth/brokelabs-images) container base:** `ghcr.io/sigeth/brokelabs-images/alpine-3.24` (busybox shell) or `debian-13-slim` (bash). It brings the unprivileged `student` user and group (UID/GID 1000, a home directory, no extra groups) and runs as `USER 1000:1000`. Switch to `USER root` only to install packages, end with `drop-package-manager` (nothing is installed at run time), and come back to `USER 1000:1000`. Never rely on `root`, or on a distribution's default user that belongs to `sudo` or `adm`.
- **Only the tools the challenge needs.** Add packages one by one. No `sudo`, `systemd`/`systemctl`, compilers or editors the tasks do not use: every extra binary is attack surface, and a distraction.
- **Pin the base image by digest**, and tag the machine image with the challenge version.

`brokelabs challenge images .` lists the images and the ones the challenge builds. The workspace dev loop builds and loads them into the dev cluster.

### Virtual machines (`runtime: vm`)

A `vm` machine boots a KubeVirt containerDisk. Prefer a [brokelabs-images](https://github.com/Sigeth/brokelabs-images) base **as is** (e.g. `ghcr.io/sigeth/brokelabs-images/debian-13:<version>`) and break or prepare it in `setup` scripts: they run as root over SSH when the instance starts. Every challenge on the same base then shares one cached disk per node. The platform creates the users at boot: `student` (UID 1000, passwordless `sudo`) and `brokelabs-checker` (the checker's account: leave its `sudo` rule alone).

Instances usually have no internet access, so a setup script cannot install packages. When the machine needs some, build your own image from a base with a recipe, `images/<machine>/image.yaml`:

```yaml
version: 0.1.0                                       # tag it like the challenge
source:
  image: ghcr.io/sigeth/brokelabs-images/debian-13:1.0.1
run: [install.sh]                                    # run as root in the disk, with network access
```

[brokelabs-ci](https://github.com/Sigeth/brokelabs-ci) builds it (`dagger call vm-image --source=images/<machine>`). Each such image is a full copy of the disk, so keep it for when setup scripts are not enough. Any image meeting the base image contract works; check one with `brokelabs image test IMAGE`.

## Checks

Each task has one check, evaluated **from outside** the student's machines:

| Check | Passes when |
|---|---|
| `shell` | a command exits with `expectExit` (default 0) and its output matches `expectOutput` |
| `file` | a file exists (or not) with the expected `mode`, `owner` and content (`contains` / `notContains`) |
| `sshLogin` | logging in as `user` is `allowed` or `denied` (vm). The checker presents the student's key (or `password`), so for another user, install the student's public key for that user in your setup (`cp /home/student/.ssh/authorized_keys /root/.ssh/`). `denied` needs a running SSH server that refuses the login. |
| `port` | a port is `open` or `closed` |
| `http` | a request returns `expectStatus` and a body matching `expectBody` |
| `kubeCEL` | a CEL expression holds for `all`, `any` or `none` of the selected objects (vcluster) |
| `goss` | a [goss](https://github.com/goss-org/goss) spec passes |

On `vm` machines, `shell` and `file` checks run over SSH, as the check's `user` (default `root`) through `sudo`. The student is `student` (UID 1000) with passwordless `sudo`; setup scripts run as root.

Good checks:
- **Test behavior, not files.** "SSH as root is denied" beats "sshd_config contains PermitRootLogin no", because a student with root can fake a file but not a behavior.
- **Fail closed.** If the student breaks the checker's access, the task fails. It never passes.
- **One task, one idea**, with a title that says what "solved" means.
- **Guard what must keep working.** A task true from the start that the student must not break ("you can still log in with your key") is a guard: `guard: true` and `points: 0`.

Need logic beyond these? Set `spec.validation.checkerImage` to a custom checker image. The [brokelabs-checker SDK](https://github.com/Sigeth/brokelabs-checker) makes that a single Go function.

## Workflow

```sh
brokelabs challenge lint .      # validate (same rules as the platform)
```

Try it on a local platform (the workspace dev cluster):

```sh
brokelabs challenge manifest . | kubectl apply -f -   # install it, unsigned (development only)
```

Publish it to a registry (push the machine images first):

```sh
brokelabs challenge build .     # writes dist/<name>-<version>, an OCI image layout
brokelabs challenge push dist/my-challenge-0.1.0 ghcr.io/your-account/my-challenge
```

`build` pins every image by digest and fails if a tag cannot be resolved, so the package always starts the images it was built with. `solution/` is published next to the package as a separate artifact (an OCI referrer), never inside it. Registry credentials come from `docker login` or `podman login`.

Test it on a platform with the solution (the workspace dev cluster, or any cluster with BrokeLabs):

```sh
brokelabs challenge test .      # every task fails, the solution runs, every task passes
```

The solution is `solution/<machine>.sh` for each machine that has one, or `solution/solve.sh` on the first terminal machine. It runs with `sh` as the student, as if typed in the terminal (the student has `sudo` on `vm` machines). Guards must pass before and after it.

Sign what you publish, with a key (`brokelabs challenge sign --key`) or, in CI, keyless (`--keyless`), and list your packages in a catalog with `brokelabs catalog index`.

Rules for publishing:
- Bump `metadata.version` for every release.
- Keep `solution/` working.

## License

[MIT](LICENSE). Your challenge can use any license: set it in `spec.license`.
