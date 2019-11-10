# GoCD

## Passing .kube/config as a volume to GoCD Agent Pod

First we need to [generate *.kube/config*](docs/generating-kubeconf.md) and then store it in a Kubernetes secret. After finished generating a `.kube/config` file you should store its content in a file let's say `/tmp/stage-kubeconf` issue the following commands to create a secret:

```bash
kubectl create secret generic -n gocd gocd-deployer-kubeconf --from-file=config=/tmp/stage-kubeconf
```

Now this secret can be mounted by GoCD Agent Pod:

```yaml
      volumeMounts:
      - name: kubeconf
        mountPath: /root/.kube
        readOnly: true
      volumes:
      - name: kubeconf
        secret:
          secretName: gocd-deployer-kubeconf
```

For the whole example refer to [agent-runners/default-with-kubeconf.yaml](agent-runners/default-with-kubeconf.yaml).


## Documentation
### [Generating .kube/config file for GoCD deployer](docs/generating-kubeconf.md)
