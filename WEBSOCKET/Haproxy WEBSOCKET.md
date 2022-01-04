# Teste de https utilizando o Ingress Controller haproxy:
*Apos a prepação do ambiente podemos:

## Expor o serviço:
```
kubectl create deployment teste-app --image=gcr.io/google-samples/hello-app:1.0
```
      *Retorna informação similar a:*
      ```
      deployment.apps/teste-app created
      ```
```
kubectl expose deployment teste-app --port=8080 --target-port=8080
```
      *Retorna informação similar a:*
      ```
      service/teste-app exposed
      ```

## Instalação do haproxy:
  ```
  helm repo add haproxytech https://haproxytech.github.io/helm-charts

  helm repo update

  helm install kubernetes-ingress haproxytech/kubernetes-ingress \
    --set controller.service.type=LoadBalancer
  ```
### Verificar a versão
  ```
  ```
### Verificar se instalou todos os componentes:
  ```
  kubectl get all 
  ```
      *Retorna informação similar a:*
      ```
      NAME                                                      READY   STATUS    RESTARTS   AGE
      pod/kubernetes-ingress-967547b7f-4s7d6                    1/1     Running   0          86s
      pod/kubernetes-ingress-967547b7f-hc7rl                    1/1     Running   0          85s
      pod/kubernetes-ingress-default-backend-759ccc6c98-tdbsw   1/1     Running   0          86s
      pod/kubernetes-ingress-default-backend-759ccc6c98-zwqgz   1/1     Running   0          86s
      pod/teste-app-95bc4bdc-l4thh                              1/1     Running   0          2m29s

      NAME                                         TYPE           CLUSTER-IP    EXTERNAL-IP     PORT(S)                                     AGE
      service/kubernetes                           ClusterIP      10.88.0.1     <none>          443/TCP                                     10m
      service/kubernetes-ingress                   LoadBalancer   10.88.1.144   35.247.223.96   80:30743/TCP,443:32044/TCP,1024:30249/TCP   87s
      service/kubernetes-ingress-default-backend   ClusterIP      None          <none>          8080/TCP                                    87s
      service/teste-app                            ClusterIP      10.88.6.92    <none>          8080/TCP                                    2m22s

      NAME                                                 READY   UP-TO-DATE   AVAILABLE   AGE
      deployment.apps/kubernetes-ingress                   2/2     2            2           87s
      deployment.apps/kubernetes-ingress-default-backend   2/2     2            2           87s
      deployment.apps/teste-app                            1/1     1            1           2m30s

      NAME                                                            DESIRED   CURRENT   READY   AGE
      replicaset.apps/kubernetes-ingress-967547b7f                    2         2         2       87s
      replicaset.apps/kubernetes-ingress-default-backend-759ccc6c98   2         2         2       87s
      replicaset.apps/teste-app-95bc4bdc                              1         1         1       2m30s
      ```
##  Criar a variavel com o IP:
```
  kubectl get namespaces
  kubectl get services
```
      *Retorna informação similar a:*
      ```
      NAME                                 TYPE           CLUSTER-IP    EXTERNAL-IP     PORT(S)                                     AGE
      kubernetes                           ClusterIP      10.88.0.1     <none>          443/TCP                                     11m
      kubernetes-ingress                   LoadBalancer   10.88.1.144   35.247.223.96   80:30743/TCP,443:32044/TCP,1024:30249/TCP   2m21s
      kubernetes-ingress-default-backend   ClusterIP      None          <none>          8080/TCP                                    2m21s
      teste-app                            ClusterIP      10.88.6.92    <none>          8080/TCP                                    3m16s
      ```
 ``` 
  export EXEMPLO_IP=$(kubectl get -o jsonpath="{.status.loadBalancer.ingress[0].ip}" service kubernetes-ingress)
  echo $EXEMPLO_IP
```
      *Retorna informação similar a:*
      ```
      35.247.210.158
      ```


##  Criar o Ingress Controller:
```
cat <<EOF > ingress-teste.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: ingress-teste
  annotations:
    kubernetes.io/ingress.class: "haproxy"
spec:
  rules:
  - host: ${EXEMPLO_IP}.nip.io
    http:
      paths:
      - pathType: Prefix
        path: "/"
        backend:
          service:
            name: teste-app
            port:
              number: 8080
EOF
kubectl apply -f ingress-teste.yaml
```
      *Retorna informação similar a:*
      ```
      ingress.networking.k8s.io/ingress-teste created
      ```

##  Testar a conexão:
  ```
  curl -i $EXEMPLO_IP.nip.io
  ```
      *Retorna informação similar a:*
      ```
      HTTP/1.1 200 OK
      date: Sat, 18 Dec 2021 21:55:30 GMT
      content-length: 64
      content-type: text/plain; charset=utf-8

      Hello, world!
      Version: 1.0.0
      Hostname: teste-app-95bc4bdc-l4thh
      ```



