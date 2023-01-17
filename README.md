# mongodb-atlas
Operador MongoDB Atlas - Gerencie seus clusters MongoDB Atlas a partir do Kubernetes

# MongoDB Atlas Operator (trial version)
[![MongoDB Atlas Operator](https://github.com/mongodb/mongodb-atlas-kubernetes/workflows/Test/badge.svg)](https://github.com/mongodb/mongodb-atlas-kubernetes/actions/workflows/test.yml?query=branch%3Amain)
[![MongoDB Atlas Go Client](https://img.shields.io/badge/Powered%20by%20-go--client--mongodb--atlas-%2313AA52)](https://github.com/mongodb/go-client-mongodb-atlas)

O MongoDB Atlas Operator fornece uma integração nativa entre a plataforma de orquestração do Kubernetes e o MongoDB Atlas — o único serviço de banco de dados de documentos multinuvem que oferece a versatilidade necessária para criar aplicativos sofisticados e resilientes que podem se adaptar às mudanças nas demandas dos clientes e nas tendências do mercado.

> Status atual: *versão de teste*. O Operador oferece aos usuários a capacidade de provisionar
> Projetos Atlas, clusters e usuários de banco de dados usando as especificações do Kubernetes e vincular informações de conexão
> em aplicativos implantados no Kubernetes. Mais recursos como endpoints privados, gerenciamento de backup, autenticação LDAP/X.509, etc.
> ainda estão por vir.

A documentação completa para o Operador pode ser encontrada [here](https://docs.atlas.mongodb.com/atlas-operator/)

## Guia rápido
### Etapa 1. Implantar o operador Kubernetes usando tudo em um arquivo de configuração
```
kubectl apply -f https://raw.githubusercontent.com/mongodb/mongodb-atlas-kubernetes/main/deploy/all-in-one.yaml
```
### Etapa 2. Criar cluster do Atlas

**1.** Crie um segredo de chave de API do Atlas
Para trabalhar com o Atlas Operator, você precisa fornecer [informações de autenticação](https://docs.atlas.mongodb.com/configure-api-access)
 para permitir que o Operador do Atlas se comunique com a API do Atlas. Depois de gerar uma chave pública e privada no Atlas, você pode criar um segredo do Kuberentes com:
```
kubectl create secret generic mongodb-atlas-operator-api-key \
         --from-literal="orgId=<the_atlas_organization_id>" \
         --from-literal="publicApiKey=<the_atlas_api_public_key>" \
         --from-literal="privateApiKey=<the_atlas_api_private_key>" \
         -n mongodb-atlas-system
```

**2.** Crie um recurso personalizado `AtlasProject`

O CustomResource `AtlasProject` representa os Projetos Atlas em nosso cluster Kubernetes. Você precisa especificar
`projectIpAccessList` com os endereços IP ou blocos CIDR de quaisquer hosts que se conectarão ao Atlas Cluster.
```
cat <<EOF | kubectl apply -f -
apiVersion: atlas.mongodb.com/v1
kind: AtlasProject
metadata:
  name: my-project
spec:
  name: Test Atlas Operator Project
  projectIpAccessList:
    - ipAddress: "192.0.2.15"
      comment: "IP address for Application Server A"
    - ipAddress: "203.0.113.0/24"
      comment: "CIDR block for Application Server B - D"
EOF
```
**3.** Crie um recurso personalizado `AtlasCluster`.
O exemplo abaixo é uma configuração mínima para criar um cluster M10 Atlas na região leste dos EUA da AWS. Para obter uma lista completa de propriedades, verifique
`atlasclusters.atlas.mongodb.com` [especificação CRD](config/crd/bases/atlas.mongodb.com_atlasclusters.yaml)):
```
cat <<EOF | kubectl apply -f -
apiVersion: atlas.mongodb.com/v1
kind: AtlasCluster
metadata:
  name: my-atlas-cluster
spec:
  name: "Test-cluster"
  projectRef:
    name: my-project
  providerSettings:
    instanceSizeName: M10
    providerName: AWS
    regionName: US_EAST_1
EOF
```

**4.** Crie uma senha de usuário do banco de dados Kubernetes Secret
```
kubectl create secret generic the-user-password --from-literal="password=P@@sword%"
```

**5.** Crie um recurso personalizado `AtlasDatabaseUser`

Para se conectar a um Atlas Cluster, o usuário do banco de dados precisa ser criado. O recurso `AtlasDatabaseUser` deve fazer referência
a senha Kubernetes Secret criada na etapa anterior.
```
cat <<EOF | kubectl apply -f -
apiVersion: atlas.mongodb.com/v1
kind: AtlasDatabaseUser
metadata:
  name: my-database-user
spec:
  roles:
    - roleName: "readWriteAnyDatabase"
      databaseName: "admin"
  projectRef:
    name: my-project
  username: theuser
  passwordSecretRef:
    name: the-user-password
EOF
```
**6.** Aguarde até que o recurso personalizado `AtlasDatabaseUser` esteja pronto

Aguarde até que o recurso AtlasDatabaseUser chegue ao status "pronto" (ele aguardará até que o cluster seja criado, o que pode levar cerca de 10 minutos):
```
kubectl get atlasdatabaseusers my-database-user -o=jsonpath='{.status.conditions[?(@.type=="Ready")].status}'
True
```
### Etapa 3. Conecte seu aplicativo ao Atlas Cluster

O Atlas Operator criará um Kubernetes Secret com as informações necessárias para se conectar ao Atlas Cluster criado
na etapa anterior. Um aplicativo no mesmo Kubernetes Cluster pode montar e usar o Secret:

```
...
containers:
      - name: test-app
        env:
         - name: "CONNECTION_STRING"
           valueFrom:
             secretKeyRef:
               name: test-atlas-operator-project-test-cluster-theuser
               key: connectionString.standardSrv

```
