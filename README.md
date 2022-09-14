RHEL for Edge Image Builder demo
====

[Red Hat Enterprise Linux (RHEL) for Edge](https://www.redhat.com/en/technologies/linux-platforms/enterprise-linux/edge-computing) puts a consistent layer on inconsistent edge environments, enabling you to confidently scale edge workloads with reliable updates and intelligent rollbacks. Like [RHCOS](https://docs.openshift.com/container-platform/4.11/architecture/architecture-rhcos.html), it is based on [ostree](https://github.com/ostreedev/ostree) which provides easy (and small in size) updates, rollbacks and background staging of upgrades.

Using Image Builder, IT teams can quickly create, deploy, and easily maintain custom edge-optimized OS images over the life of the system. Image Builder is provided by Red Hat Enterprise Linux and contains everything needed to run edge workloads in specific places. These images can be customized for unique edge workloads, helping keep edge deployments consistent, scalable, more secure, and compliant.

In this demo, we will see how we can customize the installation (and installed) image in order to provide wifi functionality to the deployed system.

Requirements (host preparation)
---

We will need a development host, where image builder will run:
  * [Red Hat Enterprise 9](https://developers.redhat.com/products/rhel/download)
  * at least two core CPU
  * 4GB memory
  * 20GB available space on `/var` mountpoint

This can easily be a dedicated virtual machine.

On the host (we'll call it `image-builder` from now on), you must have the following packages installed:

```
$ dnf install -y osbuild-composer composer-cli
```

Also, `obuild-composer` service must be started and enabled:

```
$ systemctl enable --now osbuild-composer.socket
```

Blueprint creation
---

Custom images are created from a _blueprint_ in TOML format which specifies customization parameters. For our demo, we are including wifi wireless support through extra packages and a administrator user:

```
name = "wifi-container"
description = "Wireless enabled RHCOS and user customization"
version = "0.0.1"
modules = []
groups = []
distro = ""

[[packages]]
name = "NetworkManager-wifi"
version = "*"

[[customizations.user]]
name = "kwozyman"
description = "User Kwozy"
password = "$6$ZRlM4KS1CfCRIC09$S/FhlIj3zD6mFsFX.IlCdZfVTuU2AstJxCAV4wTCyuhjGaBlNw.i7BHRUQ1Woh2P0uRnHPfajgHSvYuNPy01F0"
key = "ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAIMPkccS+SKCZEWGJzH7ew0eNPItvqeGFpOhZprmL9owO fortress_of_solitude"
home = "/home/kwozyman/"
shell = "/usr/bin/bash"
groups = ["users", "wheel"]
```

After creating the above file (`wifi-enabled-blueprint.toml`), we can push it to the Image Builder:

```
$ composer-cli blueprints push blueprint-wifi-container.toml
```

...and list the blueprints to confirm:

```
$ composer-cli blueprints list
wifi-enabled
```

In order to pull the NetworkManager component for configuring wireless NICs, we are including package `NetworkManager-wifi`, with it's latest (`*`) version. This, of course, pulls a lot of dependencies with all the kernel drivers and firmware packages needed. To see the complete list, you can use the `depsolve` command:

```
$ composer-cli blueprints depsolve wifi-container
```

Container for Edge creation
---

In order for the installer image to be built, we need a http server with the base OS layer. This sounds complicated, but Red Hat offers an easy container with a web server and the desired files:

```
$ composer-cli compose start wifi-container edge-container
```

The above command will take some time and we can follow the progress with `composer-cli compose status`. After it's finished, we can download the container image by using the id listed in `status`:

```
$ composer-cli compose image <uid>
```

...then load and start the image in `podman`:

```
$ podman load -i <tar-file>
$ podman tag <id-returned-by-podman-load> localhost/edge-container
$ podman run --rm --detach --name edge-container --publish 8080:8080 localhost/edge-container
```

Edge installer image creation
---

We will need a new blueprint in order to generate the installation iso:

```
name = "wifi-iso"
description = "Wireless enabled RHCOS and user customization"
version = "0.0.1"
modules = []
groups = []
distro = ""
```

Please take note as to the lack of customizations. Including customizations here would overlap with the first blueprint and installer generation would not be possible.

As with the first blueprint, we need to `composer-cli blueprint push blueprint-wifi-iso.toml`.

With the Edge container running, we can now generate the actual installer `.iso` for our customized RHCOS image:

```
$ composer-cli compose start-ostree --ref rhel/9/x86_64/edge --url http://localhost:8080/repo wifi-iso edge-installer
```

Please note the above `--ref` which is served by the edge-container we've started earlier. Like for the container, we can check composition status with `composer-cli compose status` and when it's finished, we can download the .iso file:

```
$ composer-cli compose image <uid>
```
