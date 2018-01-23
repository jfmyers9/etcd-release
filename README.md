# etcd-release

This is an experimental implementation of an ETCD release using BPM.

**NOTE**: This release is not meant for external consumption, and is mainly an
exploration to learn more about ETCD.

## Deploying

By design, an initial deploy of this ETCD release requires the user to specify
that they are bootstrapping the cluster.  This can be done using the operations
file provided. For example:

```
bosh -d etcd deploy deployment/etcd.yml -o deployment/operations/bootstrap.yml --vars-store vars.yml
```

On subsequent deploys, one should remove the bootstraping operations file as
follows:

```
bosh -d etcd deploy deployment/etcd.yml --vars-store vars.yml
```

Once the bootstraping operations file has been removed, the user can safely
scale the cluster up and down.  Please refer to the [ETCD clustering
documentation](https://coreos.com/etcd/docs/latest/v2/clustering.html) for more
information.
