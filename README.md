
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
can be installed under different names, possibly by making symbolic
links to its binary from those
`/usr/libexec/kubernetes/kubelet-plugins/volume/exec` subdirectories.
It will search for "helper" programs in the matching subdirectories
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
parameter `get`, the path of the file under the mountpoint, and mount
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
to exist. If it is either a copy or a symbolic link of
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
- driver: example.com/hashicorp-cli-fuse
```

# Authorization in the external vault

The FUSE flexVolume driver and its helper are invoked as `root` on the
Node, outside of the containers. So it can authenticate to external
secret source or vault using the identity and credentials of the Node,
for example with Kerberos principal (host's `krb5.keytab` or separate
one), TLS certificate, or a token.

The identity of the Pod is passed to the flexVolume driver during
`mount` (`kubernetes.io/pod.*` flags), and by configuring `mount-param`,
the information gets to the helper program. It can incorporate the pod
identity in the name of the secret being requested from the external
vault, for example the [hashicorp-cli-helper.sh](hashicorp-cli-helper.sh)
shows invoking `hashicorp-vault read` with parameter
`secret/pod-secrets/$3/$4/${2%/*}` where the `$2` is the path under the
mountpoint and `$3` and `$4` are the subsequent parameters specified by
`mount-param`.

In a pod `test-pod` in namespace `default` with
`example.com/hashicorp-cli-fuse` flexVolume mounted under
`/mnt/hashicorp` and using the [example helper](hashicorp-cli-helper.sh),

```
cat /mnt/hashicorp/db/password
```

in the container will execute

```
hashicorp-vault read -field="password" "secret/pod-secrets/default/test-pod/db"
```

on the Node.

Alternatively, with different vault, the pod identity information
can be passed for example in HTTP(S) request headers, for example

```
curl -H 'X-Pod-Namespace: default' -H 'X-Pod-name: test-pod' ...
```

Depending on the Kubernetes environment setup, different access
control approaches and authorization mechanisms might be deployed
on the external vault or in front of it, in authorization service or
proxy. From a rather static one when given set of Nodes, access to
secrets under certain path (for example based on namespace) is
allowed, to dynamic verification that the given Pod is currently
scheduled on the given Node.

For the second approach,
[custodia.kubernetes](https://github.com/latchset/custodia.kubernetes)
plugin to [Custodia](https://github.com/latchset/custodia) can be
used. It uses the Kubernetes API to do the Pod-is-on-given-Node check.
The [custodia-cli-helper.sh](custodia-cli-helper.sh) example helper
shows one possible way to invoke remote Custodia, via

```
custodia-cli get ...
```

call.

Authorization of the request can also happen directly on the Node but
will most likely be be done on the remote vault anyway.

In any case, the authorization of the request on the remote vault is
outside of the scope of this FUSE flexVolume driver.

Alternatively, the helper could likely use some different secret of
the Pod to authenticate with the identity of the Pod directly.

# Deployment expectations

## Administrator

* The flexVolume FUSE driver needs to be installed and configured on
Masters and Nodes, ideally via symbolic links from under
`/usr/libexec/kubernetes/kubelet-plugins/volume/exec`, providing the
name of the driver under which it will be known to Kubernetes.

* The helper script with matching name must exist and should support
`mount` and `get` parameters as described above.

* If the helper script or tools called from it should authenticate to
the external vault or secrets source with the Node identity, Nodes'
identities have to be established and credentials placed on them. This
might include IPA-enrolling the Node hosts to Identity Management /
FreeIPA solution, joining to Active Directory, getting and installing
client certificates, or per-Node tokens.

* On the external vault, access control or authorization mechanisms
should be deployed.

* PSP which allows `flexVolume` and appropriate flexVolume `driver`
need to be created and users whose Pods should have access to the
secrets should be added to those PSPs.

## User

* The Pod specification will use `volumeMounts` configuration with the
FUSE flexVolume `name` and appropriate `mountPath`. Secrets will be
presented under that mount path similar to
[Secret volumes](https://kubernetes.io/docs/concepts/configuration/secret/).

* Images can be adjusted to configure that secrets should be found in
files under the mount path, for example `SSLCertificateFile
/mnt/custodia/certs/HTTP/default` for Apache HTTP Server's mod_ssl
module, configured either in the images or in Pod specification via
environment variables (`PGPASSFILE=/mnt/hashicorp/db/password` for
PostgreSQL), or symbolic links can be created in the images pointing
from the default locations to the location under mount path.
