### 1 - Create the namespace

```bash
kubectl create ns local-k8s-setup
```

### 2 - Create the secret and configmaps

Create self signed SSL certificate and key to access the application securely.

```copy
openssl req -nodes -new -x509 -keyout key.pem -out cert.pem -days 365
```

Create tls secret from the cert and key

```copy
kubectl create secret tls nginxsecret --key key.pem --cert cert.pem  -n local-k8s-setup
```

Create the nginx configuration file 

```copy
cat <<EOF>nginx.conf

server {
        listen 80 default_server;
        listen [::]:80 default_server ipv6only=on;

        listen 443 ssl;
    
        root /usr/share/nginx/html;
        index index.html;
    
        server_name localhost;
        ssl_certificate /etc/nginx/ssl/tls.crt;
        ssl_certificate_key /etc/nginx/ssl/tls.key;
    
        location / {
                try_files $uri $uri/ =404;
        }
}
EOF
```

Create a Configmap for the nginx configuration file.

```copy
kubectl create configmap nginxconfigmap --from-file=nginx.conf -n local-k8s-setup
```

Create the index.html.

```copy
cat <<EOF>index.html
<!DOCTYPE html>
<html>
	<head>
		<title>OpenLabs StarterKit Tutorial</title>
		<style>
    	body {
      	 width: 35em;
      	 margin: 0 auto;
       	font-family: Tahoma, Verdana, Arial, sans-serif;
      	}
		</style>
	</head>
	<body>
		<h1>Kubernetes and OpenShift StarterKit Tutorial</h1>
		<p>Thank you for going through the Starter Kit Tutorial.</p>
	</body>
</html>
EOF
```

Create a Configmap for the index.html file.

```copy
kubectl create configmap indexconfigmap --from-file=index.html -n local-k8s-setup
```

### 3 - Deploy the application

Create the yaml file to deploy the service and nginx.

```copy
cat <<EOF>nginxapp.yaml
---
apiVersion: v1
kind: Service
metadata:
  name: nginxsvc
  labels:
    app: nginx
spec:
  type: NodePort
  ports:
  - port: 80
    protocol: TCP
    name: http
  - port: 443
    protocol: TCP
    name: https
  selector:
    app: nginx
---
apiVersion: v1
kind: Pod
metadata:
  name: starterkit-nginx
  labels:
    app: nginx
spec:
  volumes:
  - name: secret-volume
    secret:
       secretName: nginxsecret
  - name: configmap-volume
    configMap:
       name: nginxconfigmap
  - name: index-volume
    configMap:
       name: indexconfigmap
  containers:
  - name: nginxhttps
    image: nginx
    ports:
    - containerPort: 443
    - containerPort: 80
    volumeMounts:
    - mountPath: /etc/nginx/ssl
      name: secret-volume
    - mountPath: /etc/nginx/conf.d
      name: configmap-volume
    - mountPath: /usr/share/nginx/html
      name: index-volume
EOF
```

Create the service and application

```copy
kubectl create -f nginxapp.yaml -n local-k8s-setup
```

### 4 - Verify the setup is complete

```copy
kubectl get svc, pod, configmap, secret -n local-k8s-setup
```
