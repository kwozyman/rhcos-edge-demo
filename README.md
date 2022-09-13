RHCOS for Edge Image Builder demo
====

Red Hat Enterprise Linux CoreOS (RHCOS) represents the next generation of single-purpose container operating system technology by providing the quality standards of Red Hat Enterprise Linux (RHEL) with automated, remote upgrade features. In other words, RHCOS is a immutable operating system from Red Hat, developed as an Openshift component.

Next, we will see how you customize the installation (and installed) image.

Requirements (host preparation)
---

We will need a development host, where image builder will run:
  * [Red Hat Enterprise 9](https://developers.redhat.com/products/rhel/download)
  * at least two core CPU
  * 4GB memory
  * 20GB available space on `/var` mountpoint

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

Custom images are created from a _blueprint_ in TOML format which specifies customization parameters. For our demo, we are including wifi wireless support through extra packages:

```
name = "wifi-enabled"
description = "Wireless enabled RHCOS"
version = "0.0.1"
modules = []
groups = []
distro = ""

[[packages]]
name = "NetworkManager-wifi"
version = "*"
```

After creating the above file (`wifi-enabled-blueprint.toml`), we can push it to the Image Builder:

```
$ composer-cli blueprints push wifi-enabled-blueprint.toml
```

...and list the blueprints to confirm:

```
$ composer-cli blueprints list
wifi-enabled
```

In order to pull the NetworkManager component for configuring wireless NICs, we are including package `NetworkManager-wifi`, with it's latest (`*`) version. This, of course, pulls a lot of dependencies with all the kernel drivers and firmware packages needed. To see the complete list, you can use the `depsolve` command:

```
$ composer-cli blueprints depsolve wifi-enabled
```

Container for Edge creation
---

In order for the installer image to be built, we need a http server with the base OS layer. This sounds complicated, but Red Hat offers an easy container with a web server and the desired files:

```
$ composer-cli compose start wifi-enabled edge-container
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

With the Edge container running, we can now generate the actual installer `.iso` for our customized RHCOS image:

```
$ composer-cli compose start-ostree --ref rhel/9/x86_64/edge --url http://localhost:8080/repo wifi-enabled edge-installer
```

Please note the above `--ref` which is served by the edge-container we've started earlier. Like for the container, we can check composition status with `composer-cli compose status` and when it's finished, we can download the .iso file:

```
$ composer-cli compose image <uid>
```
