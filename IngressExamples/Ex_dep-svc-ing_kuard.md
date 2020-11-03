### example_deploy_arm.yaml
```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: kuard
spec:
  selector:
    matchLabels:
      app: kuard
  replicas: 1
  template:
    metadata:
      labels:
        app: kuard
    spec:
      containers:
      - image: gcr.io/kuar-demo/kuard-arm64:1
        imagePullPolicy: Always
        name: kuard
        ports:
        - containerPort: 8080
```        

### example_svc.yaml
```        
apiVersion: v1
kind: Service
metadata:
  name: kuard
spec:
  ports:
  - port: 80
    targetPort: 8080
    protocol: TCP
  selector:
    app: kuard
```        

### example_ingress.yaml
```        
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: kuard
  annotations:
    kubernetes.io/ingress.class: "nginx"
    #cert-manager.io/issuer: "letsencrypt-staging"

spec:
  tls:
  - hosts:
    - b8kery.almond.local
    secretName: example-tls
  rules:
  - host: b8kery.almond.local
    http:
      paths:
      - path: /
        backend:
          serviceName: kuard
          servicePort: 80
```        