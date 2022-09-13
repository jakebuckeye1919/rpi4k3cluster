Docker-Registry TLS
Note

This guide for installing a Docker registry with TLS, enabling HTTPS, with self-signed certificate. This is mainly required for OpenFaaS.

This guide in general is a copy of the non TLS version with some tweaks.

Namespace
I will install everything related to Docker-registry into its own namespace called docker-registry. So, we will create that first:


kubectl create namespace docker-registry
Storage
Since we are going to store docker images in our personal registry to be later used with OpenFaaS, it would be a shame if they disappeared every time the pod reschedules to another node.

We need persistent storage that follows our pod around and provides it with the same data all the time.

If you followed my setup, you should have longhorn installed already.

PersistentVolumeClaim
A PersistentVolumeClaim volume is used to mount a PersistentVolume into a Pod. PersistentVolumeClaims are a way for users to "claim" durable storage (such as a GCE PersistentDisk or an iSCSI volume) without knowing the details of the particular cloud environment.

We will create new folder called docker-registry and a new file pvc.yaml in it.


cd
mkdir docker-registry
cd  docker-registry
nano pvc.yaml
In our pvc.yaml:


apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: longhorn-docker-registry-pvc
  namespace: docker-registry
spec:
  accessModes:
    - ReadWriteOnce
  storageClassName: longhorn
  resources:
    requests:
      storage: 15Gi
We are telling Kubernetes to use Longhorn as our storage class, and to claim/create 15 GB disk space for persistent storage. We will call it longhorn-docker-registry-pvc, and we will reference it by this name later.

Important

Notice I have specified namespace. This is important, since only pods/deployment in that namespace would be able to see the disk.

To learn more about volumes check out the official documentation here: https://kubernetes.io/docs/concepts/storage/persistent-volumes/

Apply our pvc.yaml:


kubectl apply -f pvc.yaml
And check


root@control01:/home/ubuntu/docker-registry# kubectl get pvc -n docker-registry
NAME                           STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS   AGE
longhorn-docker-registry-pvc   Bound    pvc-39662498-535a-4abd-9153-1c8dfa74749b   15Gi       RWO            longhorn       5d6h

#longhorn should also create automatically PV ( physical volume )
root@control01:/home/ubuntu/docker-registry# kubectl get pv -n docker-registry
NAME                                STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS   AGE
longhorn-docker-registry-pvc        Bound    pvc-700169df-2996-4031-bef5-e4cb7a560264   15Gi       RWO            longhorn       10d
Cool, cool: now we have storage! (Status might be different, something like: Unattached)

Creating certificates for docker registry
I literally spent over 24 hours to get this to work with the setup I have, including OpenFaaS and so on... so hopefully I did not forget a step :smile:

Generate certificates in your docker-register directory:


#install opnessl if its not
sudo apt-get install openssl

# generate certificate and key
openssl req -x509 -newkey rsa:4096 -sha256 -days 3650 -nodes -keyout registry.key -out registry.crt -subj "/CN=registry.cube.local" -addext "subjectAltName=DNS:registry.cube.local,DNS:*.cube.local,IP:192.168.0.201"
Important

This is very important: my entry in /etc/hosts for the registry will be 192.168.0.202 registry registry.cube.local. I know what IP it will be, as we will set it later (remember metalLB?), and there is no DNS server, so every node will have to have this in /etc/hosts. Call it whatever you want, but add the correct names into subjectAltName parameters. Without it, there might be issues where some tools complain about incorrectly signed certificates and missing SAN. Something like: x509: cannot validate certificate for <IP> because it doesn't contain any IP SANs

Now you have two new files in your docker-registry directory:


root@control01:~/docker-registry# ls | grep regis
registry.crt
registry.key
Adding TLS to the Kubernetes secret
Before I had separate disk in Longhorn for the certificates and had to copy the certificates there and then have it attach to the pod. That worked fine, but there is more easy solution. Kubernetes secret is a way to store secrets in the cluster.

Note

Individual secrets are limited to 1MiB in size. Having too many secrets in the cluster can cause performance issues.

Create a secret in your docker-registry namespace from the registry.crt and registry.key files:


kubectl create secret tls docker-registry-tls-cert -n docker-registry --cert=registry.crt --key=registry.key
Info

We are creating special secret type tls. This is a special type of secret that is used to store TLS certificates. More about secrets is here: https://kubernetes.io/docs/concepts/configuration/secret/

In essence think of it as internal storage for secrets in the cluster. These are then put as files inside the pod. You will see them in deployment files.

If you want to look at your secrets, you can do it like this:


kubectl get secret docker-registry-tls-cert -o yaml  -n docker-registry
You can also save it to a file from that output, and can apply it the same as any yaml file you have. Just remove these from metadata:


metadata:
  creationTimestamp: "2022-05-25T18:26:52Z"  <-- REMOVE
  name: docker-registry-tls-cert
  namespace: docker-registry  <-- REMOVE
  resourceVersion: "55302"  <-- REMOVE
  uid: 35208da3-19f1-4259-bfd1-5b276bb5948c  <-- REMOVE
