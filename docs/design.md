# Design

## BPM for Process Management

By using [BPM](https://github.com/cloudfoundry-incubator/bpm-release) for
process management, we are able to simplify the process execution and lifecycle
of ETCD. BPM forces the use of a declarative configuration for process execution,
and the remaining bash scripts are simple and easily audited.
By running inside containers, we no longer need to manage a large number of
generic concerns such as pidfiles, log redirection, process limits, and
filesystem permissions. We can now rely on a heavily tested Go program than
largely untested bash scripts.

An added feature of using BPM is that we greatly increase the security profile
of the ETCD release. BPM runs processes inside
[runc](https://github.com/opencontainers/runc) containers with de-escalated
privileges. You can read more about the benefits of using BPM
[here](https://github.com/cloudfoundry-incubator/bpm-release/blob/master/README.md).

## Strictly Following ETCD Documentation

ETCD has a large set of [well documented
recommendations](https://coreos.com/etcd/docs/latest/op-guide/clustering.html)
for operating clusters. A major goal in the implementation of this release is
to codify these recommendations.

### Bootstrapping the Cluster

One intentional design decision is forcing the responsibility of bootstrapping
the cluster on the operator. Rather than develop a complex set of logic around
attempting to detect initial deploys, this release forces the operator to be
declarative about bootstrapping a cluster. This is reflected in the fact that a
clusters initial deploy must be performed with the `bootstrap.yml` operation
file:

```
bosh -d etcd deploy deployment/etcd.yml --vars-store vars.yml -o deployment/operations/bootstrap.yml
```

Through this declarative operation and the static DNS addresses provided by
[BOSH DNS](https://bosh.io/docs/dns.html), we are able to closely follow the
[static bootstrapping
documentation](https://coreos.com/etcd/docs/latest/op-guide/clustering.html#static)
provided by ETCD. This is reflected in the [bash
script](../jobs/etcd/templates/etcd.erb) used to launch ETCD.

**Note**: This declarative configuration can be removed if BOSH implements a reliable way for jobs to detect initial deploys.
**Note**: A cluster can continue to be deployed using the `bootstrap.yml` operations file if the size and membership of the cluster do not change.

### Normal Cluster Operation

When BOSH performs a deploy it will stop the job, remount the persistent disk
if necessary, and then start the job. More documentation on this can be found
[here](https://bosh.io/docs/job-lifecycle.html). This process is largely
identical to the [member migration
documentation](https://coreos.com/etcd/docs/latest/v2/admin_guide.html#member-migration)
provided by ETCD. Updating the peer URL of the node is not necessary as BOSH
DNS will keep the DNS address of the node static as the GUID used to identify
the instance will not change. With regards to the data directory, a BOSH deploy
will result in one of the following cases:
* Normal Deploy: The disk will not be unmounted and thus the data directory will remain unchanged.
* Stemcell Deploy: The disk will not be remounted after VM creation and prior to starting the process.
* Persistent Disk Size Change: The contents of the previous persistent disk will be copied to the new persistent disk prior to starting the process.

Thus BOSH handles all of the management for the persistent data directory
properly. The only remaining responsibility of the release is to properly stop
and launch the ETCD process.

### Scaling Clusters

In order to scale the cluster up and down, the operator is also forced into a
declarative operation. Specifically the operator must change the number of
instances in the ETCD job and remove the `bootstrap.yml` operations file.

When scaling the cluster up, if a node has no data in it's persistent data
directory it will follow the documentation for [adding a new
member](https://coreos.com/etcd/docs/latest/op-guide/runtime-configuration.html#add-a-new-member).
By calling `etcdctl member add`, we are provided with the clustering flags
necessary to deploy the node.

**Note**: Due to this necessary runtime configuration, we are unable to use a
configuration file for this release. This is due to the fact that BPM prevents
a job from writing to its configuration directory by mounting it as read only.
ETCD will ignore any flags passed on the command line if a configuration file
is provided.

When scaling the cluster down, if a node detects that the `BOSH_JOB_NEXT_STATE`
environment variable shows that the persistent disk is being removed, the
process for [removing a
member](https://coreos.com/etcd/docs/latest/op-guide/runtime-configuration.html#remove-a-member)
will be followed. This can be seen in the [drain
script](../jobs/etcd/templates/drain.erb).  Further documentation for drain
scripts can be found [here](https://bosh.io/docs/drain.html).

### Disaster Recovery

Disaster recovery is intentionally not tackled by this release. The goal of
this release is to provide consistency and stability during normal operation.
If for some reason disaster recovery steps are required, the operator will be
required to perform these actions manually. In the future we could provide a
set of errands to successfully perform disaster recovery, but I would like to
keep the disaster recovery process a manual process.

## Operator Experience

A goal of this relase was to improve the user experience for operators.
By using BPM, the operator is provided with a useful set of commands to
strace their process (`bpm trace`), tail logs (`bpm logs`), and inspect
process state (`bpm pid`, `bpm list`, etc). By default `bpm` will be placed
on the `PATH` of an operator when launching a login shell.

Following the above design, this release provides a [useful helper
script](../jobs/etcd/templates/etcdctl.erb) which simplifies interacting
with the ETCD cluster for debug purposes. This binary will also be places
on the `PATH` of the operator whenever initiating a login shell.

**Note**: It is recommended that users use `sudo -i` after `bosh ssh` so that
the a new login shell is launched when escalating to the `root` user.

## Configuration Properties

The [configuration properties](../jobs/etcd/spec) exposed by this release very
closely reflect the [configuration properties exposed by
ETCD](https://coreos.com/etcd/docs/latest/op-guide/configuration.html). This is
intentional as I wanted to make operating the BOSH release of ETCD very similar
to operating ETCD manually.

The two exceptions are the security and clustering flags. The security
configuration of the release enforces Mutual Auth TLS. The clustering flags are
closely tied to BOSH DNS and BOSH lifecycle so they are managed for the user.

## Follow BOSH Exemplar Release

This release tries to follow many of the principles set forth by the [exemplar
release](https://github.com/cloudfoundry/exemplar-release). This includes only
checking process health with regards to starting the process with `monit`. It
also exposes client and peer configuration through separate links for
consumption by other BOSH jobs.

**Note**: I have considered adding a simple health check in the `post-start` as
[documented](https://github.com/cloudfoundry/exemplar-release#post-start-docs),
but have not fully considered the side effects of this change.
