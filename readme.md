run

```sh
aws configure
openshift-install create install-config --dir ./openshift-install
openshift-install create cluster --dir ./openshift-install
```

```sh
export gitops_repo=https://github.com/trevorbox/acm-virt-dr.git #<your newly created repo>
export cluster_name=hub #<your hub cluster name, typically "hub">
export cluster_base_domain=$(oc get ingress.config.openshift.io cluster --template={{.spec.domain}} | sed -e "s/^apps.//")
export platform_base_domain=${cluster_base_domain#*.}
export region=$(oc get infrastructure cluster -o jsonpath='{.status.platformStatus.aws.region}')
oc apply -f .bootstrap/subscription.yaml
oc apply -f .bootstrap/cluster-rolebinding.yaml
sleep 60
envsubst < .bootstrap/argocd.yaml | oc apply -f -
sleep 30
envsubst < .bootstrap/root-application.yaml | oc apply -f -
```

> Note: needed to `oc delete certmanager cluster` to fix race condition

To get the prod and non-prod cluster created you'll have to prepare a secret in the way ACM expects it, then run:

```sh
oc new-project open-cluster-management
oc delete secret aws-credentials -n open-cluster-management
# cat ~/.aws/credentials
export AWS_KEY=
export AWS_SECRET=
export SSH_PRIV_KEY= #$(cat ~/.ssh/id_aws.pub)
oc create secret generic aws-credentials \
  --from-literal=ssh-privatekey="${SSH_PRIV_KEY}" \
  --from-literal=aws_access_key_id="${AWS_KEY}" \
  --from-literal=aws_secret_access_key="${AWS_SECRET}" \
  -n open-cluster-management
oc apply -f ./.rosa/aws-secret.yaml
```

If you are using RHDP, after a restart run this to deal with the annoying race condition affecting ArgoCD

```sh
oc rollout restart deployment/openshift-gitops-operator-controller-manager -n openshift-operators
oc delete pods --all -n openshift-gitops
```