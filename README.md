# Create Kubernetes user account restricted to namespace

This is the instructions to create a user and restrict user to access only one namespace in Kubernetes.

## Step 1: Create a namespace

Let’s start by creating a namespace that will be used for this demo `blue-green`

```
kubectl create namespace blue-green
```

## Step 2: Create a Service Account

**We’ll create a service account called demo-user in the blue-green namespace**
```
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: ServiceAccount
metadata:
  name: demo-user
  namespace: blue-green 
EOF
```


## Step 3: Create a Role

A role contains rules that represent a set of permissions that grant access to resources within a namespace.

**First confirm API versions for RBAC available in your Kubernetes cluster:**
```
kubectl api-versions| grep  rbac   
```


**Let’s create a role which will give created account complete access to namespace resources.**

```
cat <<EOF | kubectl apply -f -
kind: Role
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  namespace: blue-green
  name: poc-role
rules:
- apiGroups: ["", "extensions", "apps"]
  resources: ["deployments", "replicasets", "pods", "services", "ingresses"]
  verbs: ["get", "list", "watch", "create", "update", "patch", "delete"] # You can also use ["*"]
EOF
```

Confirm role creation using the following command:
```
kubectl get roles -n blue-green 
```



## Step 4: Bind the role to a user
```
cat <<EOF | kubectl apply -f -
kind: RoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: admin-view
  namespace: blue-green
subjects:
- kind: ServiceAccount
  name: demo-user
  namespace: blue-green
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: poc-role
EOF
```

**Confirm role binding creation using the following command:**
```
kubectl get rolebindings --namespace blue-green
```


###Check user token name:

```
$ kubectl describe sa demo-user -n blue-green
Name:                demo-user
Namespace:           blue-green
Labels:              
Annotations:         kubectl.kubernetes.io/last-applied-configuration:
                       {"apiVersion":"v1","kind":"ServiceAccount","metadata":{"annotations":{},"name":"demo-user","namespace":"blue-green"}}
Image pull secrets:  
Mountable secrets:   demo-user-token-k9qbl
Tokens:              demo-user-token-k9qbl
Events:             
```


**Get service account token to be used to access Kubernetes  through kubectl command line.**
```
export NAMESPACE="blue-green"
export K8S_USER="demo-user"
kubectl -n ${NAMESPACE} describe secret $(kubectl -n ${NAMESPACE} get secret | (grep ${K8S_USER} || echo "$_") | awk '{print $1}') | grep token: | awk '{print $2}'\n
          
```


My output is:
```
eyJhbGciOiJSUzI1NiIsImtpZCI6ImR2V0dxTEpYUUV6alpzMWFxOThyMTN0dDlkT2dWTlE3VlNuTDFENm9RbUUifQ.eyJpc3MiOiJrdWJlcm5ldGVzL3NlcnZpY2VhY2NvdW50Iiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9uYW1lc3BhY2UiOiJibHVlLWdyZWVuIiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9zZWNyZXQubmFtZSI6ImRlbW8tdXNlci10b2tlbi04eHZqYiIsImt1YmVybmV0ZXMuaW8vc2VydmljZWFjY291bnQvc2VydmljZS1hY2NvdW50Lm5hbWUiOiJkZW1vLXVzZXIiLCJrdWJlcm5ldGVzLmlvL3NlcnZpY2VhY2NvdW50L3NlcnZpY2UtYWNjb3VudC51aWQiOiI1NTFlNWEzZi02NmI3LTQ0ODAtYTNkOC04ZDBiOTkwOTE4MmMiLCJzdWIiOiJzeXN0ZW06c2VydmljZWFjY291bnQ6Ymx1ZS1ncmVlbjpkZW1vLXVzZXIifQ.V0uvKpR34nintkwDHDXis1zr3z8wsvC3AxAsiiNBnVIXVxGWvHfFIMjHT1UcKL1RGiDJE1gWrG9iLdjQETojWDXL60kFqZBrFUg4I7CerA4i8iz7VxNjHoUsRpMciZDwiBjJvyVqHfeRUPV3ksgb6a4b_YB1WJghs5urwvY6qZl2GEicTl8-BbOHEEyRa9bN1lmIXJNqusRseXKowS4VTLwaGdfqIkAICLkIOz52d17MYnwp0yui5OUBcEewNODo1xf4X1J3Ce_A0KOKuoGRNZgbQxRqUcEXvlxbIGR1cpT1hFGc1BC19KImLEzSEjSKtOIGSzzblYlTr7Fbh2ggQw      
```


**Get certificate data**
```
kubectl  -n ${NAMESPACE} get secret `kubectl -n ${NAMESPACE} get secret | (grep ${K8S_USER} || echo "$_") | awk '{print $1}'` -o "jsonpath={.data['ca\.crt']}" 
```

