# Configurar SSL no cluster k8s
## Passos

#### Passo 1 - Configurando os serviços de back-end
- Criando service da aplicação 1
```yaml
apiVersion: v1
kind: Service
metadata:
  name: echo1
spec:
  ports:
  - port: 80
    targetPort: 5678
  selector:
    app: echo1
```

- Criando deployment da aplicação 1
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: echo1
spec:
  selector:
    matchLabels:
      app: echo1
  replicas: 2
  template:
    metadata:
      labels:
        app: echo1
    spec:
      containers:
      - name: echo1
        image: hashicorp/http-echo
        args:
        - "-text=echo1"
        ports:
        - containerPort: 5678
```


- Criando service da aplicação 2
```yaml
apiVersion: v1
kind: Service
metadata:
  name: echo2
spec:
  ports:
  - port: 80
    targetPort: 5678
  selector:
    app: echo2
```

- Criando deployment da aplicação 2
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: echo2
spec:
  selector:
    matchLabels:
      app: echo2
  replicas: 2
  template:
    metadata:
      labels:
        app: echo2
    spec:
      containers:
      - name: echo2
        image: hashicorp/http-echo
        args:
        - "-text=echo2"
        ports:
        - containerPort: 5678
```

#### Passo 2 - Configurando o Controlador Ingress do Kubernetes para Nginx

- Criar recursos obrigatórios
```bash
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/nginx-0.26.1/deploy/static/mandatory.yaml
```
- Criar LoadBalancer
```bash
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/nginx-0.26.1/deploy/static/provider/cloud-generic.yaml
```
- Confirmando se os pods do Controlador estão UP
```bash
   kubectl get pods --all-namespaces -l app.kubernetes.io/name=ingress-nginx 
```
- Resultado esperado
```bash
NAMESPACE       NAME                                        READY   STATUS    RESTARTS   AGE
ingress-nginx   nginx-ingress-controller-7fb85bc8bb-lnm6z   1/1     Running   0          2m42s
```
- Confirmado se o LoadBalancer está UP
```bash
kubectl get svc --namespace=ingress-nginx
```
- Resultado esperado
```bash
NAME            TYPE           CLUSTER-IP      EXTERNAL-IP       PORT(S)                      AGE
ingress-nginx   LoadBalancer   10.245.247.67   203.0.113.0   80:32486/TCP,443:32096/TCP   20h
```
- Anotar EXTERNAL-IP do LoadBalancer

#### Passo 3 - Criando o Recurso do Ingress

- Criar o arquivo de Ingresso -> Ex: echo_ingress.yaml

```yaml
apiVersion: networking.k8s.io/v1beta1
kind: Ingress
metadata:
  name: echo-ingress # Nome do ingress
spec:
  rules:
  # Lista de domínios vinculados ao SERVICE da aplicação
  - host: echo1.example.com
    http:
      paths:
      - backend:
          serviceName: echo1
          servicePort: 80
  - host: echo2.example.com
    http:
      paths:
      - backend:
          serviceName: echo2
          servicePort: 80
```

- Aplicar echo_ingress.yaml
```bash
kubectl apply -f echo_ingress.yaml
```

- Testando requisição na aplicação
```bash
curl echo1.example.com
```
#### Passo 4 - Instalando e configurando o Cert-Manager

