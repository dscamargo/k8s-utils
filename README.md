# Em desenvolvimento :wrench:
# k8s-utils :fire:
Arquivos úteis para configuração do cluster e deploy de aplicações no k8s.

### Nodejs 

| Arquivo | Descrição |
| ------ | ------ |
| configmap.yaml | Variáveis de ambiente atráves de configmap |
| deployment.yaml | Deployment para aplicações NodeJS |
| deployment-w-configmap.yaml | Exemplo de deployment de aplicações NodeJS usando configmaps |
| service.yaml | Service usando NodePort para usar dominio com SSL |

### SSL :lock:

| Arquivo | Descrição |
| ------ | ------ |
| ingress.yaml | Nginx ingress para certificados SSL |

#### Issuers

| Arquivo | Descrição |
| ------ | ------ |
| prod_issuers.yaml | Emissor de SSL para clusters em produção |
| staging_issuers.yaml | Emissor de SSL fake para testes |