Then you can have secret.yaml file and can store it in git or whatever. No need to manually create the secret. This also help if you use it later on with Argo CD gitops deployment.

Deployment
Now we will create a simple deployment of Docker registry and let it loose on our Kubernetes cluster.

Create a file in your Docker registry directory called docker.yaml:


---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: registry
  namespace: docker-registry
spec:
  replicas: 1
  selector:
    matchLabels:
      app: registry
  template:
    metadata:
      labels:
        app: registry
        name: registry
    spec:
      nodeSelector:
        node-type: worker
      containers:
      - name: registry
        image: registry:2
        ports:
        - containerPort: 5000
        env:
        - name: REGISTRY_HTTP_TLS_CERTIFICATE
          value: "/certs/tls.crt"
        - name: REGISTRY_HTTP_TLS_KEY
          value: "/certs/tls.key"
        volumeMounts:
        - name: lv-storage
          mountPath: /var/lib/registry
          subPath: registry
        - name: certs
          mountPath: /certs
      volumes:
        - name: lv-storage
          persistentVolumeClaim:
            claimName: longhorn-docker-registry-pvc
        - name: certs
          secret:
            secretName: docker-registry-tls-cert
What to pay attention to:

namespace - I specified docker-registry.
replicas - I'm using 1, so there will be only one docker registry running.
nodeSelector - As mentioned before in setting up my Kubernetes, I have labeled worker nodes with node-type=worker. This will make it so that the deployment prefers those nodes.
image - This will tell Kubernetes to download registry:2 from the official Docker hub.
containerPort - Which port the container will expose/use.
volumeMounts - Definition of where in the pod we will mount our persistent storage.
volumes - Definition where we refer back to PVC we created before.
env - This will be passed as environmental variables into the container, and used by docker registry.
volumeMounts
From bottom, you can see we are mounting our persistent storage, but also we are mounting our certificates. We name our volumes lv-storage and certs. Please note that the certs volume is a secret, and not a persistent volume claim.

volumes
In this section we tell the POD to mount our longhorn-docker-registry-pvc to /var/lib/registry. As here normally live images you upload to the registry. We also mount our certificates to /certs.

env
Docker registry have options you can influence by setting environment variables. In this case, we are setting the environment variables for the registry to use our certificates and their locations. So in /certs/tls.crt and /certs/tls.key.

Apply the deployment and wait a little for everything to come online.


kubectl apply -f docker.yaml
Check with:


# Deployment
root@control01:/home/ubuntu/docker-registry# kubectl get deployments -n docker-registry
NAME       READY   UP-TO-DATE   AVAILABLE   AGE
registry   1/1     1            1           21s
# Pods ( should be 1 )
root@control01:/home/ubuntu/docker-registry# kubectl get pods -n docker-registry
NAME                       READY   STATUS    RESTARTS   AGE
registry-6fdc5fc5d-npslq   1/1     Running   0          29s
We are not done yet. We also need to create a service to make the registry available cluster-wide, and ideally on the same IP/name all the time no matter what node it runs on.

Service
Again, if you followed my network setting we have set up metalLB to provide us with external IPs for pods. Therefore, we use this as a LoadBalancer service for our Docker-registry.

In your folder docker-registry, create service.yaml and paste in the following:


apiVersion: v1
kind: Service
metadata:
  name: registry-service
  namespace: docker-registry
spec:
  selector:
    app: registry
  type: LoadBalancer
  ports:
    - name: docker-port
      protocol: TCP
      port: 5000
      targetPort: 5000
  loadBalancerIP: 192.168.0.202
What to pay attention to:

kind - Service, just to let Kubernetes know what we are creating.
name - Just a name for our service.
namespace - I specified docker-registry, because the deployment we are targeting is in that name space.
selector and app - The value for this is lifted from our deployment where this is set : app: registry.
type - Here, we tell Kubernetes that we want LoadBalancer (MetalLB).
ports - We define port on our external IP and targetPort (that’s the port inside the app/container).
loadBalancerIP - This is optional, but I have included it here. This will allow us to specify which IP we want for the external IP. If you remove that line, MetalLB will assign the next free IP from the pool we allocated to it.
Apply the service:


kubectl apply -f service.yaml
Give it a few seconds to get the IP and check:


root@control01:/home/ubuntu/docker-registry# kubectl get svc -n docker-registry
NAME               TYPE           CLUSTER-IP   EXTERNAL-IP     PORT(S)          AGE
registry-service   LoadBalancer   10.43.5.16   192.168.0.202   5000:32096/TCP   7m48s
Fantastic! The service seems to be up and running with external port 5000. About the 32096 port behind it, this might be different for you. It is assigned to a node where the pod is running. In essence, it’s like this: External IP:5000 -> Node where the Pod/Container is:32096 -> container inside:5000. I hope that make sense :smile:

To get more info about the service, we can ask Kubectl to describe it to us:


