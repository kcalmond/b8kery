Using examples here: https://www.runf11s.com/PiHole-and-Unbound-on-k8s

### config map 
```
apiVersion: v1
kind: ConfigMap
metadata:
  name: pihole-config
  namespace: pihole
data:
  ServerIP: 192.168.1.201
  DNS1: 1.1.1.1
  DNS2: 9.9.9.9
  WEBPASSWORD: testpihole
  TZ: America/Los_Angeles
```

### TCP/UDP services

```
apiVersion: v1
kind: Service
metadata:
  name: pihole-svc-tcp
  namespace: pihole
  annotations:
    metallb.universe.tf/allow-shared-ip: PiHoleShared
spec:
  type: LoadBalancer
  loadBalancerIP: 192.168.0.201
  ports:
    - port: 80
      targetPort: http
      protocol: TCP
      name: http
    - port: 53
      targetPort: dns-tcp
      protocol: TCP
      name: dns-tcp
  selector:
    app: pihole
---
apiVersion: v1
kind: Service
metadata:
  name: pihole-svc-udp
  namespace: pihole
  annotations:
    metallb.universe.tf/allow-shared-ip: PiHoleShared
spec:
  type: LoadBalancer
  loadBalancerIP: 192.168.0.201
  ports:
    - port: 53
      targetPort: dns-udp
      protocol: UDP
      name: dns-udp
  selector:
    app: pihole
```

### Deployment

```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: pihole
  namespace: pihole
  labels:
    name: pihole
spec:
  selector:
    matchLabels:
      app: pihole
  template:
    metadata:
      labels:
        app: pihole
    spec:
      containers:
      - name: pihole
        image: pihole/pihole
        resources:
          limits:
            memory: "256Mi"
            cpu: "750m"
        ports:
        - containerPort: 53
          name: dns-tcp
          protocol: TCP
        - containerPort: 53
          name: dns-udp
          protocol: UDP
        - containerPort: 80
          name: http
          protocol: TCP
        envFrom:
        - configMapRef:
            name: pihole-config
        securityContext:
          capabilities:
            add:
              - NET_ADMIN
        livenessProbe:
          httpGet:
            path: /admin.index.php
            port: 80
          initialDelaySeconds: 60
          periodSeconds: 5
        readinessProbe:
          httpGet:
            path: /admin.index.php
            port: 80
          initialDelaySeconds: 25
          periodSeconds: 5
```
Check service and app deployment:
```
> k get svc,ing -o wide
NAME                     TYPE           CLUSTER-IP      EXTERNAL-IP     PORT(S)                     AGE   SELECTOR
service/pihole-svc-tcp   LoadBalancer   10.144.44.67    192.168.0.201   80:31183/TCP,53:31729/TCP   35m   app=pihole
service/pihole-svc-udp   LoadBalancer   10.144.42.228   192.168.0.201   53:32557/UDP                34m   app=pihole
 ~/GH/b8kery/raspbernetes | master ?5 ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ 13:38:29 
> curl 192.168.0.201

    <!doctype html>
    <html lang='en'>
        <head>
            <meta charset='utf-8'>
            
            <title>● </title>
            <link rel='stylesheet' href='pihole/blockingpage.css'>
            <link rel='shortcut icon' href='admin/img/favicons/favicon.ico' type='image/x-icon'>
        </head>
        <body id='splashpage'>
            <img src='admin/img/logo.svg' alt='Pi-hole logo' width='256' height='377'>
            <br>
            <p>Pi-<strong>hole</strong>: Your black hole for Internet advertisements</p>
            <a href='/admin'>Did you mean to go to the admin panel?</a>
        </body>
    </html>
    %         
```

