---
title: Create IPFS ClusterPeer
---

> IPFS Cluster provides data orchestration across a swarm of IPFS daemons by allocating, replicating and tracking a global pinset distributed among multiple peers.
> [Source](https://cluster.ipfs.io)

Using `ClusterPeer` resource, you can deploy ipfs cluster peers which maintain pinset across multiple peers.

### Prerequisites

Before we start, you need to:

1. Deploy an ipfs peer `Peer` because it's required by `ClusterPeer`. Check our [create ipfs peer](peer) guide.
2. Create cluster secret which secures access to cluster:

```bash
$ CLUSTER_SECRET=$(openssl rand -hex 32)
$ kubectl create secret generic cluster-secret --from-literal=secret=$CLUSTER_SECRET

```

3. Create cluster peer ID and private key using [ipfs-key](https://github.com/whyrusleeping/ipfs-key)

```bash
$ ipfs-key -type ed25519 | base64

Generating a 2048 bit ed25519 key...
Success!
# highlight-start
ID for generated key: 12D3KooWBcEtY8GH4mNkri9kM3haeWhEXtQV7mi81ErWrqLYGuiq
CAESQOH/DvUJmeJ9z6m3wAStpkrlBwJQxIyNSK0YGf0EI5ZRGpwsWxl4wmgReqmHl8LQjTC2iPM0QbYAjeY3Z63AFnI=
# highlight-end
```

4. create Kubernetes secret from the generated private key in step(3) above.

```bash
$ CLUSTER_PEER_PRIVATEKEY=CAESQOH/DvUJmeJ9z6m3wAStpkrlBwJQxIyNSK0YGf0EI5ZRGpwsWxl4wmgReqmHl8LQjTC2iPM0QbYAjeY3Z63AFnI=
$ kubectl create secret generic cluster-peer-privatekey --from-literal=key=$CLUSTER_PEER_PRIVATEKEY
```

## Deploying First Cluster Peer

![cluster peer](/img/ipfs-cluster-peer.png)

Now we're ready yo deploy a cluster peer and connect it to the peer we've deployed in step(1)

```yaml
#cluster-peer.yaml
apiVersion: ipfs.kotal.io/v1alpha1
kind: ClusterPeer
metadata:
  Name: cluster-peer
spec:
  # generated in step(3)
  id: "12D3KooWBcEtY8GH4mNkri9kM3haeWhEXtQV7mi81ErWrqLYGuiq"
  # generated in step(4)
  privatekeySecretName: cluster-peer-privatekey
  # deployed in step(1)
  peerEndpoint: /dns4/peer-sample/tcp/5001
  # generated in step(2)
  clusterSecretName: cluster-secret
```

:::note
Note that `id` and `privateKeySecretName` are optional, but they're useful to get predictable cluster peer id to be used in future steps as bootstrap peer.

Default consensus is `crdt` and can be changed using `consensus: ` spec parameter.
:::

Let's deploy cluster peer manifest file:

```bash
$ kubectl apply -f cluster-peer.yaml

clusterpeer.ipfs.kotal.io/cluster-peer created
```

Kotal `ClusterPeer` controller will get a notification that a new `ClusterPeer` resource has been deployed, and will create all the Kubernetes resources (pod, service, configmap ...) necessary for it to work as expected. You can verify peer resources has been created by fetching all cluster peers:

```bash
$ kubectl get clusterpeers

NAME           AGE
cluster-peer   10s
```

Get the pods created by our cluster peer:

```bash
$ kubectl get pods

NAME             READY   STATUS    RESTARTS   AGE
cluster-peer-0   1/1     Running   0          2m
```

Check the logs of the running cluster peer:

```bash
$ kubectl logs -f cluster-peer-0

2021-04-30T00:31:24.109Z	INFO	service	ipfs-cluster-service/daemon.go:46	Initializing. For verbose output run with "-l debug". Please wait...
2021-04-30T00:31:24.133Z	INFO	cluster	ipfs-cluster/cluster.go:136	IPFS Cluster v0.13.2+git812c8e3631d4d6771a6e04112e12cd7726e23f7a listening on:

        /ip4/10.244.0.8/tcp/9096/p2p/12D3KooWBcEtY8GH4mNkri9kM3haeWhEXtQV7mi81ErWrqLYGuiq
        /ip4/127.0.0.1/tcp/9096/p2p/12D3KooWBcEtY8GH4mNkri9kM3haeWhEXtQV7mi81ErWrqLYGuiq

2021-04-30T00:31:24.134Z	INFO	ipfsproxy	ipfsproxy/ipfsproxy.go:320	IPFS Proxy: /ip4/127.0.0.1/tcp/9095 -> /ip4/127.0.0.1/tcp/5001
2021-04-30T00:31:24.134Z	INFO	restapi	rest/restapi.go:521	REST API (HTTP): /ip4/127.0.0.1/tcp/9094
2021-04-30T00:31:24.134Z	INFO	crdt	go-ds-crdt@v0.1.20/crdt.go:278	crdt Datastore created. Number of heads: 0. Current max-height: 0
2021-04-30T00:31:24.136Z	INFO	cluster	ipfs-cluster/cluster.go:651	Cluster Peers (without including ourselves):
2021-04-30T00:31:24.136Z	INFO	cluster	ipfs-cluster/cluster.go:653	    - No other peers
# highlight-next-line
2021-04-30T00:31:24.136Z	INFO	cluster	ipfs-cluster/cluster.go:666	** IPFS Cluster is READY **
```

As you can see our cluster peer is up and running as indicated by the healthy logs and the highlighted line `IPFS Cluster is READY`.

Let's add some content to the pinset by executing commands inside cluster peer container:

```bash
$ kubectl exec -it cluster-peer-0 -- sh
# now we're inside cluster-peer container

$ echo "kotal simplifies Blockchain DevOps" > kotal.txt

# add kotal.txt to the pinset
$ ipfs-cluster-ctl add kotal.txt
added QmUiwYtz1VxeryVCpTLWqrbeUi9HyGH4AnpSG6BrmdMhzL kotal.txt

# leave cluster-peer container
$ exit
```

Let's fetch the pinset of the ipfs peer by executing commands inside peer -created in step(1)- container:

```bash
$ kubectl exec -it peer-sample-0 -- sh
# now we're inside peer-sample container

# List objects pinned to local storage
$ ipfs pin ls
QmUNLLsPACCz1vLxQVkXqqLX5R1X345qqfHbsf67hvA3Nn recursive
QmUiwYtz1VxeryVCpTLWqrbeUi9HyGH4AnpSG6BrmdMhzL recursive

$ ipfs cat QmUiwYtz1VxeryVCpTLWqrbeUi9HyGH4AnpSG6BrmdMhzL
kotal simplifies Blockchain DevOps

# leave ipfs peer container
$ exit
```

As you can see, content pinned by the cluster peer, has been added to the pinset of the ipfs peer and available in its local storage.

## Deploying Second Cluster Peer

![cluster peer](/img/ipfs-cluster-peer-join.png)

The only prerequisite to the second cluster peer is:

1. Deploy an ipfs peer `Peer`, and **name it** `peer-sample-two`. It's required by `ClusterPeer`. Check our [create ipfs peer](peer) guide.

We will reuse cluster secret that was generated in step(2) during deploying first cluster peer above.

```yaml
# cluster-peer-two.yaml
apiVersion: ipfs.kotal.io/v1alpha1
kind: ClusterPeer
metadata:
  Name: cluster-peer-two
spec:
  # deployed in step(1)
  peerEndpoint: /dns4/peer-sample-twp/tcp/5001
  # generated during deploying first cluster peer in step(2)
  clusterSecretName: cluster-secret
  # first cluster peer listening address
  bootstrapPeers:
    - /ip4/cluster-peer/tcp/9096/p2p/12D3KooWBcEtY8GH4mNkri9kM3haeWhEXtQV7mi81ErWrqLYGuiq
```

Deploying this cluster manifest will deploy a cluster peer that connects to the first cluster peer and will connect to its own ipfs peer `peer-sample-two` instance.

```bash
$ kubectl apply -f cluster-peer-two.yaml

clusterpeer.ipfs.kotal.io/cluster-peer-two created
```

You can verify peer resources has been created by fetching all cluster peers:

```bash
$ kubectl get clusterpeers

NAME               AGE
cluster-peer       10m
cluster-peer-two   1m
```

Get the pods created by our cluster peer:

```bash
$ kubectl get pods

NAME                 READY   STATUS    RESTARTS   AGE
# ... other peers
cluster-peer-two-0   1/1     Running   0          2m
```

Check the logs of the running cluster peer:

```bash
$ kubectl logs -f cluster-peer-two-0

2021-05-01T19:17:12.043Z	INFO	service	ipfs-cluster-service/daemon.go:46	Initializing. For verbose output run with "-l debug". Please wait...
2021-05-01T19:17:12.352Z	INFO	cluster	ipfs-cluster/cluster.go:136	IPFS Cluster v0.13.2+git812c8e3631d4d6771a6e04112e12cd7726e23f7a listening on:

        /ip4/10.244.0.35/tcp/9096/p2p/12D3KooWGVZek7gdarvj7gePvnV9x8ekSeSdruYojgj5dmU466Wv
        /ip4/127.0.0.1/tcp/9096/p2p/12D3KooWGVZek7gdarvj7gePvnV9x8ekSeSdruYojgj5dmU466Wv


2021-05-01T19:17:12.441Z	INFO	restapi	rest/restapi.go:521	REST API (HTTP): /ip4/127.0.0.1/tcp/9094
2021-05-01T19:17:12.441Z	INFO	ipfsproxy	ipfsproxy/ipfsproxy.go:320	IPFS Proxy: /ip4/127.0.0.1/tcp/9095 -> /ip4/127.0.0.1/tcp/5001
2021-05-01T19:17:12.441Z	INFO	service	ipfs-cluster-service/daemon.go:218	Bootstrapping to /dns4/cluster-peer/tcp/9096/p2p/12D3KooWBcEtY8GH4mNkri9kM3haeWhEXtQV7mi81ErWrqLYGuiq
2021-05-01T19:17:12.442Z	INFO	crdt	go-ds-crdt@v0.1.20/crdt.go:278	crdt Datastore created. Number of heads: 0. Current max-height: 0
2021-05-01T19:17:12.442Z	INFO	crdt	crdt/consensus.go:277	'trust all' mode enabled. Any peer in the cluster can modify the pinset.
2021-05-01T19:17:12.447Z	INFO	cluster	ipfs-cluster/cluster.go:651	Cluster Peers (without including ourselves):
2021-05-01T19:17:12.447Z	INFO	cluster	ipfs-cluster/cluster.go:653	    - No other peers
# highlight-start
2021-05-01T19:17:12.448Z	INFO	cluster	ipfs-cluster/cluster.go:666	** IPFS Cluster is READY **
2021-05-01T19:17:12.551Z	INFO	cluster	ipfs-cluster/cluster.go:1020	12D3KooWGVZek7gdarvj7gePvnV9x8ekSeSdruYojgj5dmU466Wv: joined 12D3KooWBcEtY8GH4mNkri9kM3haeWhEXtQV7mi81ErWrqLYGuiq's cluster
# highlight-end
```

As you can see cluster peer two is up and running as indicated by the healthy logs and the highlighted line `IPFS Cluster is READY`.

Cluster peer two has joined the first cluster peer as indicated by the highlighted log line `joined 12D...uiq's cluster`

Finally, delete all peers and cluster peers and Kubernetes will delete all Kubernetes resources created for them like pods, services ... etc:

```bash
$ kubectl delete peers --all

peers.ipfs.kotal.io "peer-sample" deleted
peers.ipfs.kotal.io "peer-sample-two" deleted

$ kubectl delete clusterpeers --all

clusterpeers.ipfs.kotal.io "cluster-peer" deleted
clusterpeers.ipfs.kotal.io "cluster-peer-two" deleted
```

:fire:
