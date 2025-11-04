# ArgoCD Agents configuration 

Following the blog post here : https://developers.redhat.com/blog/2025/10/06/using-argo-cd-agent-openshift-gitops

## Notes on deployment

Deploying to a cluster that already has the GitOps operator installed and has an instance of ArgoCD in the openshift-gitops namespace.

Start in the openshift-gitops-agent directory

Tidy up any old contexts :

oc config delete-context principal
oc config delete-context managed-cluster

### Hub Cluster

oc apply -k control-plane/namespaces/base

oc config current-context

oc config rename-context $(oc config current-context) principal

argocd-agentctl pki init --principal-context principal --principal-namespace argocd 

export SUBDOMAIN=$(oc get dns cluster -o jsonpath='{.spec.baseDomain}')
echo $SUBDOMAIN

argocd-agentctl pki issue principal --dns argocd-agent-principal-argocd.apps.$SUBDOMAIN --principal-context principal --principal-namespace argocd

argocd-agentctl pki issue resource-proxy --dns argocd-agent-resource-proxy.argocd.svc.cluster.local --principal-context principal --principal-namespace argocd

oc apply -k control-plane/principal/base

oc create secret generic argocd-redis -n argocd --from-literal=auth="$(oc get secret argocd-redis-initial-password -n argocd -o jsonpath='{.data.admin\.password}' | base64 -d)"

oc rollout restart deploy/argocd-agent-principal -n argocd

oc get pods -n argocd

All pods should report running OK


argocd-agentctl agent create managed-cluster \
  --principal-context principal \
  --principal-namespace argocd \
  --resource-proxy-server argocd-agent-resource-proxy.argocd.svc.cluster.local:9090 \
  --resource-proxy-username foo \
  --resource-proxy-password bar


### Managed cluster

Assuming OpenShift GitOps operator is already installed on the cluster.

 oc login -u kubeadmin -p <password>  --insecure-skip-tls-verify=true <API URL>

oc apply -k managed-cluster/argocd/base

Wait for pods to be ready

oc get pods -n argocd

oc project argocd

oc config current-context

oc config rename-context $(oc config current-context) managed-cluster

oc create secret generic argocd-redis -n argocd --from-literal=auth="$(oc get secret argocd-redis-initial-password -n argocd -o jsonpath='{.data.admin\.password}' | base64 -d)"

argocd-agentctl pki issue agent managed-cluster --principal-context principal --principal-namespace argocd --agent-context managed-cluster --agent-namespace argocd --upsert

kustomize build managed-cluster/agent/base | envsubst | oc apply -f -

Wait a short time and then check the logs for any errors

oc logs deploy/argocd-agent-agent -n argocd

Validate that the application project is created successfully

oc get appproject -n argocd

### Deploy the test application

oc config use-context principal

oc apply -f control-plane/apps/managed-cluster/bgd/application.yaml -n managed-cluster

oc get apps -n managed-cluster

### Get the ArgoCD web UI details on the principal

oc get route/argocd-server -n argocd -o jsonpath='{"https://"}{.spec.host}{"\n"}'

oc get secret/argocd-cluster -n argocd -o jsonpath='{.data.admin\.password}' | base64 -d 
echo ""
