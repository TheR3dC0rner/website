---
title: "Helm Part I: Building our test deployment"
summary:  "Building a basic Kubernetes deployment to build our helm template around"
date: 2025-09-27
tags: ["deployment", "helm", "scripting"]
draft: false
--- 
In this entry I'm going to walk through my thought process on building a test deployments then use those to model our helm charts.  First let's go over what we are trying to accomplish.  

{{< mermaid >}}
flowchart TB
  EP[("C2 Redirector VPS")] -->|HTTPS| IC[/"Ingress Controller"/]
  IC --> IR["Ingress Resource
  (path rules)"]

  %% specific path first (higher priority)
  IR -->|"path = /c2traffic"| S_C2[Service: AdaptixC2]
  S_C2 --> P_C2[(Pods: AdaptixC2)]

  %% fallback / catch-all
  IR -->|"path = /"| S_MAIN[Service: Frontend Website]
  S_MAIN --> P_MAIN[(Pods: Nginx )]

  classDef svc fill:#f9f,stroke:#333,stroke-width:1px;
  class S_C2,S_MAIN svc;

{{< /mermaid >}}

**Diagram: a VPS redirector → Ingress Controller → path-based routing to AdaptixC2 or frontend.**

Based on this diagram, let's make a checklist of items we are going to need and which should go in helm charts and which should be external YAML.  I will be going over how to deploy our redirectors once we start building out our production cluster so for now this will not be covered.

Eventually we will need 2 helm charts, one for AdaptixC2 and one for our Nginx backend.  

AdaptixC2:

- base_deployment.yaml (main deployment file for the AdaptixC2 pods)
{{< alert >}}
  **Note**  The deployment will have to generate self signed certs to be able to serve up ssl and mount them
{{< /alert >}}
- operator-service.yaml (loadbalancer port exposed for operators to access)
- web-service.yaml (clusterIp for ingress controller)
- adaptixc2-profile-secret.yaml This will be the AdaptixC2 profile and this should be left out the helm chart file so when deploy it from helm we can encrypt it in our git archive.
- pvc.yaml (a persistent volume claim to store the AdaptixC2 data stored 
{{< alert >}}  
**Note** this will be mounted at  /home/adaptix/data
{{< /alert >}}
Now that we have a basic checklist of items we need to build a test deployment of AdaptixC2.  I will give a brief description each item before going through the YAML.  

## **base_deployment.yaml**

In Kubernetes a deployment handles a set of Pods to run an application workload.  It also controls the replication state of pods, including the strategy if the Pod has failed, in our case we want it to restart.  

A Pod in Kubernetes is a set of containers.  These containers could be continuously running services,  initialization tasks or helper services (more commonly known as sidecars)

```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: adaptix-deployment
  namespace: default
  labels:
    app: adaptixc2
    tier: ops
spec:
  replicas: 1 # Desired number of pod replicas
  selector:
    matchLabels:
      app: adaptixc2
      tier: ops
  template:
    metadata:
      labels:
        app: adaptixc2
        tier: ops
    spec:
      initContainers:
      - name: mkcert #uses mkcerts to create certificates for Adaptix C2 Operator Ports
        image: alpine/mkcert:latest
        volumeMounts:
        - mountPath: /mnt
          name: server-certs
        #command: ["/bin/sh","-c"]
        args:
          - -cert-file
          - /mnt/cert.pem
          - -key-file
          - /mnt/cert.key
          - adaptix
          - adaptix.internal
      - name: fixpermissions #fixes the file permissions to match our security context
        image: alpine:latest
        volumeMounts:
        - mountPath: /mnt
          name: server-certs
        command: ["/bin/sh"]
        args: ["-c", "chown 999:999 /mnt/cert.*"]

      containers:
      - name: adaptixc2 #main adaptixc2 container
        securityContext:
          runAsUser: 999
          runAsGroup: 999
        image: gitea.dev.th3redc0rner.com/redcorner/adaptixc2:2025-09-15 # Container image to use
        args:
          - -profile
          - /home/adaptix/profile.cfg
        volumeMounts:
        - name: adaptix-data
          mountPath: /home/adaptix/data #Adaptix C2 Data Directory
        - name: profile-secret
          mountPath: /home/adaptix/profile.cfg
          subPath: profile.cfg
        - name: server-certs
          mountPath: /home/adaptix/cert/

        ports:
        - name: httpport
          containerPort: 8080 # Port the container listens on
        - name: httpsport
          containerPort: 8443
        - name: operatorport
          containerPort: 4321
        - name: proxyport
          containerPort: 4444
        livenessProbe:
          tcpSocket:
            port: 4321
        readinessProbe:
          tcpSocket:
            port: 4321
        resources:
          limits:
            memory: "128Mi"
            cpu: "500m"
      volumes:
      - name: adaptix-data
        persistentVolumeClaim:
          claimName: adaptix-pvc
      - name: server-certs
        emptyDir: {} #mkcert will create the certs here
      - name: profile-secret
        secret:
          secretName: adaptix-profile
          items:
          - key: profile-data
            path: profile.cfg
```


To break this deployment down:

The deployment contains two initialization jobs.  The first uses 'mkcert' to create self signed certs and then the second  fixes the permissions on those certificates.  These certs will be used by the AdaptixC2 operator port .  

They are created in an 'emtpyDir'.  These cert are recreated every time the pod restarts.  

The AdaptixC2 container pulls the custom image we built and hosted on our internal Git instances.

It mounts the generated certificates to the directory:
`/home/adaptix/cert/`

It will then mount the profile as a stored secret to the file:
`/home/adaptix/profile.cfg`

Once we declare a persistent volume it will be mounted at:

`/home/adaptix/data`

The difference between the a PeristentVolume (PVC) and an 'emtpydir' is that if the pod restarts the data will be retained until the entire the deployment is deleted.  In our course we are storing our data in the our storage provider Longhorn.

The deployment also defines the following ports:

The operators will connect at port: 
- '4321'
The listeners will be hosted on either port
- '8080'
- '8443'
The socksproxy will be available on:
- '4444'

Finally we gave the pod some probes to make sure nothing has crashed.  

We check that port 4321 opens and stays open.  

If the port closes, the container is declared unhealthy and restarts.


The covers deployment file.  

## adaptix-profile-secret.yaml

The  'adaptix-profile-secret.yaml' contains our 'profile.json' file which we mount in the deployment as profile.cfg.  This is the clear text version.  We can turn into our stored secret by running the following the command:

```
 kubectl create secret generic adaptix-profile --from-file=profile-data=profile.json --dry-run=client -o yaml >adaptix-profile-secret.yaml
```


**profile.json

```
{
  "Teamserver": {
    "interface": "0.0.0.0",
    "port": 4321,
    "endpoint": "/endpoint",
    "password": "pass",
    "cert": "cert/cert.pem",
    "key": "cert/cert.key",
    "extenders": [
      "extenders/listener_beacon_http/config.json",
      "extenders/listener_beacon_smb/config.json",
      "extenders/listener_beacon_tcp/config.json",
      "extenders/agent_beacon/config.json",
      "extenders/listener_gopher_tcp/config.json",
      "extenders/agent_gopher/config.json"
    ],
    "access_token_live_hours": 12,
    "refresh_token_live_hours": 168
  },

  "ServerResponse": {
    "status": 404,
    "headers": {
      "Content-Type": "text/html; charset=UTF-8",
      "Server": "AdaptixC2",
      "Adaptix Version": "v0.8"
    },
    "page": "404page.html"
  },

  "EventCallback": {
    "Telegram": {
      "token": "",
      "chats_id": []
    },
    "new_agent_message": "New agent: %type% (%id%)\n\n%user% @ %computer% (%internalip%)\nelevated: %elevated%\nfrom: %externalip%\ndomain: %domain%",
    "new_cred_message": "New secret [%type%]:\n\n%username% : %password% (%domain%)\n\nStorage: %storage%\nHost: %host%",
    "new_download_message":"File saved: %path% [%size%] from %computer% (%user%)"
  }
}
```

This will output a file like this:

```
apiVersion: v1
data:
  profile-data: ewogICJUZWFtc2VydmVyIjogewogICAgImludGVyZmFjZSI6ICIwLjAuMC4wIiwKICAgICJwb3J0IjogNDMyMSwKICAgICJlbmRwb2ludCI6ICIvZW5kcG9pbnQiLAogICAgInBhc3N3b3JkIjogInBhc3MiLAogICAgImNlcnQiOiAiY2VydC9jZXJ0LnBlbSIsCiAgICAia2V5IjogImNlcnQvY2VydC5rZXkiLAogICAgImV4dGVuZGVycyI6IFsKICAgICAgImV4dGVuZGVycy9saXN0ZW5lcl9iZWFjb25faHR0cC9jb25maWcuanNvbiIsCiAgICAgICJleHRlbmRlcnMvbGlzdGVuZXJfYmVhY29uX3NtYi9jb25maWcuanNvbiIsCiAgICAgICJleHRlbmRlcnMvbGlzdGVuZXJfYmVhY29uX3RjcC9jb25maWcuanNvbiIsCiAgICAgICJleHRlbmRlcnMvYWdlbnRfYmVhY29uL2NvbmZpZy5qc29uIiwKICAgICAgImV4dGVuZGVycy9saXN0ZW5lcl9nb3BoZXJfdGNwL2NvbmZpZy5qc29uIiwKICAgICAgImV4dGVuZGVycy9hZ2VudF9nb3BoZXIvY29uZmlnLmpzb24iCiAgICBdLAogICAgImFjY2Vzc190b2tlbl9saXZlX2hvdXJzIjogMTIsCiAgICAicmVmcmVzaF90b2tlbl9saXZlX2hvdXJzIjogMTY4CiAgfSwKCiAgIlNlcnZlclJlc3BvbnNlIjogewogICAgInN0YXR1cyI6IDQwNCwKICAgICJoZWFkZXJzIjogewogICAgICAiQ29udGVudC1UeXBlIjogInRleHQvaHRtbDsgY2hhcnNldD1VVEYtOCIsCiAgICAgICJTZXJ2ZXIiOiAiQWRhcHRpeEMyIiwKICAgICAgIkFkYXB0aXggVmVyc2lvbiI6ICJ2MC44IgogICAgfSwKICAgICJwYWdlIjogIjQwNHBhZ2UuaHRtbCIKICB9LAoKICAiRXZlbnRDYWxsYmFjayI6IHsKICAgICJUZWxlZ3JhbSI6IHsKICAgICAgInRva2VuIjogIiIsCiAgICAgICJjaGF0c19pZCI6IFtdCiAgICB9LAogICAgIm5ld19hZ2VudF9tZXNzYWdlIjogIk5ldyBhZ2VudDogJXR5cGUlICglaWQlKVxuXG4ldXNlciUgQCAlY29tcHV0ZXIlICglaW50ZXJuYWxpcCUpXG5lbGV2YXRlZDogJWVsZXZhdGVkJVxuZnJvbTogJWV4dGVybmFsaXAlXG5kb21haW46ICVkb21haW4lIiwKICAgICJuZXdfY3JlZF9tZXNzYWdlIjogIk5ldyBzZWNyZXQgWyV0eXBlJV06XG5cbiV1c2VybmFtZSUgOiAlcGFzc3dvcmQlICglZG9tYWluJSlcblxuU3RvcmFnZTogJXN0b3JhZ2UlXG5Ib3N0OiAlaG9zdCUiLAogICAgIm5ld19kb3dubG9hZF9tZXNzYWdlIjoiRmlsZSBzYXZlZDogJXBhdGglIFslc2l6ZSVdIGZyb20gJWNvbXB1dGVyJSAoJXVzZXIlKSIKICB9Cn0K
kind: Secret
metadata:
  creationTimestamp: null
  name: adaptix-profile
  namespace: default
```

We will keep this out of our helm file since we will need to encrypt it but for smoking testing our deployment this fine.  

We are then going to create 2 service YAML files:

## **operator_service.yaml:

```
apiVersion: v1
kind: Service
metadata:
 name: opsoperatorservice
 labels:
  tier: ops
  app: adaptixc2
 namespace: default
spec:
  type: LoadBalancer
  ports:
    - protocol: TCP
      port: 4321
      targetPort: operatorport
      name: operator-port
    - protocol: TCP
      port: 4444
      targetPort: proxyport
      name: proxy-port
  selector:
   tier: ops
   app: adaptixc2
```

This opens up the ports Red Team operators on each node of the cluster.  The '4321' port is how the AdaptixC2 client connects to the AdaptixC2 server.  Port '4444' is the port the operators can direct SOCKS5 traffic through our victims.   When we build our helm charts we want to make these configurable.  

## ** adaptixweb_service.yaml

```
apiVersion: v1
kind: Service
metadata:
 name: opsadaptixwebservice
 labels:
  tier: ops
  app: adaptixc2
 namespace: default
spec:
  type: ClusterIP
  ports:
    - protocol: TCP
      port: 443
      targetPort: httpsport
  selector:
   tier: ops
   app: adaptixc2
```

This service will open up an HTTPS port to direct our external traffic to the internal listener on our AdaptixC2 server.  Since its a ClusterIP it won't be directly exposed to services outside our cluster but will be exposed to internal services.


## ** pvc.yaml

The last item will be our storage.  Currently we are giving each C2 server 2 gigabytes of internal storage of the C2 data.  This should be plenty.  
```
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: adaptix-pvc
  namespace: default
spec:
  accessModes:
    - ReadWriteOnce
  storageClassName: longhorn
  resources:
    requests:
      storage: 2Gi
```

## Updating our organization to public
Packages are only exposed without authentication for download to public users/organizations.  When I created this organization I had made it limited.  You can be changed by going to your organization in the upper right.



![Pasted Image](<images/Pasted image 20250926160942.png>)

Clicking Settings in the upper right:

![Pasted Image](<images/Pasted image 20250926161017.png>)

Change Visibility to Public:
![Pasted Image](<images/Pasted image 20250926161045.png>)

## Deploying our test 

To deploy these files ware going to run:
```
kubectl apply -f pvc.yaml
kubectl apply -f adaptix-profile-secret.yaml
kubectl apply -f base_deployment.yaml
kubectl apply -f adaptixweb_service.yaml
kubectl apply -f operator_service.yaml
```

If there are no errors you can check the progress by running:
```
kubectl get pods
```

Your output should look like this
```
NAME                                  READY   STATUS    RESTARTS   AGE
adaptix-deployment-7fb9fb7945-5pwlw   1/1     Running   0          16h

```

If you run kubectl get services your output should be something like this:

```
NAME                   TYPE           CLUSTER-IP       EXTERNAL-IP                                      PORT(S)                         AGE
kubernetes             ClusterIP      10.96.0.1        <none>                                           443/TCP                         47d
opsadaptixwebservice   ClusterIP      10.101.189.175   <none>                                           443/TCP                         4d17h
opsoperatorservice     LoadBalancer   10.97.154.179    192.168.200.186,192.168.201.175,192.168.201.41   4321:31050/TCP,4444:31029/TCP   6d7h
```

Now we can test whether we can reach the operator port.

## Testing our AdaptixC2 deployment

I have a WSL (Windows Subsystem for Linux) box with Kali already set up with the AdaptixC2 client. Using SSH local port forwarding I'm going to open the port locally on my machine and forward the traffic to our Kubernetes cluster.

```
ssh admin1@devadmin1.lan -L 0.0.0.0:4321.192.168.200.168:4321
```

Now I can launch the AdaptixClient
![Pasted Image](<images/Pasted image 20250924201453.png>)
If all is successful I should be able to complete the connection:
![Pasted Image](<images/Pasted image 20250924201552.png>)

It appears our deployment is fully working.  


Next blog we will perform the same process for making a deployable web front end for this C2.  


