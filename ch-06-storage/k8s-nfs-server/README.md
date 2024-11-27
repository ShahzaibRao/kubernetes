# NFS-server-for-kubernetes-cluster
Dynamic NFS storage provisioning in Kubernetes streamlines the creation and management of NFS volumes for your Kubernetes applications. It eliminates the need for manual intervention or pre-provisioned storage. The NFS provisioner dynamically creates persistent volumes (PVs) and associates them with persistent volume claims (PVCs), making the process more efficient. If you have an external NFS share and want to use it in a pod or deployment, the nfs-subdir-external-provisioner provides a solution for effortlessly setting up a storage class to automate the management of your persistent volumes.





# Prerequisites

1. Pre-installed Kubernetes Cluster
2. Root access to all nodes(servers) 
	
	
	
# Step 1: Installing the NFS Server(NFS-Server)

Do this in a seperate VM and dedicate that VM to storage purpose of NFS only


	sudo apt-get update
	sudo apt-get install nfs-common nfs-kernel-server -y
	
	
# Step 2: Create a directory to export(NFS-Server)

	sudo mkdir -p /data/nfs
	sudo chown nobody:nogroup /data/nfs
	sudo chmod 2770 /data/nfs
	
# Step 3: Export directory and restart NFS service(NFS-Server)


	sudo nana /etc/exports
	# add this line 
	/nfs/kubedata   *(rw,sync,no_subtree_check,no_root_squash,no_all_squash,insecure)
	# save this and close the file
	
	sudo exportfs -av
	
	sudo systemctl restart nfs-server

	sudo systemctl status nfs-server

# Step 4: Install NFS client packages on Kubernetes Nodes(Msster and Worker both)

	sudo apt update
	sudo apt install nfs-common -y

# Step 5: Install and Configure NFS Client Provisioner(Master Node)
	
	#install helm if not present in the kubernetes cluster
	wget https://get.helm.sh/helm-v3.7.1-linux-amd64.tar.gz || curl -LO https://get.helm.sh/helm-v3.7.1-linux-amd64.tar.gz
	tar -zxvf helm-v3.7.1-linux-amd64.tar.gz
	sudo mv linux-amd64/helm /usr/local/bin/helm
	helm version

	helm repo add nfs-subdir-external-provisioner https://kubernetes-sigs.github.io/nfs-subdir-external-provisioner
	helm repo update
	
	kubectl create namespace nfs-server
	
	helm install nfs-subdir-external-provisioner \
	nfs-subdir-external-provisioner/nfs-subdir-external-provisioner -n nfs-server\
	--set nfs.server=<IP address of the NFS-Server> \
	--set nfs.path=/data/nfs \
	--set storageClass.onDelete=true
	
	# Check pods and storage classes:
	kubectl get pod -n nfs-server
	kubectl get sc


# Step 6: Verify the nfs working with kubernetes(Master Node)

Instead of manually creating PVs, you only need to create PVCs. The provisioner will handle the rest, including creating the PVs on the NFS server and binding them to the PVCs.

	vi nfs-deploy.yml

	---
	apiVersion: apps/v1
	kind: Deployment
	metadata:
	  name: nfs-test-deployment
	spec:
	  replicas: 1
	  selector:
		matchLabels:
		  app: nfs-test
	  template:
		metadata:
		  labels:
			app: nfs-test
		spec:
		  containers:
		  - name: nginx
			image: nginx
			volumeMounts:
			- name: nfs-pv
			  mountPath: /usr/share/nginx/html
		  volumes:
		  - name: nfs-pv
			persistentVolumeClaim:
			  claimName: nfs-pvc
	---
	apiVersion: v1
	kind: PersistentVolumeClaim
	metadata:
	  name: nfs-pvc
	spec:
	  accessModes:
		- ReadWriteMany
	  storageClassName: nfs-default-storage-class
	  resources:
		requests:
		  storage: 1Gi

	
Execute this command 

	kubectl apply -f nfs-deploy.yml

Then ssh to the nginx pod and verify the nfs

	kubectl exec -it nfs-test-deployment-77f6d8dbd6-7j56q -- /bin/bash

	df -kh
You will that all of the disk usage here and whatever change are done in the pods shared directory will be reflected in the nfs-server
