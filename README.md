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

      $ for profile in master cluster1 cluster2; do minikube start -p $profile; done

  ### 2- Setup Kubefed on a master cluster 

      $ kubectl config use-context master
      $ helm repo add kubefed-charts https://raw.githubusercontent.com/kubernetes-sigs/kubefed/master/charts
      $ helm repo list
      $ helm search repo kubefed, check the version and use it in next step
      $ helm --namespace kube-federation-system upgrade -i kubefed kubefed-charts/kubefed --version=<x.x.x> --create-namespace

check the installation using: 

      $ kubectl get pods -n kube-federation-system -p master

  ### 3- Cluster Registration

   To enable multi cluster networking through minikube, we will need to modify iptable rules. Please refer this https://docs.docker.com/network/iptables

   #### Rules:

     $ iptables -I DOCKER-USER -i br-bb97104b4d71 -o br-8e2ede86a2b7 -j ACCEPT
     $ iptables -I DOCKER-USER -i br-bb97104b4d71 -o br-2897dc7639e0 -j ACCEPT
     $ iptables -I DOCKER-USER -i br-2897dc7639e0 -o br-8e2ede86a2b7  -j ACCEPT
     $ iptables -I DOCKER-USER -i br-2897dc7639e0 -o br-bb97104b4d71  -j ACCEPT
     $ iptables -I DOCKER-USER -i br-8e2ede86a2b7 -o br-bb97104b4d71  -j ACCEPT
     $ iptables -I DOCKER-USER -i br-8e2ede86a2b7 -o br-2897dc7639e0  -j ACCEPT

  **br-bb97104b4d71 br-2897dc7639e0 br-8e2ede86a2b7** being my three bridge interfaces for each cluster, change according to your interfaces. You can get your bridge interfaces using **ifconfig, ip -a or brctl show** commands. You might want to add one single rule to accept all traffic, but I decided to add individual rules. 

   Download kubefedctl utility to interact with multiple clusters:

   Check the latest release of kubefedctl on https://github.com/kubernetes-sigs/kubefed/releases and download.

      $ wget https://github.com/kubernetes-sigs/kubefed/releases/download/v0.10.0/kubefedctl-0.10.0-linux-amd64.tgz
      $ tar xvzf kubefedctl-0.10.0-linux-amd64.tgz
      $ chmod +x kubefedctl; mv kubefedctl /usr/local/bin
      $ kubefedctl version

   Join the cluster using kubefedctl utility

      $ kubefedctl join master --cluster-context master --host-cluster-context master --v=2
      $ kubefedctl join cluster1 --cluster-context cluster1 --host-cluster-context master --v=2
      $ kubefedctl join cluster2 --cluster-context cluster2 --host-cluster-context master --v=2

   Checking status of joined clusters:

      $ kubectl -n kube-federation-system get kubefedclusters

      This should show all the cluster in **Ready=True** state. If not check above steps and troubleshoot.

  ### 4 - Test the federation cluster:

       create cluster in master cluster:  

         $ kubectl create ns testing --context=master

      federate the namsespace using kubefedctl: 

         $kubefedctl federate ns testing

      check if it is replicated in other clusters:  

         $ kubectl get ns --context=cluster1/cluster2
