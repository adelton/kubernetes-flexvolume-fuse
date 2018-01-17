
Kubernetes FUSE flexVolume driver for external secret handling
==============================================================

# Motivation

* For containers to be able to fetch secrets from external secret
sources or vaults, via mechanism which does not require additional
logic in the containers.

* Expose the secrets as files but retrieve their content only when the
software in containers needs them.

* Proxy the request to the external vault authenticated as the Node but
having the ability to pass the Pod identity with the request, for
auditing and authorization control.

* Have easily extensible mechanism which invokes configured helper
program or shell script to fetch the secrets.

# Workflow

Upon startup, Master and Nodes discover available flexVolume drivers
(plugins) by searching `/usr/libexec/kubernetes/kubelet-plugins/volume/exec`
subdirectories for files `<vendor>~<driver>/<driver>` and executing
them with `init` parameter.

The flexVolume driver [flexvolume-fuse-external](flexvolume-fuse-external.c)
can be installed under different names, possibly by making symlinks
to its binary from those
`/usr/libexec/kubernetes/kubelet-plugins/volume/exec` subdirectories.
It will search for "helper" programs in the matchin subdirectories
under `/usr/libexec/kubernetes/kubelet-plugins/volume/libexec`.

If found, the driver will confirm that it is operational and report
capabilities

```
{
    "attach": false,
    "selinuxRelabel": false
}
```

When creating the Pod and preparing its mountpoints on a Node, the
flexVolume driver will be executed with `mount` parameter and two
additional parameters, mountpoint path and JSON hash of flags, for
example

```
/var/lib/kubernetes/volumes/pods/215904af-a29b-11e7-a06b-5254005fe346/volumes/example.com~secrets-fuse/secrets-fus
e
{
    "kubernetes.io/fsGroup":"1000020000",
    "kubernetes.io/fsType":"",
    "kubernetes.io/pod.name":"test-pod",
    "kubernetes.io/pod.namespace":"default",
    "kubernetes.io/pod.uid":"215904af-a29b-11e7-a06b-5254005fe346",
    "kubernetes.io/pvOrVolumeName":"hashicorp-cli",
    "kubernetes.io/readwrite":"rw",
    "kubernetes.io/serviceAccount.name":"default"
}
```

The flexVolume driver [flexvolume-fuse-external](flexvolume-fuse-external.c)
will call the helper program with `mount` parameter and expect JSON
hash on standard output which will configure the behaviour of the
driver when entries on the FUSE filesystem get accessed. Supported
values in the helper response to the `mount` operation are:

* `enable-dirs`: array of directories that should be "enabled" in the
FUSE filesystem;

* `mount-param`: array of `mount` flag names -- their values will be
passed as third and subsequent parameter when the helper gets later
invoked with `get` parameter, retrieving the entry content.

For example, the helper can respond to the `mount` call with standard
output content

```
{
    "enable-dirs": [ "/db" ],
    "mount-param": [ "kubernetes.io/pod.namespace", "kubernetes.io/pod.name" ],
}
```

meaning under the mountpoint in the container `db/` directory will be
presented by the FUSE flexVolume driver.

This mechanism allows using the same binary FUSE flexVolume driver,
symlinked under different names, and use custom helper scripts not
just to retrieve the secrets but also to configure which additional
information will be passed to the helper when the file on the
FUSE-backed volume get accessed.

Access to files under the mountpoint in containers will invoke the
FUSE flexVolume driver which will in turn call the helper with
parameter `get`, the path of the file under the mountpount, and mount
flag values configured by the `mount-param` array from helper's
`mount` response. The `kubernetes.io/pod.namespace`,
`kubernetes.io/pod.name`, and `kubernetes.io/pod.uid` are the most
obvious choices as they represent Pod's identity which can be passed
to the helper and then to the external vault, for authorization
purposes and to present each Pod with its own secret value.

The FUSE flexVolume driver will cache the last retrieved secret to
avoid repetitive invocations of the helper for multiple `getattr`,
`open`, and `read` operations for the same file.

# Example Pod

```
apiVersion: v1
kind: Pod
metadata:
  name: test-pod-hashicorp
spec:
  containers:
  - image: registry.access.redhat.com/rhel7
    name: test-pod-hashicorp
    command: ["/usr/bin/sleep"]
    args: ["infinity"]
    volumeMounts:
    - mountPath: /mnt/hashicorp
      name: hashicorp-cli
  volumes:
  - flexVolume:
      driver: example.com/hashicorp-cli-fuse
    name: hashicorp-cli
```

When Pod above is created, it will expect
`/usr/libexec/kubernetes/kubelet-plugins/volume/exec/example.com~hashicorp-cli-fuse/hashicorp-cli-fuse`
to exist. If it is either a copy or a symlink of 
[flexvolume-fuse-external](flexvolume-fuse-external.c), it will in
turn expect
`/usr/libexec/kubernetes/kubelet-plugins/volume/libexec/example.com~hashicorp-cli-fuse/hashicorp-cli-fuse`
to exist which will configure the flexVolume FUSE driver during
`mount` operation and which will be expected to produce secrets on
standard output when invoked with `get` parameter when files on the
FUSE filesystem will be retrieved.

An [example helper](hashicorp-cli-helper.sh) will configure `/db`
subdirectory, so in the container, `/mnt/hashicorp` and
`/mnt/hashicorp/db` will be presented and access to files under
`/mnt/hashicorp/db` will cause the `get` helper invocation.

# Access control in Kubernetes

For users to be able to use the FUSE flexVolume driver, they need to
be added to Pod Security Policy (PSP) which allows `flexVolume` volume
and `<vendor>/<driver>` driver. For example, when the driver is named
`example.com/hashicorp-cli-fuse`, the PSP should include

```
volumes:
- flexVolume
allowedFlexVolumes:
- driver: example.com/custodia-cli-fuse
```

# Authorization in the external vault

The FUSE flexVolume driver and its helper are invoked as `root` on the
Node, outside of the containers. So it can authenticate to external
secret source or vault using the identity and credentials of the Node,
be it Kerberos principal, TLS certificate, or a token. The identity of
the Pod is passed to the flexVolume driver during `mount` so this
information can be passed along when retrieving the secret.

Authorization of the request can happen directly on the Node but will
most likely be be done on the remote vault. Since both the identity of
the Node and the Pod is known, the authorization mechanism on the
external site can for example verify via Kubernetes API that given
Pod is indeed currently scheduled on the Node from which the request
is seen coming.

In any case, the authorization of the request on the remote vault is
outside of the scope of this FUSE flexVolume driver.

Alternatively, the helper could likely use some different secret of
the Pod to authenticate with the identity of the Pod directly.
