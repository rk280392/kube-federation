This repo can be used to learn and setup k8s federation with minikube and kubefed.

# Version Information:

kubectl:
   Client Version: v1.24.3
   Kustomize Version: v4.5.4
   Server Version: v1.24.3

minikube: v1.26.1

kubefed: 0.10.0

kubefedctl: v0.10.0 prerelease

# Steps

  ### 1- Create Clusters
      Create multiple clusters, one for master and others for workers. I used minikube on local system to set this up with profiles master, cluster1, cluster2

      - for profile in master cluster1 cluster2; do minikube start -p $profile; done

  ### 2- Setup Kubefed on a master cluster 

      - kubectl config use-context master
      - helm repo add kubefed-charts https://raw.githubusercontent.com/kubernetes-sigs/kubefed/master/charts
      - helm repo list
      - helm search repo kubefed, check the version and use it in next step
      - helm --namespace kube-federation-system upgrade -i kubefed kubefed-charts/kubefed --version=<x.x.x> --create-namespace
      - check the installation using: kubectl get pods -n kube-federation-system -p master

  ### 3- Cluster Registration

   To enable multi cluster networking through minikube, we will need to modify iptable rules. Please refer this https://docs.docker.com/network/iptables

   #### Rules:

      - iptables -I DOCKER-USER -i br-bb97104b4d71 -o br-8e2ede86a2b7 -j ACCEPT
      - iptables -I DOCKER-USER -i br-bb97104b4d71 -o br-2897dc7639e0 -j ACCEPT
      - iptables -I DOCKER-USER -i br-2897dc7639e0 -o br-8e2ede86a2b7  -j ACCEPT
      - iptables -I DOCKER-USER -i br-2897dc7639e0 -o br-bb97104b4d71  -j ACCEPT
      - iptables -I DOCKER-USER -i br-8e2ede86a2b7 -o br-bb97104b4d71  -j ACCEPT
      - iptables -I DOCKER-USER -i br-8e2ede86a2b7 -o br-2897dc7639e0  -j ACCEPT

  **br-bb97104b4d71 br-2897dc7639e0 br-8e2ede86a2b7** being my three bridge interfaces for each cluster, change according to your interfaces. You can get your bridge interfaces using **ifconfig, ip -a or brctl show** commands. You might want to add one single rule to accept all traffic, but I decided to add individual rules. 

   Download kubefedctl utility to interact with multiple clusters:

      - check the latest release of kubefedctl on https://github.com/kubernetes-sigs/kubefed/releases and download.

      - wget https://github.com/kubernetes-sigs/kubefed/releases/download/v0.10.0/kubefedctl-0.10.0-linux-amd64.tgz
      - tar xvzf kubefedctl-0.10.0-linux-amd64.tgz
      - chmod +x kubefedctl; mv kubefedctl /usr/local/bin
      - kubefedctl version

   Join the cluster using kubefedctl utility

      - kubefedctl join master --cluster-context master --host-cluster-context master --v=2
      - kubefedctl join cluster1 --cluster-context cluster1 --host-cluster-context master --v=2
      - kubefedctl join cluster2 --cluster-context cluster2 --host-cluster-context master --v=2

   Checking status of joined clusters:

      - kubectl -n kube-federation-system get kubefedclusters

      This should show all the cluster in **Ready=True** state. If not check above steps and troubleshoot.

  ### 4 - Test the federation cluster:
      - create cluster in master cluster:  `kubectl create ns testing --context=master`
      - federate the namsespace using kubefedctl: `kubefedctl federate ns testing`
      - check if it is replicated in other clusters:  `kubectl get ns --context=cluster1/cluster2`


# NEUVECTOR in federation mode

I went ahead and installed neuvector on my servers. You don't need to setup k8s federation cluster to set Neuvector in federation mode.
Refer - https://open-docs.neuvector.com/deploying/kubernetes

