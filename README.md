# Unifi Network Application Helm Chart

Chart templates built using [Truecharts common](https://truecharts.org/common).

## Components
- [Unifi Network Application](https://hub.docker.com/r/linuxserver/unifi-network-application)
- [Bitnami Mongodb](https://github.com/bitnami/charts/blob/main/bitnami/mongodb)


## TLS
You can update TLS manually with `keytool`.

```sh
src_pass="-srcstorepass aircontrolenterprise"
pass="-deststorepass aircontrolenterprise -destkeypass aircontrolenterprise"

cd /usr/lib/unifi/data
keytool -import -noprompt -trustcacerts -alias root -file ./crt $pass $src_pass -keystore ./keystore
openssl pkcs12 -passout pass: -export -in crt -inkey key -out server.p12 -name server
keytool -importkeystore -destkeystore server.keystore -srckeystore server.p12 -srcstoretype PKCS12 $pass -srcstorepass '' -alias server
keytool -importkeystore -srckeystore ./server.keystore $pass $src_pass -destkeystore keystore
keytool -delete -alias unifi $pass -keystore ./keystore
keytool -changealias -alias server -destalias unifi $pass -keystore ./keystore
```

## Istio Ingress
For ingress I'm using [Istio](https://istio.io/latest/). The destination rule makes Istio ignore the insecure TLS and terminates TLS with its own valid cert. With this you don't have to worry about managing TLS within the pod.

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: Gateway
metadata:
  name: ingress-gateway
  namespace: istio-system
spec:
  servers:
  - hosts:
      - unifi.my-domain.com
    port:
      number: 8080
      protocol: HTTP
      name: http-unifi
  - hosts:
    - '*.my-domain.com.com'
    port:
      name: https-my-domain
      number: 443
      protocol: HTTPS
    tls:
      credentialName: my-domain-certificate
      mode: SIMPLE
---
apiVersion: networking.istio.io/v1
kind: DestinationRule
metadata:
  name: unifi-tls
spec:
  host: "unifi.unifi.svc.cluster.local"
  trafficPolicy:
    portLevelSettings:
    - port:
        number: 8443
      tls:
        mode: SIMPLE
        insecureSkipVerify: true
---
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: unifi
spec:
  gateways:
  - istio-system/ingress-gateway
  hosts:
  - unifi.my-domain.com
  http:
  - match:
    - port: 8080
    route:
    - destination:
        host: unifi.unifi.svc.cluster.local
        port:
          number: 8080
  - match:
    - port: 443
    route:
    - destination:
        host: unifi.unifi.svc.cluster.local
        port:
          number: 8443
```
