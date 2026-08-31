# Porting — PostgreSQL on HPE PCAI (EzUA)

Bitnami PostgreSQL chart adapted to be exposed on PCAI. Because PostgreSQL uses a
**raw TCP** wire protocol (not HTTP), it cannot be published through the shared
HTTPS/SSO `istio-system/ezaf-gateway`. Instead it is exposed on a dedicated Istio
**TCP** listener.

## values.yaml

`ezua.virtualService` was extended with a `tcp` block:

```yaml
ezua:
  virtualService:
    endpoint: "postgresql.${DOMAIN_NAME}"   # HTTP-only, used when tcp.enabled=false
    istioGateway: "istio-system/ezaf-gateway"
    tcp:
      enabled: true
      port: 5432                # external TCP port; also opened on the ingress gateway
      createGateway: true       # create a dedicated Istio Gateway on tcp.port
      gatewaySelector:
        istio: ingressgateway   # pod labels of the ingress gateway workload
      gateway: ""               # optional: use an existing Gateway ("namespace/name")
```

## Templates

- `templates/virtualService.yaml` — now conditional:
  - `tcp.enabled=true` → routes on the TCP layer (`tcp:` block) to the primary
    Service, bound to the dedicated Gateway. This is required for databases.
  - `tcp.enabled=false` → original HTTP route (kept for backward compatibility).
- `templates/gateway.yaml` — new. When `tcp.enabled` and `tcp.createGateway`,
  creates an Istio `Gateway` with a `protocol: TCP` server on `tcp.port`.

## Required manual step (admin)

Helm cannot patch the shared ingress gateway Service, so an admin must expose the
TCP port on it once. Add `tcp.port` (5432) to the `istio-ingressgateway` Service:

```yaml
# kubectl -n istio-system edit svc istio-ingressgateway
spec:
  ports:
    - name: tcp-postgresql
      port: 5432
      targetPort: 5432
      protocol: TCP
```

> PCAI manages Istio; a direct Service edit may be reverted on reconcile. If your
> platform installs Istio via IstioOperator, add the port to the
> `ingressGateways[].k8s.service.ports` list there instead so it persists.

Confirm the selector matches the real ingress gateway pods:

```bash
kubectl -n istio-system get gateway ezaf-gateway -o jsonpath='{.spec.selector}'
kubectl -n istio-system get pods -l istio=ingressgateway
```

## Connect

```bash
GATEWAY_IP=$(kubectl -n istio-system get svc istio-ingressgateway \
  -o jsonpath='{.status.loadBalancer.ingress[0].ip}')
psql -h $GATEWAY_IP -p 5432 -U postgres -d postgres
```

## Security notes

- Set a strong `auth.postgresPassword` (or use `auth.existingSecret`).
- Raw TCP bypasses the gateway SSO. Enable TLS (`tls.enabled: true`, connect with
  `sslmode=require`) and restrict sources (LoadBalancer firewall / CIDR, or an
  Istio `AuthorizationPolicy` on the ingress gateway).
