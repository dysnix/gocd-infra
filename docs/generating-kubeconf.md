# Generating .kube/config file for GoCD deployer

This section will walk you through custom `.kube/config` file which can be inject into a deployer user configuration. Look at the sample `.kube/config` file bellow before we actually try to generate it:

```yaml
apiVersion: v1
kind: Config
users:
- name: gocd-deployer
  user:
    token: <replace this with token info>
clusters:
- cluster:
    certificate-authority-data: <replace this with certificate-authority-data info>
    server: <replace this with server info>
  name: self-hosted-cluster
contexts:
- context:
    cluster: mycluster
    user: gocd-deployer
  name: mycluster
current-context: gocd-deployer
```

To generate the file as above to access a Kubernetes cluster we need to satisfy the following points:

- Should already have an access to the desired cluster
- Have `kubectl` installed
- Create a deployer ServiceAccount
- Create a ClusterRoleBinding for the newly created ServiceAccount
- Run a magic script to build up the config.

We assume that the first two points are already satisfied. Now let's create a `gocd-deployer` user and a cluster role binding:

```bash
NS=gocd
SA=gocd-deployer

kubectl create serviceaccount -n $NS $SA
kubectl create clusterrolebinding --clusterrole cluster-admin \
    --serviceaccount $NS:$SA $SA-binding
```

The last but not least to do some bash magic and you might expect the `.kube/config` printed out :).

```bash
cat <<EHD | NS=gocd SA=gocd-deployer bash -s
export NS
export SA

# scratch token name and value
_sa_token_secret=\$(kubectl get serviceaccount -n \$NS \$SA -o jsonpath='{.secrets[0].name}')
_sa_token=\$(kubectl get secret -n \$NS \${_sa_token_secret} -o jsonpath='{.data.token}' | base64 --decode)

# scratch the rest
_kube_context=\$(kubectl config current-context)

_kube_cluster=\$(kubectl config view -o jsonpath="{.contexts[?(@.name=='\${_kube_context}')]..cluster}")
_kube_server=\$(kubectl config view -o jsonpath="{.clusters[?(@.name=='\${_kube_cluster}')]..server}")

_tmpcrt=/tmp/\$$.gen.kube.ca.crt
kubectl get secret -n \$NS \${_sa_token_secret} -o jsonpath='{.data.ca\.crt}' | base64 --decode > \${_tmpcrt}

export KUBECONFIG=/tmp/\$$.gen.kube.conf
kubectl config set-cluster \${_kube_cluster} --server=\${_kube_server} --certificate-authority=\${_tmpcrt} --embed-certs=true >& /dev/null
kubectl config set-credentials \$SA --token=\${_sa_token} >& /dev/null
kubectl config set-context \${_kube_cluster} --cluster \${_kube_cluster} --user \$SA
kubectl config use-context \${_kube_cluster}

echo -e "\n.kube/conf:\n\n---"
cat \${KUBECONFIG}
rm -f \${_tmpcrt} \${KUBECONFIG}
EHD
```
