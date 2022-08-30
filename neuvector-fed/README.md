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