## Steps:

   ### 1- On master:

      - kubectl config use-context master
      - kubectl create namespace neuvector
      - kubectl apply -f https://raw.githubusercontent.com/neuvector/manifests/main/kubernetes/5.0.0/crd-k8s-1.19.yaml
      - kubectl apply -f https://raw.githubusercontent.com/neuvector/manifests/main/kubernetes/5.0.0/waf-crd-k8s-1.19.yaml
      - kubectl apply -f https://raw.githubusercontent.com/neuvector/manifests/main/kubernetes/5.0.0/dlp-crd-k8s-1.19.yaml
      - kubectl apply -f https://raw.githubusercontent.com/neuvector/manifests/main/kubernetes/5.0.0/admission-crd-k8s-1.19.yaml
      - kubectl create clusterrole neuvector-binding-app --verb=get,list,watch,update --resource=nodes,pods,services,namespaces
      - kubectl create clusterrole neuvector-binding-rbac --verb=get,list,watch --resource=rolebindings.rbac.authorization.k8s.io,roles.rbac.authorization.k8s.io,clusterrolebindings.rbac.authorization.k8s.io,clusterroles.rbac.authorization.k8s.io
      - kubectl create clusterrolebinding neuvector-binding-app --clusterrole=neuvector-binding-app --serviceaccount=neuvector:default
      - kubectl create clusterrolebinding neuvector-binding-rbac --clusterrole=neuvector-binding-rbac --serviceaccount=neuvector:default
      - kubectl create clusterrole neuvector-binding-admission --verb=get,list,watch,create,update,delete --resource=validatingwebhookconfigurations,mutatingwebhookconfigurations
      - kubectl create clusterrolebinding neuvector-binding-admission --clusterrole=neuvector-binding-admission --serviceaccount=neuvector:default
      - kubectl create clusterrole neuvector-binding-customresourcedefinition --verb=watch,create,get,update --resource=customresourcedefinitions
      - kubectl create clusterrolebinding  neuvector-binding-customresourcedefinition --clusterrole=neuvector-binding-customresourcedefinition --serviceaccount=neuvector:default
      - kubectl create clusterrole neuvector-binding-nvsecurityrules --verb=list,delete --resource=nvsecurityrules,nvclustersecurityrules
      - kubectl create clusterrolebinding neuvector-binding-nvsecurityrules --clusterrole=neuvector-binding-nvsecurityrules --serviceaccount=neuvector:default
      - kubectl create clusterrolebinding neuvector-binding-view --clusterrole=view --serviceaccount=neuvector:default
      - kubectl create rolebinding neuvector-admin --clusterrole=admin --serviceaccount=neuvector:default -n neuvector
      - kubectl create clusterrole neuvector-binding-nvwafsecurityrules --verb=list,delete --resource=nvwafsecurityrules
      - kubectl create clusterrolebinding neuvector-binding-nvwafsecurityrules --clusterrole=neuvector-binding-nvwafsecurityrules --serviceaccount=neuvector:default
      - kubectl create clusterrole neuvector-binding-nvadmissioncontrolsecurityrules --verb=list,delete --resource=nvadmissioncontrolsecurityrules
      - kubectl create clusterrolebinding neuvector-binding-nvadmissioncontrolsecurityrules --clusterrole=neuvector-binding-nvadmissioncontrolsecurityrules --serviceaccount=neuvector:default
      - kubectl create clusterrole neuvector-binding-nvdlpsecurityrules --verb=list,delete --resource=nvdlpsecurityrules
      - kubectl create clusterrolebinding neuvector-binding-nvdlpsecurityrules --clusterrole=neuvector-binding-nvdlpsecurityrules --serviceaccount=neuvector:default
      - kubectl get clusterrolebinding  | grep neuvector
      - kubectl get rolebinding -n neuvector | grep neuvector
      - kubectl apply -f neuvector-docker-k8s.yaml
      - kubectl apply -f neuvector-master.yaml

  ### 2- On Workers:

   Follow all the steps of master except last one. Instead of `neuvector-master.yaml` use `neuvector-worker.yaml` here.

      - kubectl apply -f neuvector-worker.yaml

  ### 3- Configuration:

   Get the Node IP and port of service:

      - minikube service neuvector-service-controller-fed-master -n neuvector -p master
      - minikube service neuvector-service-webui -n neuvector -p master

 Log in to neuvector dashboard using node ip and port of service `neuvector-service-webui`. Click on admin profile and top right and click on multiple clusters and click on promote. Add name, ip and port of `service neuvector-service-controller-fed-master`.

 Click on generate token from action menu. Copy the token and save.

 Do the same on worker nodes for worker services:

      - minikube service neuvector-service-controller-fed-worker -n neuvector -p cluster1/cluster2
      - minikube service neuvector-service-webui -n neuvector -p cluster1/cluster2

 Log in to neuvector dashboard using node ip and port of service `neuvector-service-webui`. Click on admin profile and top right and click on multiple clusters. Click on join and add name, ip and port of service `neuvector-service-controller-fed-worker`. Paste the token copied from master, primary ip and port would automatically get filled, save and that's it. 

**Congratulations, we have successfully learnt k8s federation and also deployed neuvector in federation mode**