My output:
```
LS0tLS1CRUdJTiBDRVJUSUZJQ0FURS0tLS0tCk1JSUM1ekNDQWMrZ0F3SUJBZ0lCQURBTkJna3Foa2lHOXcwQkFRc0ZBREFWTVJNd0VRWURWUVFERXdwcmRXSmwKY201bGRHVnpNQjRYRFRJeU1ETXdNekV5TVRJME1sb1hEVE15TURJeU9URXlNVEkwTWxvd0ZURVRNQkVHQTFVRQpBeE1LYTNWaVpYSnVaWFJsY3pDQ0FTSXdEUVlKS29aSWh2Y05BUUVCQlFBRGdnRVBBRENDQVFvQ2dnRUJBTlBVCnhXdVg1eGpVNFNZNlV3ekg5VGk1bkxIK3JnY2ZqS1lCdENjM1RiZVNYR1RCai81dmlvTldnaVk2NWVOMG1oL0wKT1p4STBXRWxmYndTZHBXVFZHU1B2WkNYODM2SlA5NVE1em1zZStrTVBqR0tYL0M0b1RJTGRZSWN4ZEgxcm00WgpuMm1nSVpJcjM3THlUOUNudnRwdlplcHEyYSswMVlWeEJBYTVYRlQwU0xmSkZidzZvQTVSQzNQRlF0Y3ZXbXBUCmU1U0pvTUVLZ241RGlhRERjUndHNWNPMUtzWW1kVUZTb29EOHI0d3BUNEJ6ZE1Ea0sxSWJUWCswRXhFLy9zZjEKcG03aDNjbWpaNXBDc3ZDb0ljOW53S0lseEVBN240UElQYzB0Z29iaUxMUzI5UmxZd09vbHlkZ3gvUXlSZXNUYgpGeDh5RTdhNlExYWx0QzRYazZVQ0F3RUFBYU5DTUVBd0RnWURWUjBQQVFIL0JBUURBZ0trTUE4R0ExVWRFd0VCCi93UUZNQU1CQWY4d0hRWURWUjBPQkJZRUZJNUxOdTNqZnI1VERqU0FDU095ZWtrZWRWNGJNQTBHQ1NxR1NJYjMKRFFFQkN3VUFBNElCQVFEU29KTTlrd2FwQ2M1cjFGKy9Xc2dKOHkza2dBOG5UYTI0MHAxY09wUndlVHh0YUgzdwovTVNvS0JNTkVhaDRrbGFTbm1kNTVwQ2djNUJGRTQyUHZOeHZoaEhyUUtWREk3LzFKSEJSVC9HZ1p1cytiMk5sClhwSzM3cVJBalpORThrNjd5ZUt0VlBYQnd3VlhFdjVIMjhsZ1hORUsvbVZZdkVnRU5qQVg0UjYwWmtXSG9qQzUKNFZnclBLZVJiRjVWMWdoZ3RZR0g5QUI3dk44Q05lYjBGai82Z3BzTFF5YldMMzNiYUhNMUVhRFpjNVRON1llNgpXNkJoY1FBaWdJQ0RZbERVQVltMDNFOXVXZ1V0SFFKUGxMQVh1azd2aWhxd2lFZllJeVNSWENxRjAvQmF0Ny9lCnVveEFhMlhoU0ZxWGFzTGJUT3hqdnRZdGZyb0hWQ2dwYWo2bAotLS0tLUVORCBDRVJUSUZJQ0FURS0tLS0tCg==
```


## Step 5: Creating kubectl configuration
kubeconfig path is ~/.kube

If you want to configure kubectl with obtained credentials, it will look something like below.


**Retrieve the endpoint for your cluster**
```
aws eks describe-cluster \
    --region us-west-2 \
    --name aws001-preprod-dev-eks \
    --query "cluster.endpoint" \
    --output text
```

My output:
```
https://1C5E3EA44D66F9AB9B2CE67AEF2571F7.gr7.us-west-2.eks.amazonaws.com
```





