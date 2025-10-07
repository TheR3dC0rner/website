---
title: "Helm Part II: Building our Nginx deployment"
summary:  "Building a basic Kubernetes deployment to build our helm template around"
date: 2025-10-06
tags: ["deployment", "helm", "scripting", "nginx"]
draft: false
---

In this entry we are going to focus on creating the deployment files for  our Nginx backend for hosting static websites to act as our frontend for our C5's. Like before let's make a checklist of what we want this to do:

- Nginx backend
- Nginx running not as root but in a security context of a user
- Sync the website from a Git repo rather than hosting it on the webserver”

## Creating the website repo

First, go to your organization

![Pasted image 20250926161131.png](images/Pasted%20image%2020250926161131.png)
Click New Repository

![Pasted image 20250926161215.png](images/Pasted%20image%2020250926161215.png)


Create the repository, keep it public


![Pasted image 20250926161322.png](images/Pasted%20image%2020250926161322.png)

Click on teams

![Pasted image 20250926161423.png](images/Pasted%20image%2020250926161423.png)

Give our Operators Permission by clicking View
![Pasted image 20250926161508.png](images/Pasted%20image%2020250926161508.png)
Then click repositories

![Pasted image 20250926161545.png](images/Pasted%20image%2020250926161545.png)
Then adding our new repository

![Pasted image 20250926161626.png](images/Pasted%20image%2020250926161626.png)

Now we are going to add some test files.

 On our jumpbox we are going to create this structure and push it to this repo:

```
mkdir hostedsites
mkdir hostedsites/www.example.com
echo "<HTML><H1>This is our test page</H1></HTML>" > hostedsites/www.example.com/index.html
cd hostedsites
git init
git checkout -b main
git add .
git commit -m "first commit"
git remote add origin git@gitssh.dev.th3redc0rner.com:RedCorner/hostedsites.git
git push -u origin main
 ```


## Building our base_deployment.yaml
This will build our Nginx and sidecar deployment.

This deployment should be a bit easier since the Nginx container is already  a common container but by default runs as root.  Using our deployment file and a new config we can deploy this as a container in user context.  We also want a GitOps-style workflow, that we can store then in a repository have the webserver fetch the static sites so we will add a side container of the another common container, git-sync.

Let's start again with our base_deployment.yaml:

```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-gitsync
  labels:
    app: nginx-gitsync
spec:
  replicas: 1
  selector:
    matchLabels:
      app: nginx-gitsync
  template:
    metadata:
      labels:
        app: nginx-gitsync
    spec:
      securityContext:
        runAsUser: 1000
        fsGroup: 1000
      containers:
        - name: nginx
          image: nginx:1.27-alpine
          ports:
            - containerPort: 8080
              name: httpport
          args: ["nginx", "-g", "daemon off;"]
          volumeMounts:
            - name: webroot
              mountPath: /usr/share/nginx/html
            - name: nginx-cache
              mountPath: /var/cache/nginx
            - name: nginx-run
              mountPath: /run
            - name: nginx-conf
              mountPath: /etc/nginx/nginx.conf
              subPath: nginx.conf
          securityContext:
            runAsUser: 1000
            runAsGroup: 1000
            runAsNonRoot: true

        - name: git-sync
          image: registry.k8s.io/git-sync/git-sync:v4.2.3
          volumeMounts:
            - name: webroot
              mountPath: /git
          env:
            - name: GITSYNC_REPO
              value: "https://gitea.dev.th3redc0rner.com/RedCorner/hostedsites"   # <-- update to your repo
            - name: GITSYNC_REF
              value: "main"
            - name: GITSYNC_ROOT
              value: "/git"
            - name: GITSYNC_LINK
              value: "website"
            - name: GITSYNC_PERIOD
              value: "30s"

      volumes:
        - name: webroot
          emptyDir: {}
        - name: nginx-cache
          emptyDir: {}
        - name: nginx-run
          emptyDir: {}
        - name: nginx-conf
          configMap:
            name: nginx-conf
            items:
              - key: nginx.conf
                path: nginx.conf
```

Lets' break down this deployment file.  It contains the Nginx webserver a side container for syncing the websites to an emptyDir.   

Currently it's pointed to our internal webhosting and the Nginx config can be modified to point to the root of the web frontend we want.


The Nginx config  is customized to for the root of nginx to be our custom domain name and to listen on port 8080.  

## configmap.yaml
This will represent the Nginx config file.  Additionally it will put our website domain as the root of the Nginx web server.

```

apiVersion: v1
kind: ConfigMap
metadata:
  name: nginx-conf
data:
  nginx.conf: |
    events {}
    http {
      server {
        listen 8080;
        root /usr/share/nginx/html/website/www.example.com;
        location / {
          index index.html;
        }
      }
    }

```


