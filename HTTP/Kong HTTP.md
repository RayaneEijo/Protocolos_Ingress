# Teste de https utilizando o Ingress Controller kong:
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

## Instalação do Kong:
  ```
  kubectl create -f https://bit.ly/k4k8s
  ```
  ou
  ```
  helm repo add kong https://charts.konghq.com
  helm repo update
# Helm 3
  helm install kong/kong --generate-name --set ingressController.installCRDs=false 
  ```
### Verificar a versão
  ```
  ```
### Verificar se instalou todos os componentes:
  ```
  kubectl get all -n kong
  ```
      *Retorna informação similar a:*
      ```
      NAME                               READY   STATUS    RESTARTS   AGE
      pod/ingress-kong-694fb8d8f-x7h7p   1/2     Running   0          44s

      NAME                              TYPE           CLUSTER-IP    EXTERNAL-IP      PORT(S)                      AGE
      service/kong-proxy                LoadBalancer   10.88.0.206   35.247.210.158   80:31429/TCP,443:30683/TCP   45s
      service/kong-validation-webhook   ClusterIP      10.88.5.140   <none>           443/TCP                      45s

      NAME                           READY   UP-TO-DATE   AVAILABLE   AGE
      deployment.apps/ingress-kong   0/1     1            0           44s

      NAME                                     DESIRED   CURRENT   READY   AGE
      replicaset.apps/ingress-kong-694fb8d8f   1         1         0       44s
      ```
##  Criar a variavel com o IP:
```
  kubectl get namespaces
  kubectl get services -n kong
```
      *Retorna informação similar a:*
      ```
      NAME                      TYPE           CLUSTER-IP    EXTERNAL-IP      PORT(S)                      AGE
      kong-proxy                LoadBalancer   10.88.0.206   35.247.210.158   80:31429/TCP,443:30683/TCP   86s
      kong-validation-webhook   ClusterIP      10.88.5.140   <none>           443/TCP                      86s
      ```
 ``` 
  export EXEMPLO_IP=$(kubectl get -o jsonpath="{.status.loadBalancer.ingress[0].ip}" service -n kong kong-proxy)
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
    kubernetes.io/ingress.class: "kong"
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
      Content-Type: text/plain; charset=utf-8
      Content-Length: 64
      Connection: keep-alive
      Date: Sat, 18 Dec 2021 21:22:34 GMT
      X-Kong-Upstream-Latency: 2
      X-Kong-Proxy-Latency: 1
      Via: kong/2.5.1

      Hello, world!
      Version: 1.0.0
      Hostname: teste-app-95bc4bdc-68fnr
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

      Benchmarking 35.247.210.158.nip.io (be patient)
      Completed 5000 requests
      Completed 10000 requests
      Completed 15000 requests
      Finished 16471 requests


      Server Software:
      Server Hostname:        35.247.210.158.nip.io
      Server Port:            80

      Document Path:          /
      Document Length:        64 bytes

      Concurrency Level:      100
      Time taken for tests:   20.000 seconds
      Complete requests:      16471
      Failed requests:        0
      Keep-Alive requests:    16371
      Total transferred:      4529277 bytes
      HTML transferred:       1054144 bytes
      Requests per second:    823.54 [#/sec] (mean)
      Time per request:       121.428 [ms] (mean)
      Time per request:       1.214 [ms] (mean, across all concurrent requests)
      Transfer rate:          221.15 [Kbytes/sec] received

      Connection Times (ms)
                    min  mean[+/-sd] median   max
      Connect:        0    1  12.8      0     119
      Processing:   117  118   3.4    118     152
      Waiting:      117  118   3.4    118     152
      Total:        117  120  14.8    118     269

      Percentage of the requests served within a certain time (ms)
        50%    118
        66%    118
        75%    118
        80%    118
        90%    119
        95%    122
        98%    134
        99%    234
      100%    269 (longest request)
      ```


