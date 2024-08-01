# Unifi Network Application Helm Chart

Chart templates built using [Truecharts common](https://truecharts.org/common).

## Components
- [Unifi Network Application](https://hub.docker.com/r/linuxserver/unifi-network-application)
- [Bitnami Mongodb](https://github.com/bitnami/charts/blob/main/bitnami/mongodb)


## TLS
Todo, explain how to update default cert with `keytool`.

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

## Ingress
For ingress I'm using [Istio](https://istio.io/latest/) which I'm not including templates for in this chart.

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
      - unifi.my-domain.com
    port:
      number: 443
      protocol: HTTPS
      name: https-unifi
	# Unifi requires to host the TLS cert itself
    tls:
      mode: PASSTHROUGH
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
  tls:
  - match:
    - port: 443
      sniHosts:
      - unifi.my-domain.com
    route:
    - destination:
        host: unifi.unifi.svc.cluster.local
        port:
          number: 8443
  http:
  - match:
    - port: 8080
    route:
    - destination:
        host: unifi.unifi.svc.cluster.local
        port:
          number: 8080
```
