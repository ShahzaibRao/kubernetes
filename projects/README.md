# Exposing Kubernetes Apps to the Internet with Cloudflare Tunnel, Ingress Controller, and ExternalDNS
in this tutorial, I will demonstrate how to configure Cloudflare Tunnel, external-dns, and an Ingress Controller to work together. We will create a single tunnel to Cloudflare using cloudflared, route traffic from Cloudflare to an Ingress Controller, and use this tunnel to expose applications to the internet through an Ingress resource. Additionally, we will use external-dns to automatically create all necessary DNS records in Cloudflare. This guide is designed to assist users looking to expose their services to the internet via Cloudflare with a focus on simple and home deployments.
![image](https://github.com/user-attachments/assets/c5221781-2411-4225-b12d-70edbe7931c9)
## Why You Need a Domain
If you plan to expose your services to the internet, having your own domain is a good idea. Domains are affordable, with some costing less than $5 a year. Purchasing one via Cloudflare significantly improves your deployments and allows you to easily utilize their free Cloudflare Tunnels. If, like me, you love to do home labs, having your own domain is a must!
![image](https://github.com/user-attachments/assets/1985c545-9bd3-48eb-a50c-40291215ca48)
## What is Cloudflare Tunnel?
Cloudflare Tunnel provides a secure way to connect your resources to Cloudflare without needing a publicly routable IP address. Instead of sending traffic to an external IP, a lightweight daemon in your infrastructure (cloudflared) creates outbound-only connections to Cloudflare’s global network.

It’s also worth mentioning that Cloudflare Tunnels are free. The purpose behind this could be to attract users to adopt Cloudflare services in their professional environments or to improve their IPS through user engagement.
## Why Pipe the Tunnel to Ingress Instead of Directly to the App?
cloudflared pipes the traffic from the tunnel to the desired service. While you can configure cloudflared to direct traffic to each individual app, I find it much more convenient to route traffic through an Ingress Controller. This approach simplifies management and reduces complexity in your setup.

With a single tunnel piped to the Ingress Controller, you only need to update one resource(Ingress) to expose a new app, rather than updating multiple configurations. This method also simplifies GitOps. Additionally, I don’t see any significant security drawbacks to this approach.

If you pipe traffic directly to the app, you would either need to manage certificates at the app level or use plain HTTP. Both options are not so convenient and practical. In my approach, TLS is terminated at the Ingress, which is good for home and small deployments.
## Cloudflare free plan limitations
Does it have limitations? Indeed it does! However, these limitations are negligible for our scenario. I will mention specific limitations throughout the article as they become relevant.
## Prerequisites
Kubernetes cluster with an Ingress controller installed
Cloudflare account (free or paid)
kubectl and helm installed on your local machine
An owned domain name and DNS zone configured with Cloudflare

## My Lab
In this guide, I will have my kubernated  1 master and 2  worker node cluster , along with ingress-nginx, external-dns, and a Cloudflare Tunnel using my own domain, raoshahzaib.site, hosted on Cloudflare. Finally, I will install Bitnami WordPress in my cluster to simulate the app I want to expose and make it accessible on the internet.
## Prepare the lab

![image](https://github.com/user-attachments/assets/c18e593a-30fe-41d5-aec6-159c1ebd006f)
## Install the ingress controller (using nginx)

Since we are using a Cloudflare Tunnel, it is sufficient to keep nginx exposed with a ClusterIP. Run the following commands and wait until ingress-nginx becomes healthy:

    helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
    helm repo update
    helm upgrade -i ingress-nginx ingress-nginx/ingress-nginx \
        --namespace kube-system \
        --set controller.service.type=ClusterIP \
        --set controller.ingressClassResource.default=true \
        --wait
Done. Now, we have met all the prerequisite conditions.

## Let’s start
### Namespace
Let’s deploy external-dns and cloudflared in the cloudflare namespace.
    kubectl create namespace cloudflare
## Cloudflare Token
We need a token that will be used by both cloudflared and external-dns.
Navigate to [API tokens](https://dash.cloudflare.com/profile/api-tokens) in the Cloudflare portal.
![image](https://github.com/user-attachments/assets/2cadfffd-cf04-4dea-be85-72620cd9fd6e)
Set the following permissions to your token:
Set the following permissions to your token:

Use your apiKey and Cloudflare account email to create the secret. This secret will be used by both cloudflared and eternal-dns:

    kubectl create secret generic cloudflare-api-key \
      --from-literal=apiKey=<your-api-key> \
      --from-literal=email=<your-email> \
      --namespace=cloudflare

## Create the tunnel
Here is the [original guide](https://developers.cloudflare.com/cloudflare-one/tutorials/many-cfd-one-tunnel/). We will follow it partially.
First, install the `cloudflared` tool. Instructions can be found [here](https://developers.cloudflare.com/cloudflare-one/connections/connect-apps/install-and-setup/installation). I will use the binary method:

Then, log in to obtain the certificate. It will pop up the browser. You will need to log in using the browser:

    cloudflared tunnel login

Once you log in, on the next screen, you will need to authorize the creation of the tunnel for your zone.
![image](https://github.com/user-attachments/assets/595f2f19-9494-4f8c-a347-21fe37eb0bc0)

Next, create the tunnel. I will name the tunnel “demo-lab”. It will output the tunnel ID, which you should copy as we will need it later:

      rao@control-plane:~/downloads$ cloudflared tunnel create demo-lab
      Tunnel credentials written to /home/rao/.cloudflared/23024f29-ff79-480e-8cd3-8f1ace161421.json. cloudflared chose this file based on where your origin certificate was found. Keep this file secret. To revoke these credentials, delete the tunnel.
      
      Created tunnel demo-lab with id 23024f29-ff79-480e-8cd3-8f1ace161421
      
## Finally, let’s create the secret using the data from the previous output:
    kubectl create secret generic tunnel-credentials \
      --from-file=credentials.json=/home/rao/.cloudflared/23024f29-ff79-480e-8cd3-8f1ace161421.json \
      --namespace=cloudflare
    secret/tunnel-credentials created

The original guide asks us to create a CNAME to associate the tunnel with a DNS record. We will do this later using external-dns.

## Change Cloudflare TLS mode
Navigate to the “SSL/TLS” -> “Overview” dashboard in your domain, and change the mode from the default “Flexible” to “Full”.
![image](https://github.com/user-attachments/assets/905173c2-081d-4543-a202-0edf0f05848a)
![image](https://github.com/user-attachments/assets/8a609adb-f680-4d65-96d3-19ca8f42d592)
Navigate to “SSL/TLS” -> “Edge Certificates” and ensure that Cloudflare has issued the certificate for your domain. This certificate will be presented to end users trying to connect to your service.
![image](https://github.com/user-attachments/assets/dfd365a9-32f2-4bab-9afa-2352661de1af)

IMPORTANT NOTE: I wasted a few hours on this. The free plan issues edge certificates only for yourdomain.com and *.yourdomain.com. This means that if you try to expose your app under a subdomain, for example, myapp.infra.yourdomain.com, Cloudflare will not issue the certificate for the app. Under the free plan, using this method, you can expose your apps solely under the root domain, for example — myapp.yourdomain.com.

![image](https://github.com/user-attachments/assets/cf242194-d502-42d4-85dd-23265b44b976)

IMPORTANT NOTE 2: If your domain is not hosted on Cloudflare, you will need to upload your custom SSL Certificate here, which may require upgrading the plan. That’s why I recommend purchasing the domain through Cloudflare.

## Cloudflared deployment
    cat > tunne-values.yaml <<EOF
    cloudflare:
      tunnelName: "demo-lab"
      tunnelId: "23024f29-ff79-480e-8cd3-8f1ace161421"
      secretName: "tunnel-credentials"
      ingress:
        - hostname: "*.raoshahzaib.site"
          service: "https://ingress-nginx-controller.kube-system.svc.cluster.local:443"
          originRequest:
            noTLSVerify: true
    
    resources:
      limits:
        cpu: "100m"
        memory: "128Mi"
      requests:
        cpu: "100m"
        memory: "128Mi"
    
    replicaCount: 1
    EOF
----------------------------------------------------------------------
      helm repo add cloudflare https://cloudflare.github.io/helm-charts
      helm repo update
      helm upgrade --install cloudflare cloudflare/cloudflare-tunnel \
        --namespace cloudflare \
        --values tunne-values.yaml \
        --wait


## Config Explanation
cloudflare.tunnelName && cloudflare.tunnelId: Populate these fields using the information from the previous steps.
cloudflare.secretName: This is the secret name that we configured previously.
cloudflare.ingress: Configuration for the ingress rules.
— hostname: The hostname pattern that this ingress rule applies to (*.monkale.io means any subdomain of monkale.io).
— service: The service URL to route the traffic to. We will route all incoming traffic to the ingress. In this case, it is an nginx ingress controller deployed in the kube-system namespace:
https://ingress-nginx-controller.kube-system.svc.cluster.local:443
— originRequest.noTLSVerify: Disables TLS verification for the origin service (set to true). This is exactly the “TLS Full” mode scenario. Once the traffic arrives from the tunnel, it hits the nginx service, which possesses a default fake certificate valid for “ingress.local”. This would cause TLS issues, so we disable TLS verification.
replicaCount: By default, it installs 2 replicas. For small deployments, 1 replica will be enough.


## Review tunnel
Ensure that the cloudflared deployment is up and running:
      rao@control-plane:~/kubernetes/projects$ kubectl get deploy -n cloudflare cloudflare-cloudflare-tunnel
      NAME                           READY   UP-TO-DATE   AVAILABLE   AGE
      cloudflare-cloudflare-tunnel   1/1     1            1           95s
## Go to the Zero Trust dashboard in Cloudflare, and navigate to your domain -> Network -> Tunnels.

![image](https://github.com/user-attachments/assets/e5b44e2f-1eb2-4321-a3f0-72e29af960ab)

Ensure that your tunnel is “Healthy”.

If you experience problems during the configuration, you may want to see live logs. Click on “Connector ID” and then you will be able to stream live logs.
## Install external-dns
Simply copy and paste the following commands. The policy=sync setting ensures that external-dns is eligible to add and remove records as well. The source=ingress setting means that we will create records for our ingresses only (by default, it also creates records for services).

    helm repo add kubernetes-sigs https://kubernetes-sigs.github.io/external-dns/
    helm repo update
    helm upgrade --install external-dns kubernetes-sigs/external-dns \
      --namespace cloudflare \
      --set sources[0]=ingress \
      --set policy=sync \
      --set provider.name=cloudflare \
      --set env[0].name=CF_API_TOKEN \
      --set env[0].valueFrom.secretKeyRef.name=cloudflare-api-key \
      --set env[0].valueFrom.secretKeyRef.key=apiKey \
      --wait

Check if external-dns is up and running.

    kubectl get deploy -n cloudflare external-dns
    NAME           READY   UP-TO-DATE   AVAILABLE   AGE
    external-dns   1/1     1            1           64s

## Create Ingress resource
      rao@control-plane:~/kubernetes/projects$ kubectl apply -f .

    kubectl get all -n wordpress
    NAME                            READY   STATUS    RESTARTS   AGE
    pod/mariadb-7976995ffb-vhl4d    1/1     Running   0          8m58s
    pod/wordpress-c7d97577f-dq4n4   1/1     Running   0          8m58s

    NAME                TYPE           CLUSTER-IP     EXTERNAL-IP   PORT(S)        AGE
    service/mariadb     ClusterIP      10.97.213.53   <none>        3306/TCP       8m58s
    service/wordpress   LoadBalancer   10.96.16.9     <pending>     80:30805/TCP   8m58s

    NAME                        READY   UP-TO-DATE   AVAILABLE   AGE
    deployment.apps/mariadb     1/1     1            1           8m58s
    deployment.apps/wordpress   1/1     1            1           8m58s


![alt text](image.png)

## Now, let’s expose it to the Internet. The configuration is explained below.

apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  annotations:
    external-dns.alpha.kubernetes.io/cloudflare-proxied: "true"
    external-dns.alpha.kubernetes.io/hostname: blog.raoshahzaib.site
    external-dns.alpha.kubernetes.io/target: 23024f29-ff79-480e-8cd3-8f1ace161421.cfargotunnel.com
  name: demo-wordpress
  namespace: wordpress
spec:
  rules:
  - host: blog.raoshahzaib.site
    http:
      paths:
      - backend:
          service:
            name: demo-wordpress
            port:
              number: 80
        path: /
        pathType: Prefix


## Let’s review the Ingress section.
external-dns.alpha.kubernetes.io/hostname: Sets the hostname for the DNS record, in this case, wordpress.monkale.io.
external-dns.alpha.kubernetes.io/target: Points to the Cloudflare Tunnel endpoint, replace this id with your tunnel-id followed by .cfargotunnel.com.
external-dns.alpha.kubernetes.io/cloudflare-proxied: Indicates that Cloudflare should proxy the traffic, set to true.
hosts[0].host : Hostname that will be used to share the app.
## Review and test
As you can see external-dns has successfully created a few records.

![alt text](image-1.png)

Let’s test! Browse to blog.raoshahzaib.site and see if it works.
![alt text](image-2.png)


## Inspect the server certificate. It should be the default edge certificate provided by Cloudflare, issued by Google.
![alt text](image-3.png)

## Summary and Conclusion
We successfully set up the cloudflared tunnel and pointed it to the Ingress Controller. Then, we successfully applied Ingress rules and tested the connection. The whole process is extremely fast and can be easily automated using tools like ArgoCD and Flux.
