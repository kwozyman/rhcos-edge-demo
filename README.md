RHCOS Image Builder demo
====

Red Hat Enterprise Linux CoreOS (RHCOS) represents the next generation of single-purpose container operating system technology by providing the quality standards of Red Hat Enterprise Linux (RHEL) with automated, remote upgrade features. In other words, RHCOS is a immutable operating system from Red Hat, developed as an Openshift component.

Next, we will see how you customize the installation (and installed) image.

Requirements
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