## nginxwebservice.yaml

The web service is places this is in a clusterIP so that it can be accessed by services only internal to our k8s cluster.


```
apiVersion: v1
kind: Service
metadata:
  name: nginx-gitsync
  labels:
    app: nginx-gitsync
spec:
  selector:
    app: nginx-gitsync
  ports:
    - name: http
      port: 80
      targetPort: httpport
  type: ClusterIP
```
## Deploying and initial test

Now let's deploy all these files one by one to make sure we have no errors:

```
kubectl apply -f base_deployment.yaml
kubectl apply -f configmap.yaml
kubectl apply -f nginxwebservice.yaml
kubectl get pods
```

If we have no errors the output should be:


```
NAME                                  READY   STATUS    RESTARTS   AGE
adaptix-deployment-7fb9fb7945-9ldcl   1/1     Running   0          15h
nginx-gitsync-5dcb5776bd-wdzlq        2/2     Running   0          3m12s
```

Now to test if we are serving up the correct files:
```
kubectl port-forward services/nginx-gitsync 80:8080
```

In a separate terminal session from our jumpshost:

```
curl localhost:8080
```

We should see the output:

```
<HTML><H1>This is our test page</H1></HTML>
```

## Adding a listener for AdaptixC2

Now we are going to create the listener for our agents in AdaptixC2 for the rest of the testing.


```
ssh admin1@devadmin1.lan -L 0.0.0.0:4321:192.168.200.186:4321
```

Then we connect to our AdaptixC2 instance
![Pasted image 20250930214754.png](images/Pasted%20image%2020250930214754.png)

Open the listener and right mouse click 

![Pasted image 20250930214845.png](images/Pasted%20image%2020250930214845.png)

Then create the listener  with URI being /uri
![Pasted image 20250930215442.png](images/Pasted%20image%2020250930215442.png)

For testing ,I copied the 2 snakeoil certificates that come with Kali to test the https backend.  

## Adding the ingress files and testing both deployments together

Additionally 2 ingress files will be need for www.example.com.

http-website-ingress.yaml:

```
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: webroot
spec:
  ingressClassName: nginx
  rules:
  - host: "www.example.com"
    http:
      paths:
      - pathType: Prefix
        path: "/"
        backend:
          service:
            name: nginx-gitsync
            port:
              number: 80

```

This will be the catch all and show our webserver on nginx.  One thing to take notice of is that the annotations shows the backend to be http.

adaptix-c2-ingress.yaml :

```
apiVersion: networking.k8s.io/v1
kind: Ingress

metadata:
  name: adatpixc2-https
  annotations:
    nginx.ingress.kubernetes.io/backend-protocol: "HTTPS"
spec:
  ingressClassName: nginx
  rules:
  - host: "www.example.com"
    http:
      paths:
      - pathType: Prefix
        path: "/uri"
        backend:
          service:
            name: opsadaptixwebservice
            port:
              number: 443
```

This points to any URI beginning with the prefix of /uri for example

https://www.example.com/uri/test

Will go to our the AdaptixC2.  Also notice that this annotation tells the ingress controller the backend is https.  The backend has to be https to generate https payloads.

Finally  our jumpbox host we edit the /etc/host to include www.example.com pointing to one of our nodes in the cluster.

 For example:

```
# The following lines are desirable for IPv6 capable hosts
::1 localhost ip6-localhost ip6-loopback
ff02::1 ip6-allnodes
ff02::2 ip6-allrouters
192.168.200.186 www.example.com

```

Finally lets test both ingress rules and make sure they route correctly.

Using curling  on https://www.example.com

```
 curl -k https://www.example.com
```

If correctly configured will output:

```
<HTML><H1>This is our test page</H1></HTML>
```

If use curl on C2 URI.

```
curl -k https://www.example.com/uri/test
```

The output should be this:

```
admin1@devadmin1:~/dev/nginx-gitsync$ curl -k https://www.example.com/uri/test
<!DOCTYPE html>
<html>
<head>
<title>ERROR 404 - Nothing Found</title>
</head>
<body>
<h1 class="cover-heading">ERROR 404 - PAGE NOT FOUND</h1>
</div>
</div>
</div>
</body>
</html>



```

This is AdaptixC2's 404 page.  Which means everything is being directed correctly.

I have created these resources on GitHub at:

https://github.com/TheR3dC0rner/adaptixdeploymentbuild

Next blog we are going to take these declarative deployments and turn them into deployable helm charts that we can deploy with GitOps as the third part of this helm series.

By outlining the needed parts and having a functional declarative version makes it easier to see what items need variables. 
