# ArgoCD Agents configuration 

Following the blog post here : https://developers.redhat.com/blog/2025/10/06/using-argo-cd-agent-openshift-gitops

## Notes on deployment

Deploying to a cluster that already has the GitOps operator installed and has an instance of ArgoCD in the openshift-gitops namespace.

oc apply -k control-plane/namespaces/base

oc config current-context

oc config rename-context $(oc config current-context) principal

argocd-agentctl pki init --principal-context principal --principal-namespace argocd 