**Create new config file for this demo as follows:**
```
apiVersion: v1
clusters:
- cluster:
    certificate-authority-data: LS0tLS1CRUdJTiBDRVJUSUZJQ0FURS0tLS0tCk1JSUM1ekNDQWMrZ0F3SUJBZ0lCQURBTkJna3Foa2lHOXcwQkFRc0ZBREFWTVJNd0VRWURWUVFERXdwcmRXSmwKY201bGRHVnpNQjRYRFRJeU1ETXdNekV5TVRJME1sb1hEVE15TURJeU9URXlNVEkwTWxvd0ZURVRNQkVHQTFVRQpBeE1LYTNWaVpYSnVaWFJsY3pDQ0FTSXdEUVlKS29aSWh2Y05BUUVCQlFBRGdnRVBBRENDQVFvQ2dnRUJBTlBVCnhXdVg1eGpVNFNZNlV3ekg5VGk1bkxIK3JnY2ZqS1lCdENjM1RiZVNYR1RCai81dmlvTldnaVk2NWVOMG1oL0wKT1p4STBXRWxmYndTZHBXVFZHU1B2WkNYODM2SlA5NVE1em1zZStrTVBqR0tYL0M0b1RJTGRZSWN4ZEgxcm00WgpuMm1nSVpJcjM3THlUOUNudnRwdlplcHEyYSswMVlWeEJBYTVYRlQwU0xmSkZidzZvQTVSQzNQRlF0Y3ZXbXBUCmU1U0pvTUVLZ241RGlhRERjUndHNWNPMUtzWW1kVUZTb29EOHI0d3BUNEJ6ZE1Ea0sxSWJUWCswRXhFLy9zZjEKcG03aDNjbWpaNXBDc3ZDb0ljOW53S0lseEVBN240UElQYzB0Z29iaUxMUzI5UmxZd09vbHlkZ3gvUXlSZXNUYgpGeDh5RTdhNlExYWx0QzRYazZVQ0F3RUFBYU5DTUVBd0RnWURWUjBQQVFIL0JBUURBZ0trTUE4R0ExVWRFd0VCCi93UUZNQU1CQWY4d0hRWURWUjBPQkJZRUZJNUxOdTNqZnI1VERqU0FDU095ZWtrZWRWNGJNQTBHQ1NxR1NJYjMKRFFFQkN3VUFBNElCQVFEU29KTTlrd2FwQ2M1cjFGKy9Xc2dKOHkza2dBOG5UYTI0MHAxY09wUndlVHh0YUgzdwovTVNvS0JNTkVhaDRrbGFTbm1kNTVwQ2djNUJGRTQyUHZOeHZoaEhyUUtWREk3LzFKSEJSVC9HZ1p1cytiMk5sClhwSzM3cVJBalpORThrNjd5ZUt0VlBYQnd3VlhFdjVIMjhsZ1hORUsvbVZZdkVnRU5qQVg0UjYwWmtXSG9qQzUKNFZnclBLZVJiRjVWMWdoZ3RZR0g5QUI3dk44Q05lYjBGai82Z3BzTFF5YldMMzNiYUhNMUVhRFpjNVRON1llNgpXNkJoY1FBaWdJQ0RZbERVQVltMDNFOXVXZ1V0SFFKUGxMQVh1azd2aWhxd2lFZllJeVNSWENxRjAvQmF0Ny9lCnVveEFhMlhoU0ZxWGFzTGJUT3hqdnRZdGZyb0hWQ2dwYWo2bAotLS0tLUVORCBDRVJUSUZJQ0FURS0tLS0tCg==
    server: https://1C5E3EA44D66F9AB9B2CE67AEF2571F7.gr7.us-west-2.eks.amazonaws.com
  name: arn:aws:eks:us-west-2:695292474035:cluster/aws001-preprod-dev-eks


contexts:
- context:
    cluster: arn:aws:eks:us-west-2:695292474035:cluster/aws001-preprod-dev-eks
    namespace: blue-green
    user: demo-user
  name: blue-green

current-context: blue-green
kind: Config
preferences: {}


users:
- name: demo-user
  user:
    token: eyJhbGciOiJSUzI1NiIsImtpZCI6ImR2V0dxTEpYUUV6alpzMWFxOThyMTN0dDlkT2dWTlE3VlNuTDFENm9RbUUifQ.eyJpc3MiOiJrdWJlcm5ldGVzL3NlcnZpY2VhY2NvdW50Iiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9uYW1lc3BhY2UiOiJibHVlLWdyZWVuIiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9zZWNyZXQubmFtZSI6ImRlbW8tdXNlci10b2tlbi04eHZqYiIsImt1YmVybmV0ZXMuaW8vc2VydmljZWFjY291bnQvc2VydmljZS1hY2NvdW50Lm5hbWUiOiJkZW1vLXVzZXIiLCJrdWJlcm5ldGVzLmlvL3NlcnZpY2VhY2NvdW50L3NlcnZpY2UtYWNjb3VudC51aWQiOiI1NTFlNWEzZi02NmI3LTQ0ODAtYTNkOC04ZDBiOTkwOTE4MmMiLCJzdWIiOiJzeXN0ZW06c2VydmljZWFjY291bnQ6Ymx1ZS1ncmVlbjpkZW1vLXVzZXIifQ.V0uvKpR34nintkwDHDXis1zr3z8wsvC3AxAsiiNBnVIXVxGWvHfFIMjHT1UcKL1RGiDJE1gWrG9iLdjQETojWDXL60kFqZBrFUg4I7CerA4i8iz7VxNjHoUsRpMciZDwiBjJvyVqHfeRUPV3ksgb6a4b_YB1WJghs5urwvY6qZl2GEicTl8-BbOHEEyRa9bN1lmIXJNqusRseXKowS4VTLwaGdfqIkAICLkIOz52d17MYnwp0yui5OUBcEewNODo1xf4X1J3Ce_A0KOKuoGRNZgbQxRqUcEXvlxbIGR1cpT1hFGc1BC19KImLEzSEjSKtOIGSzzblYlTr7Fbh2ggQw
    client-key-data: LS0tLS1CRUdJTiBDRVJUSUZJQ0FURS0tLS0tCk1JSUM1ekNDQWMrZ0F3SUJBZ0lCQURBTkJna3Foa2lHOXcwQkFRc0ZBREFWTVJNd0VRWURWUVFERXdwcmRXSmwKY201bGRHVnpNQjRYRFRJeU1ETXdNekV5TVRJME1sb1hEVE15TURJeU9URXlNVEkwTWxvd0ZURVRNQkVHQTFVRQpBeE1LYTNWaVpYSnVaWFJsY3pDQ0FTSXdEUVlKS29aSWh2Y05BUUVCQlFBRGdnRVBBRENDQVFvQ2dnRUJBTlBVCnhXdVg1eGpVNFNZNlV3ekg5VGk1bkxIK3JnY2ZqS1lCdENjM1RiZVNYR1RCai81dmlvTldnaVk2NWVOMG1oL0wKT1p4STBXRWxmYndTZHBXVFZHU1B2WkNYODM2SlA5NVE1em1zZStrTVBqR0tYL0M0b1RJTGRZSWN4ZEgxcm00WgpuMm1nSVpJcjM3THlUOUNudnRwdlplcHEyYSswMVlWeEJBYTVYRlQwU0xmSkZidzZvQTVSQzNQRlF0Y3ZXbXBUCmU1U0pvTUVLZ241RGlhRERjUndHNWNPMUtzWW1kVUZTb29EOHI0d3BUNEJ6ZE1Ea0sxSWJUWCswRXhFLy9zZjEKcG03aDNjbWpaNXBDc3ZDb0ljOW53S0lseEVBN240UElQYzB0Z29iaUxMUzI5UmxZd09vbHlkZ3gvUXlSZXNUYgpGeDh5RTdhNlExYWx0QzRYazZVQ0F3RUFBYU5DTUVBd0RnWURWUjBQQVFIL0JBUURBZ0trTUE4R0ExVWRFd0VCCi93UUZNQU1CQWY4d0hRWURWUjBPQkJZRUZJNUxOdTNqZnI1VERqU0FDU095ZWtrZWRWNGJNQTBHQ1NxR1NJYjMKRFFFQkN3VUFBNElCQVFEU29KTTlrd2FwQ2M1cjFGKy9Xc2dKOHkza2dBOG5UYTI0MHAxY09wUndlVHh0YUgzdwovTVNvS0JNTkVhaDRrbGFTbm1kNTVwQ2djNUJGRTQyUHZOeHZoaEhyUUtWREk3LzFKSEJSVC9HZ1p1cytiMk5sClhwSzM3cVJBalpORThrNjd5ZUt0VlBYQnd3VlhFdjVIMjhsZ1hORUsvbVZZdkVnRU5qQVg0UjYwWmtXSG9qQzUKNFZnclBLZVJiRjVWMWdoZ3RZR0g5QUI3dk44Q05lYjBGai82Z3BzTFF5YldMMzNiYUhNMUVhRFpjNVRON1llNgpXNkJoY1FBaWdJQ0RZbERVQVltMDNFOXVXZ1V0SFFKUGxMQVh1azd2aWhxd2lFZllJeVNSWENxRjAvQmF0Ny9lCnVveEFhMlhoU0ZxWGFzTGJUT3hqdnRZdGZyb0hWQ2dwYWo2bAotLS0tLUVORCBDRVJUSUZJQ0FURS0tLS0tCg==
```



**Add that file path to your KUBECONFIG environment variable**
```
export KUBECONFIG=$KUBECONFIG:~/.kube/config-my-cluster
```


Let’s confirm that it works:
```
kubectl --kubeconfig ./config-demo-user get pods 
NAME                                 READY   STATUS    RESTARTS   AGE
dev-superapi-geba-654d6cdf56-gg4qf   1/1     Running   0          2d19h
dev-superapi-geba-654d6cdf56-wmw2t   1/1     Running   0          2d19h


kubectl --kubeconfig ./config-demo-user get nodes
Error from server (Forbidden): nodes is forbidden: User "system:serviceaccount:blue-green:demo-user" cannot list resource "nodes" in API group "" at the cluster scope
```
