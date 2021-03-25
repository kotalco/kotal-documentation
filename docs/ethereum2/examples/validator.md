---
title: Validator Client
---

```yaml
# validator.yaml
apiVersion: ethereum2.kotal.io/v1alpha1
# highlight-next-line
kind: Validator
metadata:
  name: teku-validator
# highlight-start
spec:
  client: teku
  network: mainnet
  beaconEndpoint: http://10.96.130.88:9999
  graffiti: Validated by Kotal
  keystores:
    - secretName: my-validator
# highlight-end
```

In the example above, we're deploying a validator client that uses ConsenSys Teku Ethereum 2.0 client `client: teku`, and syncing the main network beacon chain `join: mainnet`. We're connecting to beacon node endpoint using `beaconEndpoint: ...`. We set the graffiti text to include in propsed block `graffiti: validated by Kotal`. Loaded keystores using `keystores: ...`.

:::caution
In the example above, the keystore secret `my-validator` must exist before deploying the validator `teku-validator`, and it must contain two keys: `keystore` key hodling the BLS12-381 keystore JSON file, and `password` key holding the encryption key of the keystore.
:::

Deploy the above validator client manifest, and kotal will kick in and do everything for you:

```sh
$ kubectl apply -f validator.yaml

validator.ethereum2.kotal.io/teku-validator created
```

Confirm that the validator client has been created by:

```sh
$ kubectl get validators

NAME               AGE
teku-validator   1m
```

Get the pods created for the validator client:

```sh
$ kubectl get pods

NAME                 READY   STATUS    RESTARTS   AGE
teku-validator-0     1/1     Running   0          1m
```

Get the logs of the running validator pod:

```sh
$ kubectl logs -f teku-validator-0
```

You'll get output similar to the following:

```sh
...
21:21:30.668 INFO  - Storing validator data in: /opt/teku/kotal-data/validator
21:21:30.751 INFO  - Loading 1 validator keys...
21:21:31.759 INFO  - Loaded 1 Validators: 83dbb18
...
```

Finally delete the validator client and kubernetes will delete all resources created for the validator client like pods, services ... etc:

```sh
$ kubectl delete validator teku-validator

validator.ethereum2.kotal.io "teku-validator" deleted
```

:fire: