# CoreOS Internals docs

This document intends to be a dumping ground to briefly describe various problem domains we've hit around building/delivering/testing CoreOS style systems.

Other important links:

 - https://github.com/coreos/coreos-assembler/
 - https://github.com/coreos/fedora-coreos-config

# Initramfs 

This topic is big enough to have its own document: [README-initramfs.md](README-initramfs.md).

# CPU microcode

rpm-ostree runs dracut on the server side, and dracut knows how to pick up CPU microcode and prepend it to the initramfs.  Relevant bugs:

- https://bugzilla.redhat.com/show_bug.cgi?id=1199582
- https://bugzilla.redhat.com/show_bug.cgi?id=1803883

# Entropy

As of recently we enable `CONFIG_RANDOM_TRUST_CPU` which covers modern `x86_64` systems for example.

- https://bugzilla.redhat.com/show_bug.cgi?id=1830280
- https://github.com/openshift/machine-config-operator/issues/854

# Networking

In [this tracker issue](https://github.com/coreos/fedora-coreos-tracker/issues/24) a decision was made to use NetworkManager.  As of recently we use NetworkManager in the initramfs.  And even more recently, things have been reworked so that [afterburn can control initramfs networking](https://github.com/coreos/afterburn/pull/404) on specific clouds.

# Time synchronization

We use chrony, with some [additional custom logic for specific clouds](https://github.com/coreos/fedora-coreos-config/blob/faf387eac89d14924a1e2021d2093d0cdb8af8b3/overlay.d/20platform-chrony/usr/lib/systemd/system-generators/coreos-platform-chrony).
See also DHCP propagation: https://github.com/coreos/fedora-coreos-config/pull/412

# Aleph version

`rpm-ostree status` will show admins the state of the ostree, but a few things live outside that and are not subject to in place updates.  For example, the on-disk filesystem (default `xfs`) and its specific layout, as well as the bootloader.

See [this pull request](https://github.com/coreos/coreos-assembler/pull/768/commits/2701e91838e18d3eac0694fd0a5f003befcfb218) which added `/sysroot/.coreos-aleph-version.json` that can be used to track the version of that data.

# ignition.platform.id

See https://docs.fedoraproject.org/en-US/fedora-coreos/platforms/

The design we have today is that each CoreOS system is the same OS content - the same OSTree commit,
and beyond that the exact same bootloader version, etc.

There are differences per platform on the image formats (VHD versus qcow2 vs raw, etc).  However,
what's *inside* the disk image for each platform is almost the same.

A key difference between each image is the `ignition.platform.id` kernel argument.  From the
moment the system boots and the kernel loads the initramfs, our userspace code uses this
to reliably know its target platform.  As could be guessed from the name, [https://github.com/coreos/ignition/](ignition)
uses this, and it runs early on.

But there's other code which dynamically dispatches on the platform ID:

- https://github.com/coreos/afterburn/
- [The time sync setup code](https://github.com/coreos/fedora-coreos-config/blob/d87b52bc6a90b53e1afeab2731b52612d5e3bbc0/tests/kola/chrony/coreos-platform-chrony-generator#L9)
- [network requirement detection](https://github.com/coreos/fedora-coreos-config/blob/d87b52bc6a90b53e1afeab2731b52612d5e3bbc0/overlay.d/05core/usr/lib/dracut/modules.d/35coreos-network/coreos-enable-network.service#L13)

Notice in particular how the time synchronization code ends up reconfiguring chrony dynamically.
For other operating systems which do "per cloud" disk images, it would have been more
natural to just change `/etc/chrony.conf` per platform.  But that would mean we have a different
ostree commit checksum per platform, breaking our "image based" update model.

# multipath

A lot of history here.  A TL;DR is that nontrivial multipath setups conceptually conflict
a bit with the "CoreOS model" of booting into the desired configuration from the start.
There's also a long related issue in that we want to use a "pristine" initramfs in
general, and nontrivial multipath configuration needs to be in the initramfs.

What we ended up with is adding an `rd.multipath=default` kernel argument which 
triggers dracut to do "basic" automatic multipath setup in the stock initramfs:
https://github.com/dracutdevs/dracut/pull/780

So we still have a model then where the host boots up in a non-multipath
configuration, Ignition runs and the kernel arguments are applied, then we reboot into the
final configuration.

We don't yet document multipath for FCOS, but we do document this setup for
OpenShift that has a kola test:

- https://github.com/coreos/coreos-assembler/blob/60f675ec5037b84c01f17192d773a14166dc6a14/mantle/kola/tests/misc/multipath.go#L57

More links:

- https://github.com/coreos/ignition-dracut/issues/154
- https://bugzilla.redhat.com/show_bug.cgi?id=1944660


An example issue seems to be rooted in our use of labels to find `boot`
and `root`. The labels seem to be racy in our current code because
`multipathd.service` may take over the block devices.
