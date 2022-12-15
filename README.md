RHEL for Edge Image Builder demo
====

[Red Hat Enterprise Linux (RHEL) for Edge](https://www.redhat.com/en/technologies/linux-platforms/enterprise-linux/edge-computing) puts a consistent layer on inconsistent edge environments, enabling you to confidently scale edge workloads with reliable updates and intelligent rollbacks. Like [RHCOS](https://docs.openshift.com/container-platform/4.11/architecture/architecture-rhcos.html), it is based on [ostree](https://github.com/ostreedev/ostree) which provides easy (and small in size) updates, rollbacks and background staging of upgrades.

Using [Image Builder](https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux/9/html/composing_a_customized_rhel_system_image/index), IT teams can quickly create, deploy, and easily maintain custom edge-optimized OS images over the life of the system. Image Builder is provided by Red Hat Enterprise Linux and contains everything needed to run edge workloads in specific places. These images can be customized for unique edge workloads, helping keep edge deployments consistent, scalable, more secure, and compliant.

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

Also, `osbuild-composer` service must be started and enabled:

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
version = "1.36.*"

[[customizations.user]]
name = "kwozyman"
description = "User Kwozy"
password = "$6$ZRlM4KS1CfCRIC09$S/FhlIj3zD6mFsFX.IlCdZfVTuU2AstJxCAV4wTCyuhjGaBlNw.i7BHRUQ1Woh2P0uRnHPfajgHSvYuNPy01F0"
key = "ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAIMPkccS+SKCZEWGJzH7ew0eNPItvqeGFpOhZprmL9owO fortress_of_solitude"
home = "/home/kwozyman/"
shell = "/usr/bin/bash"
groups = ["users", "wheel"]
```

If needed, we can modify the values of `password` (e.g. `openssl passwd -6 <password>`) and `key`, and optionally other fields of `[[customizations.user]]`. They will be used to login to machines booted from the resulting image. The full list of possible image customization can be found [here](https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux/9/html/composing_a_customized_rhel_system_image/creating-system-images-with-composer-command-line-interface_composing-a-customized-rhel-system-image#image-customizations_creating-system-images-with-composer-command-line-interface).

After creating the above file (`blueprint-wifi-container.toml`), we can push it to the Image Builder:

```
$ composer-cli blueprints push blueprint-wifi-container.toml
```

...and list the blueprints to confirm:

```
$ composer-cli blueprints list
wifi-container
```

In order to pull the NetworkManager component for configuring wireless NICs, we are including package `NetworkManager-wifi`, with it's 1.36 (`1.36.*`) version at the time of writing. This, of course, pulls a lot of dependencies with all the kernel drivers and firmware packages needed. To see the complete list, you can use the `depsolve` command:

```
$ composer-cli blueprints depsolve wifi-container
```

Container for Edge creation
---

In order for the installer image to be built, we need a http server with the base OS layer. This sounds complicated, but Red Hat offers an easy container with a web server and the desired files:

```
$ composer-cli compose start wifi-container edge-container
```

The above command will take some time and we can follow the progress with `composer-cli compose status`:

```
$ composer-cli compose status
3f3c2398-8de6-483f-a784-ff0b49d26747 RUNNING  Tue Nov 8 20:41:53 2022 wifi-container  0.0.1 edge-container
```

After it's finished, we can download the container image by using the id listed in the first column of `status` output:

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

We will need a new blueprint (e.g. `blueprint-wifi-iso.toml`) in order to generate the installation iso:

```
name = "wifi-iso"
description = "Wireless enabled RHCOS and user customization"
version = "0.0.1"
modules = []
groups = []
distro = ""
```

Please take note as to the lack of customizations. Including customizations here would overlap with the first blueprint and installer generation would not be possible.

As with the first blueprint, we need to `composer-cli blueprints push blueprint-wifi-iso.toml`.

With the Edge container running, we can now generate the actual installer `.iso` for our customized RHCOS image:

```
$ composer-cli compose start-ostree --ref rhel/9/x86_64/edge --url http://localhost:8080/repo wifi-iso edge-installer
```

Please note the above `--ref` which is served by the edge-container we've started earlier.

```
  +-----------------------------------------+
  |                                         v
+------------+     +----------------+     +-------------+
| OS Builder | --> | Edge Container | --> | Install ISO |
+------------+     +----------------+     +-------------+

Fig. 1 - Image generation dependencies
```

Like for the container, we can check composition status with `composer-cli compose status` and when it's finished, we can download the .iso file:

```
$ composer-cli compose image <uid>
```

Deployment
---

Now you can proceed with the installation by using the `.iso` file outside of the `image-builder` VM. Please note this is still using Anaconda (the Red Hat Linux installer) and it is not a direct-to-disk image. A requirement for booting this iso is EFI support. For example, in order to deploy in a virtual machine with libvirt (note the `--boot` parameter):

```
virt-install --boot uefi \
--name VM_NAME --memory 2048 --vcpus 2 \
--disk size=20,path=/path/to/diskfile.qcow2 --cdrom /path/to/UUID-installer.iso \
--os-variant rhel9.0
```

Install `virt-viewer` on the host machine to be automatically connected to the guest display:

```
dnf install -y virt-viewer
```

According to what customizations are included in the blueprint, you will be presented with the remainder of sections in the installer. In our example, because we've included custom packages already, there is no software selection option in the installer.

After installation is complete, you will be presented with a RHEL for Edge system based on the blueprint.

Updates/Upgrades
---

There are two important aspects to take into consideration regarding updates:

1. the deployed system uses a reference pointing to a local path. This is in order to keep the installation self-contained to the setup disk;
2. upgraded images must be rebuilt by the administrator

In order to add the proper repository on the deployed host (`ostree remote delete` and `ostree remote add` are the important commands):

```
$ sudo ostree remote list
rhel
$ sudo ostree remote show-url rhel
file:///run/install/repo/ostree/repo

$ sudo ostree remote delete rhel
$ sudo ostree remote add --no-gpg-verify --no-sign-verify rhel-edge http://192.168.122.228:8080/repo

$ sudo ostree remote show-url rhel-web
http://192.168.122.228:8080/repo
```

Please note that in our example above, `192.168.122.228` is the image builder ip address (or wherever the edge-container image is running). Of course, firewall must be open for port 8080 on `image-builder`.

As the ostree repository is unchanged since the iso generation, there will be no updates available:

```
$ sudo rpm-ostree upgrade --check
1 metadata, 0 content objects fetched; 153 B transferred in 0 seconds; 0 bytes content written
No updates available.
```

In order to update the edge container image, we can update the `blueprint-wifi-container.toml` file by changing the package version to the latest:

```
[[packages]]
name = "NetworkManager-wifi"
version = "*"
```

and the uploading it from the `image-builder` host:

```
$ composer-cli blueprints push blueprint-wifi-container.toml
$ composer-cli blueprints show wifi-container
name = "wifi-container"
description = "Wireless enabled RHCOS and user customization"
version = "0.0.2"
modules = []
groups = []
distro = ""

[[packages]]
name = "NetworkManager-wifi"
version = "*"

[customizations]

[[customizations.user]]
name = "kwozyman"
description = "User Kwozy"
password = "$6$ZRlM4KS1CfCRIC09$S/FhlIj3zD6mFsFX.IlCdZfVTuU2AstJxCAV4wTCyuhjGaBlNw.i7BHRUQ1Woh2P0uRnHPfajgHSvYuNPy01F0"
key = "ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAIMPkccS+SKCZEWGJzH7ew0eNPItvqeGFpOhZprmL9owO fortress_of_solitude"
home = "/home/kwozyman/"
shell = "/usr/bin/bash"
groups = ["users", "wheel"]
```

Please note the `version` field is automatically updated to the next version, even though we have not explicitely changed it in the pushed blueprint file. We can now regenerate the edge container, like we did before the iso build:

```
$ composer-cli compose start wifi-container edge-container
$ podman load -i <tar-file>
$ podman tag <id-returned-by-podman-load> localhost/edge-container
$ podman run --rm --detach --name edge-container --publish 8080:8080 localhost/edge-container
```

Please make sure the previous container is stopped before starting the new one.

Moving back to the edge host, we should be able to upgrade now:

```
$ rpm-ostree upgrade
```

Custom packages
---

[Managing repositories](https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux/9/html/composing_a_customized_rhel_system_image/managing-repositories_composing-a-customized-rhel-system-image) explains how to use custom system repositories.

If we wanted to add a package from a custom or 3rd party repository, we could do this as follows.

Let's say we want our image to include the Intel(R) distribution of OpenVINO:

```
[[packages]]
name = "openvino"
version = "*"
```

First, create a TOML file, e.g. `openvino-repo.toml`:

```
id = "OpenVINO"
name = "Intel(R) Distribution of OpenVINO"
type = "yum-baseurl"
url = "https://yum.repos.intel.com/openvino/2022"
gpgkey = "https://yum.repos.intel.com/intel-gpg-keys/GPG-PUB-KEY-INTEL-SW-PRODUCTS.PUB"
check_gpg = true
check_ssl = true
system = false
```

Then, add the repository to the image composer sources:

```
composer-cli sources add openvino-repo.toml
```

Troubleshooting
---

Useful commands for troubleshooting image composes:

* Show the log of a compose:

```
composer-cli compose log <id>
```

* View diagnostics messages produced by the composer:

```
journalctl -t osbuild-composer
```

* View diagnostics messages produced by the worker:

 ```
 journalctl -t osbuild-worker
 ```

Distributing the OS as container images
---

The whole ostree image can be [distributed as a container image file](https://coreos.github.io/rpm-ostree/container/) as well!

The following instructions take place in the `containers` directory of this repo:

```
$ cd containers
```

For convenience, we're using a different blueprint, but the previous blueprint can be easily used. We are simply adding the `subscription-manager` package in the new one in order to manage Red Hat subscriptions (to install packages).

```
$ composer-cli blueprint push blueprint.toml
```

Like we did earlier, we're starting a new image compose using `compose-cli`, this time using the `edge-commit` target:

```
$ composer-cli compose start-ostree r4e-cnt
$ composer-cli compose status
```

After a couple minutes, our commit is ready to download:

```
$ composer-cli compose status
...
de4a9fa0-b48e-40ae-8d3c-bf5b45434629 FINISHED Wed Dec 14 15:11:50 2022 r4e-cnt         0.0.2 edge-commit
...
$ composer-cli compose image de4a9fa0-b48e-40ae-8d3c-bf5b45434629
de4a9fa0-b48e-40ae-8d3c-bf5b45434629-commit.tar
```

Please note that the resulting file is a tar archive.

Next, we need to extract the ostree commit. Let's use a directory called `ostree-commit` for that (of course, you will need to use the downloaded archive from earlier):

```
$ mkdir ostree-commit
$ tar xvf de4a9fa0-b48e-40ae-8d3c-bf5b45434629-commit.tar -C ostree-commit
```

We now have all the data required to construct the container:

```
$ ostree container encapsulate --repo=ostree-commit/repo rhel/9/x86_64/edge oci-archive:ostree-container.tar
```

The result is `ostree-container.tar`, which can be easily imported to podman and then sent to a image registry service (I'm using my private quay registry as the example below):

```
$ podman load -i ostree-container.tar
(...)
Loaded image: sha256:e19b5e0bc79d9f34abbedbbf651ed5dc32453fd3550d38123b922a7058362521
$ podman tag sha256:e19b5e0bc79d9f34abbedbbf651ed5dc32453fd3550d38123b922a7058362521 quay.io/kwozyman/r4e
$ podman push quay.io/kwozyman/r4e
```

*On an already deployed ostree host*, it is possible to rebase to using the container image:

```
$ rpm-ostree rebase ostree-unverified-registry:quay.io/kwozyman/r4e --experimental
$ systemctl reboot
$ rpm-ostree status
State: idle
Deployments:
● ostree-unverified-registry:quay.io/kwozyman/r4e
                   Digest: sha256:5a2652635ca393956afbb47afb66d13ad328bcfe6afa6bde6f1da840cbc5ccf1
                Timestamp: 2022-12-15T13:21:36Z
```

After a rebase, all further rpm-ostree operations work as you’d expect. For example, rpm-ostree upgrade will look for a new container version.

One can even layer packages and files using the Dockerfile syntax:

```
$ cat Containerfile
FROM quay.io/kwozyman/r4e:latest
ADD blueprint.toml /etc/initial-blueprint.toml
$ podman build . -t quay.io/kwozyman/r4e:changed
$ podman push quay.io/kwozyman/r4e:changed
```

After the container image is in a available image registry service, we can rebase on the new container:

```
$ rpm-ostree rebase ostree-unverified-registry:quay.io/kwozyman/r4e:changed --experimental
Pulling manifest: ostree-unverified-image:docker://quay.io/kwozyman/r4e:changed
Importing: ostree-unverified-image:docker://quay.io/kwozyman/r4e:changed (digest: sha256:4f12831a27830e139e2709c5a52027492b8dfd5f8b17e311f7b9dc3815faa3e4)
ostree chunk layers stored: 1 needed: 0 (0 bytes)
custom layers stored: 1 needed: 1 (628 bytes)
Fetching layer sha256:0386f043c153 (628 bytes)
Fetched layer sha256:0386f043c153
Checking out tree d580831... done
Enabled rpm-md repositories: rhel-9-for-x86_64-baseos-rpms rhel-9-for-x86_64-appstream-rpms
Importing rpm-md... done
rpm-md repo 'rhel-9-for-x86_64-baseos-rpms' (cached); generated: 2022-12-05T16:22:43Z solvables: 3061
rpm-md repo 'rhel-9-for-x86_64-appstream-rpms' (cached); generated: 2022-12-14T09:02:08Z solvables: 10460
Resolving dependencies... done
Checking out packages... done
Running pre scripts... done
Running post scripts... done
Running posttrans scripts... done
Writing rpmdb... done
Writing OSTree commit... done
Staging deployment... done
Freed: 40.3 MB (pkgcache branches: 0)
Changes queued for next boot. Run "systemctl reboot" to start a reboot
$ systemctl reboot
```

After the system reboot, we can see our changes:

```
$ rpm-ostree status
State: idle
Deployments:
● ostree-unverified-registry:quay.io/kwozyman/r4e:changed
                   Digest: sha256:4f12831a27830e139e2709c5a52027492b8dfd5f8b17e311f7b9dc3815faa3e4
                Timestamp: 2022-12-15T14:17:22Z

  ostree-unverified-registry:quay.io/kwozyman/r4e
                   Digest: sha256:5a2652635ca393956afbb47afb66d13ad328bcfe6afa6bde6f1da840cbc5ccf1
                Timestamp: 2022-12-15T13:21:36Z
$ ls -lah /etc/initial-blueprint.toml
-rw-r--r--. 1 root root 553 Dec 15 09:17 /etc/initial-blueprint.toml
```

In order to demonstrate the update process, we can change the container once more:

```
$ cat Containerfile
FROM quay.io/kwozyman/r4e:latest
ADD blueprint.toml /etc/initial-blueprint.toml
RUN echo 'Hello Edge!' > /etc/motd
$ podman build . -t quay.io/kwozyman/r4e:changed && podman push quay.io/kwozyman/r4e:changed
```

After the push, we can try to upgrade our system:

```
$ sudo rpm-ostree upgrade
Pulling manifest: ostree-unverified-image:docker://quay.io/kwozyman/r4e:changed
Importing: ostree-unverified-image:docker://quay.io/kwozyman/r4e:changed (digest: sha256:b88f193482b86141455c7307bdb53a07c2d037571c8f44794b3392be894bd1cc)
ostree chunk layers stored: 1 needed: 0 (0 bytes)
custom layers stored: 2 needed: 1 (341 bytes)
Fetching layer sha256:84da2894fdca (341 bytes)
Fetched layer sha256:84da2894fdca
Checking out tree 9fc2949... done
Enabled rpm-md repositories: rhel-9-for-x86_64-baseos-rpms rhel-9-for-x86_64-appstream-rpms
Importing rpm-md... done
rpm-md repo 'rhel-9-for-x86_64-baseos-rpms' (cached); generated: 2022-12-05T16:22:43Z solvables: 3061
rpm-md repo 'rhel-9-for-x86_64-appstream-rpms' (cached); generated: 2022-12-14T09:02:08Z solvables: 10460
Resolving dependencies... done
Checking out packages... done
Running pre scripts... done
Running post scripts... done
Running posttrans scripts... done
Writing rpmdb... done
Writing OSTree commit... done
Staging deployment... done
Freed: 40.3 MB (pkgcache branches: 0)
Run "systemctl reboot" to start a reboot
$ systemctl reboot
$ rpm-ostree status
State: idle
Deployments:
● ostree-unverified-registry:quay.io/kwozyman/r4e:changed
                   Digest: sha256:b88f193482b86141455c7307bdb53a07c2d037571c8f44794b3392be894bd1cc
                Timestamp: 2022-12-15T14:24:03Z

  ostree-unverified-registry:quay.io/kwozyman/r4e:changed
                   Digest: sha256:b88f193482b86141455c7307bdb53a07c2d037571c8f44794b3392be894bd1cc
                Timestamp: 2022-12-15T14:17:22Z
```
