Kubernetes Monitoring Using Prometheus & Grafana🛠️
![image](https://github.com/user-attachments/assets/a16d22b5-2726-4f2e-a4a6-c590b7a9390d)
## Introduction
the dynamic world of containerized applications and microservices, monitoring is indispensable for maintaining the health, performance, and reliability of your infrastructure. Kubernetes, with its ability to orchestrate containers at scale, introduces new challenges and complexities in monitoring. This is where tools like Prometheus and Grafana come into play.

Prometheus is an open-source systems monitoring and alerting toolkit originally built at SoundCloud. It excels at monitoring metrics and providing powerful query capabilities against time-series data. Meanwhile, Grafana complements Prometheus by offering visualization capabilities through customizable dashboards and graphs.

ollect and store metrics from your Kubernetes cluster and applications.
Visualize these metrics through intuitive dashboards.
Set up alerts based on predefined thresholds or anomalies.
Gain insights into the performance and resource utilization of your cluster.

## Prerequisites
Before we get started, ensure you have the following:
A running Kubernetes cluster.
kubectl command-line tool configured to communicate with your cluster.
Helm (the package manager for Kubernetes) installed.

## Setting up Prometheus and Grafana
Step 1: Adding the Helm Repository
First, add the Prometheus community Helm repository and update it:

      helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
      helm repo update
## Step 2: Installing Prometheus and Grafana

    # custom-values.yaml
    prometheus:
      service:
        type: NodePort
    grafana:
      service:
        type: NodePort


## Then, install the kube-prometheus-stack using Helm:

    helm upgrade --install kube-prometheus-stack prometheus-community/kube-prometheus-stack -f custom-values.yaml

## Step 3: Verifying the Installation
    kubectl get services
# Step 4: Accessing Prometheus and Grafana
    kubectl get nodes -o wide
Use the external IP of any node and the NodePorts to access the dashboards:

Prometheus: http://192.168.2.9:30090
Grafana: http://192.1682.9:31519


# Username:
    kubectl get secret --namespace default kube-prometheus-stack-grafana -o jsonpath="{.data.admin-user}" | base64 --decode ; echo

Password:

    kubectl get secret --namespace default kube-prometheus-stack-grafana -o jsonpath="{.data.admin-password}" | base64 --decode ; echo


![image](https://github.com/user-attachments/assets/67359c53-fc52-4ca8-9807-5afe4420eafa)
