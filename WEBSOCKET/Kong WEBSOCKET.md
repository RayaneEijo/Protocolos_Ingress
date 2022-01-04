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
kubectl expose deployment teste-app  --protocol TCP --port=8080 --target-port=8080
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

     EXEMPLO ENCONTRADO:
          curl -i -N -H "Connection: Upgrade" \
            -H "Upgrade: websocket" \
            -H "Origin: http://localhost" \
            -H "Host: ${EXEMPLO_IP}.nip.io" \
            -H "Sec-Websocket-Version: 13" \
            -H "Sec-WebSocket-Key: 123" \
            $EXEMPLO_IP/teste
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