- Criando namespace para o cert-manager
```bash
kubectl create namespace cert-manager
```
- Instalando o Cert-Manager
```bash
kubectl apply --validate=false -f https://github.com/jetstack/cert-manager/releases/download/v0.12.0/cert-manager.yaml
```
- Verificando pods aplicados pela instalação
```bash
kubectl get pods --namespace cert-manager
```
- Resultado esperado
```bash
NAME                                       READY   STATUS    RESTARTS   AGE
cert-manager-5c47f46f57-jknnx              1/1     Running   0          27s
cert-manager-cainjector-6659d6844d-j8cbg   1/1     Running   0          27s
cert-manager-webhook-547567b88f-qks44      1/1     Running   0          27s
```
- Criando emissor de certificado de TESTE
```bash
nano staging_issuer.yaml
```
- Adicione o manifesto
```diff
apiVersion: cert-manager.io/v1alpha2
kind: ClusterIssuer
metadata:
 name: letsencrypt-staging
 namespace: cert-manager
spec:
 acme:
   # The ACME server URL
   server: https://acme-staging-v02.api.letsencrypt.org/directory
   # Email address used for ACME registration
+   email: your_email_address_here
   # Name of a secret used to store the ACME account private key
   privateKeySecretRef:
     name: letsencrypt-staging
   # Enable the HTTP-01 challenge provider
   solvers:
   - http01:
       ingress:
         class:  nginx
```
- Implementando ClusterIssuer
```bash
kubectl create -f staging_issuer.yaml
```
- Resultado esperado
```bash
clusterissuer.cert-manager.io/letsencrypt-staging created
```
- Alterando o echo_ingress.yaml
```diff
apiVersion: networking.k8s.io/v1beta1
kind: Ingress
metadata:
  name: echo-ingress
+ annotations:
+   kubernetes.io/ingress.class: nginx
+   cert-manager.io/cluster-issuer: letsencrypt-staging
spec:
+  tls:
+  - hosts:
+    - echo1.example.com
+    - echo2.example.com
+    secretName: echo-tls
  rules:
  - host: echo1.example.com
    http:
      paths:
      - backend:
          serviceName: echo1
          servicePort: 80
  - host: echo2.example.com
    http:
      paths:
      - backend:
          serviceName: echo2
          servicePort: 80
```
- Atualizar echo_ingress.yaml
```bash
kubectl apply -f echo_ingress.yaml
```
- Ver estado do ingress que acabou de ser criado
```bash
kubectl describe ingress 
```
- Resultado esperado
```bash
Events:
  Type    Reason             Age               From                      Message
  ----    ------             ----              ----                      -------
  Normal  CREATE             14m               nginx-ingress-controller  Ingress default/echo-ingress
  Normal  CreateCertificate  67s   cert-manager              Successfully created Certificate "echo-tls"
  Normal  UPDATE             53s   nginx-ingress-controller  Ingress default/echo-ingress
```
- Ver estado do certificado
```bash
kubectl describe certificate
```
- Resultado esperado
```bash
Events:
  Type    Reason         Age   From          Message
  ----    ------         ----  ----          -------
  Normal  GeneratedKey  2m12s  cert-manager  Generated a new private key
  Normal  Requested     2m12s  cert-manager  Created new CertificateRequest resource "echo-tls-3768100355"
  Normal  Issued        47s    cert-manager  Certificate issued successfully
```
- Testando certificado de TESTE
```bash
wget --save-headers -O- echo1.example.com
```
- Resultado esperado
```bash
. . .
HTTP request sent, awaiting response... 308 Permanent Redirect
. . .
ERROR: cannot verify echo1.example.com's certificate, issued by ‘CN=Fake LE Intermediate X1’:
  Unable to locally verify the issuer's authority.
To connect to echo1.example.com insecurely, use `--no-check-certificate'.
```

- Isso indica que o HTTPS foi habilitado com sucesso, porém, como estamos usando o certificado de teste, o certificado não pode ser verificado.

#### Passo 5 - Implantando o emissor de produção
- Criando emissor de produção
```bash
nano prod_issuer.yaml
```

```diff
apiVersion: cert-manager.io/v1alpha2
kind: ClusterIssuer
metadata:
  name: letsencrypt-prod
  namespace: cert-manager
spec:
  acme:
    # The ACME server URL
    server: https://acme-v02.api.letsencrypt.org/directory
    # Email address used for ACME registration
+    email: your_email_address_here
    # Name of a secret used to store the ACME account private key
    privateKeySecretRef:
      name: letsencrypt-prod
    # Enable the HTTP-01 challenge provider
    solvers:
    - http01:
        ingress:
          class: nginx
```

- Aplicando emissor de produção
```bash
kubectl create -f prod_issuer.yaml
```
- Resultado esperado
```bash
clusterissuer.cert-manager.io/letsencrypt-prod created
```
- Alterando o emissor no echo_ingress.yaml
```diff
apiVersion: networking.k8s.io/v1beta1
kind: Ingress
metadata:
  name: echo-ingress
  annotations:
    kubernetes.io/ingress.class: "nginx"
+   cert-manager.io/cluster-issuer: "letsencrypt-prod"
spec:
  tls:
  - hosts:
    - echo1.example.com
    - echo2.example.com
    secretName: echo-tls
  rules:
  - host: echo1.example.com
    http:
      paths:
      - backend:
          serviceName: echo1
          servicePort: 80
  - host: echo2.example.com
    http:
      paths:
      - backend:
          serviceName: echo2
          servicePort: 80
```
- Atualizando echo_ingress.yaml
```bash
kubectl apply -f echo_ingress.yaml
```
- Resultado esperado
```bash
ingress.networking.k8s.io/echo-ingress configured
```
- Verificando o status do certificado
```bash
kubectl describe certificate echo-tls
```
- Resultado esperado
```bash
Events:
  Type    Reason        Age                  From          Message
  ----    ------        ----                 ----          -------
  Normal  GeneratedKey  8m10s                cert-manager  Generated a new private key
  Normal  Requested     8m10s                cert-manager  Created new CertificateRequest resource "echo-tls-3768100355"
  Normal  Requested     35s                  cert-manager  Created new CertificateRequest resource "echo-tls-4217844635"
  Normal  Issued        10s (x2 over 6m45s)  cert-manager  Certificate issued successfully
```
- Verificar requisição na aplicação
```bash
curl echo1.example.com
```

- Resultado esperado
```bash
<html>
<head><title>308 Permanent Redirect</title></head>
<body>
<center><h1>308 Permanent Redirect</h1></center>
<hr><center>nginx/1.15.9</center>
</body>
</html>
```
- Isso indica que o servidor está redirecionando as requisições HTTP para HTTPS.
- Agora execute o ```curl``` para ```https://echo1.example.com```
```bash
curl https://echo1.example.com
```
- Resultado esperado
```bash
echo1
```

- Pronto ! HTTPS configurado, usando um certificado do Let`s Encrypt.