# Traefik TLS with Azure Key Vault / CSI

## This procedure walks through the steps to use Traefik ingress with a TLS secret provided by the Azure Key Vault Provider for Secrets Store CSI Driver.  (https://github.com/Azure/secrets-store-csi-driver-provider-azure)



* Create Namespace for Traefik.

```
kubectl create ns traefik-ingress
```

* Install Traefik

```
helm repo add traefik https://helm.traefik.io/traefik
helm repo update
helm install traefik traefik/traefik -n traefik-ingress
```

* Create self-signed certificate for testing

``` 
openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
-out aks-ingress-tls.crt \
-keyout aks-ingress-tls.key \
-subj "/CN=tlstest.southlincolnstudios.com/O=aks-ingress-tls"
```
***Replace CN above with the appropriate hostname***

* Convert Generated cert to pfx for import to Keyvault

```
openssl pkcs12 -export -out aks-ingress.pfx -inkey aks-ingress-tls.key -in aks-ingress-tls.crt
```

* Create Keyvault

* Import Certificate
 Screens:

![Import Certificate - 1](/images/importcert-1.png)

![Import Certificate - 2](/images/importcert-2.png)

![Import Certificate - 3](/images/importcert-3.png)

* Create Access Policy for AKS SP
    * Get SP used for AKS
        ```
        az vmss identity show -g <K8s-MC-VMSS-RG> -n <K8s-VMSS-Name> -o yaml
        ```
        ```
        Output:
        principalId: null
        tenantId: null
        type: UserAssigned
        userAssignedIdentities:
        ? /subscriptions/xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx/resourceGroups/MC_AKS_CSI-Certs_AKS-CSI_Certs_eastus/providers/Microsoft.ManagedIdentity/userAssignedIdentities/AKS-CSI_Certs-agentpool
        : clientId: 22222222-2222-2222-2222-222222222222
            principalId: 11111111-1111-1111-1111-111111111111

Create a Key Vault Access Policy that leverages the <b>clientId</b> above for RBAC.  Assign it Certificate permissions so it can access the certificate.

* Install CSI-Driver via Helm

```
helm repo add csi-secrets-store-provider-azure https://raw.githubusercontent.com/Azure/secrets-store-csi-driver-provider-azure/aster/charts
helm install csi-secrets-store-provider-azure/csi-secrets-store-provider-azure --generate-name
```

Deploy a SecretProviderClass - Note this needs to be in the same Namespace as the application.
  ```
  kubectl apply -f SecretProviderclass.yaml
  ```

## Secret Provider Example

```
apiVersion: secrets-store.csi.x-k8s.io/v1alpha1
kind: SecretProviderClass
metadata:
  name: azure-tls
spec:
  provider: azure
  secretObjects:                                # secretObjects defines the desired state of synced K8s secret objects
  - secretName: ingress-tls-csi
    type: kubernetes.io/tls
    data: 
    - objectName: csitls-test
      key: tls.key
    - objectName: csitls-test
      key: tls.crt
  parameters:
    useVMManagedIdentity: "true"  
    userAssignedIdentityID: "11111111-1111-1111-1111-111111111111"  #ClientId from step above required here.  This is the Id that will be used to access Key Vault.
    usePodIdentity: "false"
    keyvaultName: "<KEYVAULT_NAME>"   # the name of the KeyVault
    objects: |
      array:
        - |
          objectName: csitls-test                        # Name of certificate in the Key Vault
          objectType: secret
    tenantId: "zzzzzzzz-zzzz-zzzz-zzzz-zzzzzzzzzzzz"     # the tenant ID of the KeyVault
```

Note to use useVMManagedIdentity: "true" as we're using the VMManagedIdentity to access the Keyvault

* Deploy Demo Application
  ```
  kubectl create ns nginxdemo
  kubectl apply -f nginxdemoapp -n nginxdemo
  ```

## Demo Application example

```  
apiVersion: apps/v1
kind: Deployment
metadata:
  creationTimestamp: null
  labels:
    app: nginx
  name: nginx
  namespace: nginxdemo
spec:
  replicas: 1
  selector:
    matchLabels:
      app: nginx
  strategy: {}
  template:
    metadata:
      creationTimestamp: null
      labels:
        app: nginx
    spec:
      containers:
      - image: nginx
        name: nginx
        volumeMounts:
        - name: secrets-store-inline
          mountPath: /mnt/certs
          readOnly: true
      volumes:
      - name: secrets-store-inline
        csi:
          driver: secrets-store.csi.k8s.io
          readOnly: true
          volumeAttributes:
            secretProviderClass: "azure-tls"
---
apiVersion: v1
kind: Service
metadata:
  creationTimestamp: null
  labels:
    app: nginx
  name: nginxdemo
spec:
  ports:
  - port: 80
    protocol: TCP
    targetPort: 80
  selector:
    app: nginx
status:
  loadBalancer: {}
```

* Create Ingress object for Application Ingress
  ```
  kubectl apply -f ingressobject.yaml -n nginxdemo
  ```

## Ingress Resource Example
```
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: tlstest
  namespace: nginxdemo
  annotations:
    kubernetes.io/ingress.class: traefik
    traefik.ingress.kubernetes.io/router.entrypoints: websecure
    traefik.ingress.kubernetes.io/router.tls: 'true'
spec:
  rules:
  - host: tlstest.southlincolnstudios.com
    http:
      paths:
      - backend:
          serviceName: nginxdemo
          servicePort: 80
        path: /
  tls:
   - secretName: ingress-tls-csi
```

## Note:  The following Annotations are required for the TLS frontend to pick up the Ingress Requests with Traefik.
```
traefik.ingress.kubernetes.io/router.entrypoints: websecure
traefik.ingress.kubernetes.io/router.tls: 'true'
```

* Get the Public IP Address for the application
```
kubectl get svc -n traefik-ingress
    NAME      TYPE           CLUSTER-IP    EXTERNAL-IP    PORT(S)                      AGE  
    traefik   LoadBalancer   10.0.161.72   52.191.82.55   80:32692/TCP,443:30414/TCP   5d19h
```
* Update public DNS record with the IP Address for the Ingress Controller Service.
  * You can also update hosts file for local testing purposes

* Test Application
    ![Application Test](/images/apptest.png)


Reference:  
https://docs.microsoft.com/en-us/azure/aks/ingress-own-tls

https://github.com/traefik/traefik-helm-chart

https://doc.traefik.io/traefik/v2.3/routing/providers/kubernetes-ingress/

https://github.com/Azure/secrets-store-csi-driver-provider-azure#optional-sync-with-kubernetes-secrets

https://github.com/Azure/secrets-store-csi-driver-provider-azure/blob/master/charts/csi-secrets-store-provider-azure/README.md

