# The Tailscale universal Docker mod

This Docker mod lets you slipstream Tailscale into [linuxserver.io](https://linuxserver.io) containers. This lets you have applications join your tailnet.

This differs from the sidecar approach of `tailscale/tailscale` ([DockerHub](https://hub.docker.com/r/tailscale/tailscale), [blog explination](https://tailscale.com/blog/docker-tailscale-guide)), which takes 2 (or more) Docker containers and then collapses the networking namespace into one. This is fragile in that restarting 1 of the multiple containers leaves the other disconnected from the networking namespace.

This Docker mod approach directly installs Tailscale into your Docker container. Making the Docker container its own separate machine on the tailnet. The disadvantage being that the Docker image must support Docker mods (which [linuxserver.io](https://linuxserver.io) containers do, but you can build your own).

## Fork

This is a fork of [tailscale-dev/docker-mod](https://github.com/tailscale-dev/docker-mod).

### Changes

* removed the installation of `iptables`, this forces `tailscaled --tun=userspace-networking` which it should have been using anyhow, but if `iptables` was installed, it didn't actually use.
* added the ENV `TAILSCALE_FURTHER_FLAGS` which is a more elegant way of adding `tailscale up <flags>`.
* added the ENV `TAILSCALE_STATEFUL_FILTERING` which sets `tailscale up --stateful-filtering`. Addresses the issue: [No longer working #24](https://github.com/tailscale-dev/docker-mod/issues/24)
* added `wget` fallback option in response to issue [Mod fails to initialize due to slow CURL #27](https://github.com/tailscale-dev/docker-mod/issues/27)
* merged edits from [chukysoria/docker-tailscale-mod](https://github.com/chukysoria/docker-tailscale-mod) and [pandalanax/docker-mod](https://github.com/pandalanax/docker-mod)
* fixed a typo in `TAILSCALE_BE_EXIT_NODE`, issue: [Incorrect Exit node environment variables #6](https://github.com/tailscale-dev/docker-mod/issues/6) (**but have not tested it**)
* updated README

## Configuration

### Environment Variables

The Docker mod exposes a bunch of environment variables that you can use to configure it.

* `(undocumented)` means this was in the original repository code, but not in the README file. It is untested by me.
* `(new)` means I added it to the forked code.

| Environment Variable (`tailscaled` & Docker mod) | Description | Example |
| :--------------------------------------------- | :---------- | :------ |
| `DOCKER_MODS` | The list of additional mods to layer on top of the running container, separated by pipes. | `ghcr.io/tailscale-dev/docker-mod:main` or `zorbatherainy/tailscale-docker-mod:latest` |
| `TAILSCALE_STATE_DIR` | The directory where the Tailscale state will be stored, this should be pointed to a Docker volume. If it is not, then the node will set itself as ephemeral, making the node disappear from your tailnet when the container exits. | `/var/lib/tailscale` |
| `TAILSCALE_TAILSCALED_LOG` (undocumented) | If enabled the tailscale log will be dumped to `/dev/null` (i.e., deleted) | `1` |
<br>
| Environment Variable (`tailscale up`) | Description | Example |
| :---------------------------------- | :---------- | :------ |
| `TAILSCALE_AUTHKEY` | The authkey for your tailnet. You can create one in the [admin panel](https://login.tailscale.com/admin/settings/keys). See [here](https://tailscale.com/kb/1085/auth-keys/) for more information about authkeys and what you can do with them. | `tskey-auth-hunter2CNTRL-hunter2hunter2` |
| `TAILSCALE_HOSTNAME` | The hostname that you want to set for the container. If you don't set this, the hostname of the node on your tailnet will be a bunch of random hexadecimal numbers, which many humans find hard to remember. | `wiki` |
| `TAILSCALE_USE_SSH` | Set this to `1` to enable SSH access to the container. | `1` |
| `TAILSCALE_BE_EXIT_NODE` (undocumented) | Set this to `1` to allow the container toact as an exit node. You may need to approve this in the admin console. | `0` |
| `TAILSCALE_STATEFUL_FILTERING` (new; chukysoria/docker-mod) | Sets `tailscale up --stateful-filtering`. Addresses the issue: [No longer working #24](https://github.com/tailscale-dev/docker-mod/issues/24) | `0` |
| `TAILSCALE_LOGIN_SERVER` | Set this value if you are using a custom login/control server (Such as headscale) | `https://headscale.example.com` |
| `TAILSCALE_FURTHER_FLAGS` (new) | Any additional flags you wish to use with `tailscale up <flags>`. You could have always done this by overloading `TAILSCALE_HOSTNAME` or another used ENV. | `--accept-dns` |
<br>
| Environment Variable (`tailscale serve/funnel`) | Description | Example |
| :-------------------------------------------- | :---------- | :------ |
| `TAILSCALE_SERVE_PORT` | The port number that you want to expose on your tailnet. This will be the port of your DokuWiki, Transmission, or other container. | `80` |
| `TAILSCALE_SERVE_MODE` | The mode you want to run Tailscale serving in. This should be `https` in most cases, but there may be times when you need to enable `tls-terminated-tcp` to deal with some weird edge cases like HTTP long-poll connections. See [here](https://tailscale.com/kb/1242/tailscale-serve/) for more information. | `https` |
| `TAILSCALE_FUNNEL` | Set this to `true`, `1`, or `t` to enable [funnel](https://tailscale.com/kb/1243/funnel/). For more information about the accepted syntax, please read the [strconv.ParseBool documentation](https://pkg.go.dev/strconv#ParseBool) in the Go standard library. | `on` |

### State Directory

Something important to keep in mind is that you really should set up a separate volume for Tailscale state. Here is how to do that with the docker commandline:

``` sh
docker volume create dokuwiki-tailscale
```

Then you can mount it into a container by using the volume name instead of a host path:

``` bash
docker run \
  ... \
  -v dokuwiki-tailscale:/var/lib/tailscale \
  ...
```

When using a docker composed file, you will need to declare the volumes in the top level volumes section:

``` yaml
volumes:
  dokuwiki-tailscale:
```

Then use it in a container:

``` yaml
services:
  dokuwiki:
    volumes:
      - dokuwiki-tailscale:/var/lib/tailscale
```

If you don't do this, all Tailscale state will be lost when the container reboots. This can range from inconvenience to annoying because you can breach your Let's Encrypt rate limits with this.