root@control01:/home/ubuntu/docker-registry# kubectl get svc -n docker-registry
NAME               TYPE           CLUSTER-IP   EXTERNAL-IP     PORT(S)          AGE
registry-service   LoadBalancer   10.43.5.16   192.168.0.202   5000:32096/TCP   7m48s
root@control01:/home/ubuntu/docker-registry# kubectl describe svc registry-service  -n docker-registry
Name:                     registry-service
Namespace:                docker-registry
Labels:                   <none>
Annotations:              <none>
Selector:                 app=registry
Type:                     LoadBalancer
IP:                       10.43.5.16
IP:                       192.168.0.202
LoadBalancer Ingress:     192.168.0.202
Port:                     docker-port  5000/TCP
TargetPort:               5000/TCP
NodePort:                 docker-port  32096/TCP
Endpoints:                10.42.8.13:5000
Session Affinity:         None
External Traffic Policy:  Cluster
Events:
  Type    Reason        Age                  From                Message
  ----    ------        ----                 ----                -------
  Normal  IPAllocated   77s (x537 over 11m)  metallb-controller  Assigned IP "192.168.0.202"
  Normal  nodeAssigned  76s (x539 over 11m)  metallb-speaker     announcing from node "cube06"
Add root certificate to nodes
Currently, we have SSL or TLS (Fuck it, I will now call it HTTPS) enabled docker registry running on Kubernetes. However, since we create the certificate we need to add it as a root certificate to our nodes, every single one! If we don’t, any service that try to use it will complain about a certificate signed by an unknown authority or something similar. So, we will make ourselves the authority like this:


#For ubuntu
ansible cube -b -m copy -a "src=registry.crt dest=/usr/local/share/ca-certificates/registry.crt"
ansible cube -b -m copy -a "src=registry.key dest=/usr/local/share/ca-certificates/registry.key"
ansible all -b -m shell -a "update-ca-certificates"
In essence, on every node you need to copy our registry.crt into /usr/local/share/ca-certificates/, and execute update-ca-certificates, which will add our certificate as a root certificate. This will fool the verification into thinking that we are the authority for the certificate (which we are) and it won’t complain.

An example of doing it manually:


root@control01:~ sudo cp registry.* /usr/local/share/ca-certificates/
root@control01:~ sudo update-ca-certificates
Updating certificates in /etc/ssl/certs...
1 added, 0 removed; done.
Running hooks in /etc/ca-certificates/update.d...
done.
Making K3s use private docker registry
I know, I know. This is taking forever.

Here is where I got my info from: https://rancher.com/docs/k3s/latest/en/installation/private-registry/

Add a dns name to /etc/hosts on every node, I named it like this:


192.168.0.202 registry registry.cube.local
A good idea is to have the /etc/hosts nice and synced between all nodes, so I will add it once into control01 node, and move it to all nodes using Ansible.


echo "192.168.0.202 registry registry.cube.local" >> /etc/hosts
ansible cube -b -m copy -a "src=/etc/hosts dest=/etc/hosts"
Now, tell k3s about it. As root, create file /etc/rancher/k3s/registries.yaml:


nano /etc/rancher/k3s/registries.yaml
Add the following:


mirrors:
  registry.cube.local:5000:
    endpoint:
      - "https://registry.cube.local:5000"
configs:
  registry.cube.local:
    tls:
      ca_file: "/usr/local/share/ca-certificates/registry.crt"
      key_file: "/usr/local/share/ca-certificates/registry.key"
Send it to every control node of the cluster.


# Make sure the directory exists
ansible cube -b -m file -a "path=/etc/rancher/k3s state=directory"

# Copy the file
ansible cube -b -m copy -a "src=/etc/rancher/k3s/registries.yaml dest=/etc/rancher/k3s/registries.yaml"
Docker registry test
Follow the guide how to install docker from here:

We will download an Ubuntu container from the official Docker registry, re-tag it and push to our registry.


root@control01:~# docker pull ubuntu:16.04
16.04: Pulling from library/ubuntu
3e30c5e4609a: Pull complete
be82da0c7e99: Pull complete
bdf04dffef88: Pull complete
2624f7934929: Pull complete
Digest: sha256:3355b6e4ba1b12071ba5fe9742042a2f10b257c908fbdfac81912a16eb463879
Status: Downloaded newer image for ubuntu:16.04
docker.io/library/ubuntu:16.04

root@control01:~# docker tag ubuntu:16.04 registry.cube.local:5000/my-ubuntu
root@control01:~# docker push registry.cube.local:5000/my-ubuntu
The push refers to repository [registry.cube.local:5000/my-ubuntu]
3660514ed6c6: Pushed
2f33c1b8271f: Pushed
753fcdb98fb4: Pushed
1632f6712b3f: Pushed
latest: digest: sha256:2e459e7ec895eb5f94d267fb33ff4d881699dcd6287f27d79df515573cd83d0b size: 1150

# Check with curl
root@control01:~# curl https://registry.cube.local:5000/v2/_catalog
{"repositories":["my-ubuntu"]}
Yay ! It worked

Hopefully, this is it; congratulations getting this far. Now, get some coffee and maybe get me one too 🙂
