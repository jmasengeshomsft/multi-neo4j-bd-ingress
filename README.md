# neo4j-ingress-demo
A repository to demonstrate how to deploy standalone Neo4J on AKS with Nginx Ingress Controller.

If you are using GitOps with FluxCD, you can easily point a flux config to this repo as it (obviously changing parameters). If using helm cli and kubectl, follow the following documentation and focus on the "values" section in each HelmRelease object and use that on the helm install command. At the end of deployment, you will have two clients connecting to two different neo4J DBs using the same ingress controller.

## Deploying Ne04J DBs per client using Helm. 

Helm Chart Link: https://neo4j.com/docs/operations-manual/current/kubernetes/helm-charts-setup/

Resources that get deployed for each client:

![image](https://github.com/jmasengeshomsft/multi-neo4j-db-ingress/assets/86074746/ded0fd1c-b879-4fa2-89ce-ae52a742fab5)


#### Pods:

![image](https://github.com/jmasengeshomsft/multi-neo4j-db-ingress/assets/86074746/5daad378-4fe6-4b55-99d1-55b36eec7c99)



##### Services:

According to the [official helm chart](https://github.com/neo4j/helm-charts/blob/e653e8fb6b3c189f92a34d98d1be400455f1d833/neo4j/values.yaml#L192), each release will create three different services:

- default: with the same name as the release and intended for internal access
- admin: for admin use. 
- neo4j: intended for external use (from the cluster perspective). In azure this can be internal or external to the cluster's vnet

![image](https://github.com/jmasengeshomsft/multi-neo4j-db-ingress/assets/86074746/9117e1f1-189d-49af-b299-d2e6610cbae9)



client's neo4j values.yaml file:
```
values:
    neo4j:
      name: client-one
      resources:
        cpu: "0.5"
        memory: "2Gi"

      # Uncomment to set the initial password
      password: "client-one-password"

      # Uncomment to use enterprise edition
      #edition: "enterprise"
      #acceptLicenseAgreement: "yes"
    services:
      neo4j:
        spec:
          type: ClusterIP
        # port: 7687
        # Uncomment to enable TLS
        tls:
         enabled: true
         secretName: client-one-tls
        # annotations:
        #   service.beta.kubernetes.io/azure-load-balancer-internal: "true"
    volumes:
      data:
        mode: "dynamic"
        dynamic:
          # * managed-csi-premium provisions premium SSD disks (recommended)
          # * managed-csi provisions standard SSD-backed disks
          storageClassName: managed-csi-premium
    # Neo4j Configuration (yaml format)
    # config:
    #   dbms.security.auth_enabled: "false"

```

## Reverse Proxy for the Browser Admin Service

client's neo4j proxy values.yaml file
```
  values:
    reverseProxy:
      serviceName: client-one-admin
      ingress:
        enabled: false
```

### Ingress Resources.

Each client has two ingresses: one for the browser based admin service and another one for bolt connection with SDKs or Cypher Shell.

#### Admin Browser Ingress
```
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: client-one-proxy-reverseproxy-ingress
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  ingressClassName: neo4j-nginx
  tls:
  - hosts:
    - client-one.myneo4jplatform.com
    secretName: client-one-tls
  rules:
  - host: client-one.myneo4jplatform.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: client-one-proxy-reverseproxy-service
            port:
              number: 80
```

#### Bolt Ingress 
```
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: client-one-bolt-ingress
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  ingressClassName: neo4j-nginx
  tls:
  - hosts:
    - client-one-tcp.myneo4jplatform.com
    secretName: client-one-tcp-tls
  rules:
  - host: client-one-tcp.myneo4jplatform.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: client-one-lb-neo4j
            port:
              number: 7687
```

## Configuring Ingress Controller

### New NGINX Ingress Controller using Helm

In order to expose the Bolt services via tcp, we need to configure NGINX ingress controller to open 7687 ports for bolt connection. We can achieve that by providing tcp port/service mapping in the values.yaml under tcp section. For example, the following set up creates a new nginx ingress controller with neo4j-nginx class and opens two ports 9001 and 9002 targetting two different services exposing 7687. References: [Exposing TCP and UDP services](https://kubernetes.github.io/ingress-nginx/user-guide/exposing-tcp-udp-services/)

```
 values:
    controller:
      service:
        externalTrafficPolicy: Local
        annotations:
          service.beta.kubernetes.io/azure-load-balancer-internal: "false"  #set to true for internal load balancer
      ingressClassResource:
        # -- Name of the ingressClass
        name: neo4j-nginx # -- This is the name of the ingressClass resource. Make sure it does not interfere with other ingressClass resources
        controllerValue: "k8s.io/neo4j-ingress-nginx"
        # -- Is this ingressClass enabled or not
        enabled: true
      ingressClass: neo4j-nginx
    tcp:  # -- TCP services. Add a new pair for each nw client: port and service name
      '9001': 'neo4j/client-one-lb-neo4j:7687'
      '9002': 'neo4j/client-two-lb-neo4j:7687'
```

### Existing Ingress Controller

Create a new config map and populate it with the following code; follow the steps located here: [Access your data via Cypher Shell](https://neo4j.com/docs/operations-manual/current/kubernetes/accessing-neo4j-ingress/#_access_your_data_via_cypher_shell)
```
apiVersion: v1
kind: ConfigMap
metadata:
  name: tcp-services
  namespace: ingress-nginx
data:
  '9001': neo4j/client-one-lb-neo4j:7687
  '9002': neo4j/client-two-lb-neo4j:7687
```

## Configuring TLS Termination. 

As of now, I was unable to connect to a Neo4J DB with a uri that contains a path. You can connect to client-one.my-db-platform.com but you cant connect to my-db-platform.com/client-one. Because of this limitation, I found it easier to use my custom domains with tls. The admin and bolt endpoints have different uri for each client. You might find this exercise difficult to follow if you are using just IP addresses. In my case the certs are pulled from key vault. 

```
apiVersion: secrets-store.csi.x-k8s.io/v1
kind: SecretProviderClass
metadata:
  name: neo4j-demo-tls
spec:
  provider: azure
  secretObjects:                            # secretObjects defines the desired state of synced K8s secret objects
    - secretName: client-one-tls
      type: kubernetes.io/tls
      data: 
        - objectName: client-one-myneo4jplatform-com
          key: tls.key
        - objectName: client-one-myneo4jplatform-com
          key: tls.crt
    - secretName: client-two-tls
      type: kubernetes.io/tls
      data: 
        - objectName: client-two-myneo4jplatform-com
          key: tls.key
        - objectName: client-two-myneo4jplatform-com
          key: tls.crt
    - secretName: client-one-tcp-tls
      type: kubernetes.io/tls
      data: 
        - objectName: client-one-tcp-myneo4jplatform-com
          key: tls.key
        - objectName: client-one-tcp-myneo4jplatform-com
          key: tls.crt
    - secretName: client-two-tcp-tls
      type: kubernetes.io/tls
      data: 
        - objectName: client-two-tcp-myneo4jplatform-com
          key: tls.key
        - objectName: client-two-tcp-myneo4jplatform-com
          key: tls.crt
  parameters:
    usePodIdentity: "false"
    useVMManagedIdentity: "true"
    userAssignedIdentityID: <kv-aks-identity>
    keyvaultName: <kv-name>                 # the name of the AKV instance
    objects: |
      array:
        - |
          objectName: client-two-myneo4jplatform-com
          objectType: secret
        - |
          objectName: client-one-myneo4jplatform-com
          objectType: secret
        - |
          objectName: client-one-tcp-myneo4jplatform-com
          objectType: secret
        - |
          objectName: client-two-tcp-myneo4jplatform-com
          objectType: secret
    tenantId: <tenant-id>
```       
