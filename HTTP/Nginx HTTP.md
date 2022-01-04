# Teste de https utilizando o Ingress Controller NGINX:
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

## Instalação do NGINX:
  ```
  kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v0.48.1/deploy/static/provider/cloud/deploy.yaml
  ```
  ou
  ```
  helm upgrade --install ingress-nginx ingress-nginx \
    --repo https://kubernetes.github.io/ingress-nginx \
    --namespace ingress-nginx --create-namespace
  ```
### Verificar a versão
  ```
  POD_NAMESPACE=ingress-nginx
  POD_NAME=$(kubectl get pods -n $POD_NAMESPACE -l app.kubernetes.io/name=ingress-nginx --field-selector=status.phase=Running -o name)
  kubectl exec $POD_NAME -n $POD_NAMESPACE -- /nginx-ingress-controller --version
  ```
      *Retorna informação similar a:*
      ```
      -------------------------------------------------------------------------------
      NGINX Ingress controller
        Release:       v1.1.0
        Build:         cacbee86b6ccc45bde8ffc184521bed3022e7dee
        Repository:    https://github.com/kubernetes/ingress-nginx
        nginx version: nginx/1.19.9

      -------------------------------------------------------------------------------
      ```
### Verificar se instalou todos os componentes:
  ```
  kubectl get all -n ingress-nginx
  ```
      *Retorna informação similar a:*
      ```
      NAME                                          READY   STATUS    RESTARTS   AGE
      pod/ingress-nginx-controller-54bfb9bb-tks7n   1/1     Running   0          2m29s

      NAME                                         TYPE           CLUSTER-IP     EXTERNAL-IP      PORT(S)                      AGE
      service/ingress-nginx-controller             LoadBalancer   10.88.6.26     35.247.210.158   80:31239/TCP,443:30416/TCP   2m29s
      service/ingress-nginx-controller-admission   ClusterIP      10.88.11.143   <none>           443/TCP                      2m29s

      NAME                                       READY   UP-TO-DATE   AVAILABLE   AGE
      deployment.apps/ingress-nginx-controller   1/1     1            1           2m30s

      NAME                                                DESIRED   CURRENT   READY   AGE
      replicaset.apps/ingress-nginx-controller-54bfb9bb   1         1         1       2m30s
      ```
##  Criar a variavel com o IP:
```
  kubectl get namespaces
  kubectl get services -n ingress-nginx
```
      *Retorna informação similar a:*
      ```
      NAME                                 TYPE           CLUSTER-IP    EXTERNAL-IP      PORT(S)                      AGE
      ingress-nginx-controller             LoadBalancer   10.88.2.206   35.247.210.158   80:31184/TCP,443:31970/TCP   82s
      ingress-nginx-controller-admission   ClusterIP      10.88.2.112   <none>           443/TCP                      82s
      ```
 ``` 
  export EXEMPLO_IP=$(kubectl get -o jsonpath="{.status.loadBalancer.ingress[0].ip}" service -n ingress-nginx ingress-nginx-controller)
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
    kubernetes.io/ingress.class: "nginx"
    nginx.ingress.kubernetes.io/ssl-redirect: "false"
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
      Date: Sat, 18 Dec 2021 19:12:36 GMT
      Content-Type: text/plain; charset=utf-8
      Content-Length: 64
      Connection: keep-alive

      Hello, world!
      Version: 1.0.0
      Hostname: teste-app-95bc4bdc-g2cqf
      ```

## Instalar benchmark  e testar (ApacheBench):
  ```
  apt-get update
  sudo apt-get install apache2-utils
  ab -k -n 50000 -c 100 -t 20 http://$EXEMPLO_IP.nip.io/
  ```
      *Retorna informação similar a:*
      ```
      This is ApacheBench, Version 2.3 <$Revision: 1843412 $>
      Copyright 1996 Adam Twiss, Zeus Technology Ltd, http://www.zeustech.net/
      Licensed to The Apache Software Foundation, http://www.apache.org/

      Benchmarking 35.247.194.204.nip.io (be patient)
      Completed 5000 requests
      Completed 10000 requests
      Completed 15000 requests
      Finished 16453 requests


      Server Software:
      Server Hostname:        35.247.194.204.nip.io
      Server Port:            80

      Document Path:          /
      Document Length:        64 bytes

      Concurrency Level:      100
      Time taken for tests:   20.005 seconds
      Complete requests:      16453
      Failed requests:        0
      Keep-Alive requests:    16353
      Total transferred:      3372365 bytes
      HTML transferred:       1052992 bytes
      Requests per second:    822.44 [#/sec] (mean)
      Time per request:       121.590 [ms] (mean)
      Time per request:       1.216 [ms] (mean, across all concurrent requests)
      Transfer rate:          164.62 [Kbytes/sec] received

      Connection Times (ms)
                    min  mean[+/-sd] median   max
      Connect:        0    1  12.8      0     119
      Processing:   117  118   4.3    117     177
      Waiting:      117  118   4.3    117     177
      Total:        117  120  15.3    117     294

      Percentage of the requests served within a certain time (ms)
        50%    117
        66%    118
        75%    118
        80%    118
        90%    120
        95%    124
        98%    135
        99%    234
      100%    294 (longest request)
      ```