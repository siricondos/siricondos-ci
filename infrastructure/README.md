# Single-VM dev deployment

This setup targets the existing k3s node at 45.79.237.142 (2 CPUs, 4 GiB RAM).
Harbor remains in Docker Compose; Traefik terminates HTTPS for Harbor, Argo CD,
and the dev applications. Caddy is not required after the ingress cutover.

## Configuration

- `traefik-values.yaml`: Helm chart `traefik/traefik` version `42.0.0`, one
  DaemonSet pod, K3s ServiceLB on ports 80/443, persistent ACME storage on a local-path PVC.
- `ingress.yaml`: Harbor routes to the host Docker bridge at `172.17.0.1:8080`;
  Argo CD routes to its existing Service on port 8081. Adjust these addresses
  when installing on another VM. Argo CD requires `server.insecure: "true"`
  in `argocd-cmd-params-cm`; HTTPS terminates at Traefik. Use `--grpc-web` with
  the Argo CD CLI through this HTTP ingress.
- `../components/dev-small`: one replica per deployment, 128 MiB memory requests,
  384 MiB limits, startup probes, Java heap sizing and frontend TCP health checks.
  These are initial dev limits; tune from measured usage before enabling all apps.
- `../argocd-appset.yaml`: generates only dev applications. The create-update
  policy preserves existing prod Applications when the controller permits
  per-ApplicationSet policy overrides. Production auto-sync must remain disabled
  and its deployments scaled to zero; do not re-enable prod on this small VM.

## Apply

Only apply after the manifests have been pushed to the `main` branch Argo CD watches.
Back up existing Applications and workloads before changing an existing ApplicationSet.
Ensure its policy is honored before removing prod from its generator. Existing prod
Application finalizers can otherwise cascade deletion of workloads.

```sh
export KUBECONFIG=/etc/rancher/k3s/k3s.yaml
helm upgrade --install traefik traefik/traefik --version 42.0.0 \
  --namespace traefik --create-namespace -f infrastructure/traefik-values.yaml
kubectl apply -f infrastructure/ingress.yaml
kubectl patch cm argocd-cmd-params-cm -n argocd --type=merge \
  -p '{"data":{"applicationsetcontroller.enable.policy.override":"true"}}'
kubectl set env deployment/argocd-applicationset-controller -n argocd \
  ARGOCD_APPLICATIONSET_CONTROLLER_ENABLE_POLICY_OVERRIDE=true
kubectl rollout status deployment/argocd-applicationset-controller -n argocd
kubectl apply -f argocd-appset.yaml
```

Release ports 80/443 from Caddy before enabling the Traefik LoadBalancer Service.
For staging, set `service.spec.type=NodePort` and test HTTPS on NodePort 30443 first.
Do not also set Traefik hostPort values: those conflict with K3s ServiceLB. Save the previous Helm
revision and Caddy configuration for rollback.

Traefik uses Let's Encrypt HTTP-01 on port 80, with its ACME account and certificates
in `/data/acme.json` on the PVC. Back up this private file securely; never commit it.
The migration imports existing Caddy certificates to avoid a TLS interruption.
Do not run multiple Traefik replicas sharing this ACME file.

## Application prerequisites

DNS for `dev.siricondos.com`, `api-dev.siricondos.com`, `admin-dev.siricondos.com`,
`argo.siricondos.com`, and `registry.siricondos.com` points to this VM.

At inspection, Harbor contained only `discovery-service:v0.0.1-dev` in the
`siricondos` project. Publish images for `authentication-service`, `file-service`,
`gateway-service`, `provider-service`, `siricondos`, and `siricondos-admin`, then
update each dev overlay's `newTag` to the actual published tag. Do not assume
`v0.0.0-dev` exists. Supply service-specific database/auth configuration and
credentials separately. The dev shared ConfigMap contains the supplied service
settings; `shared-env-secrets` in `apps-dev` supplies `POSTGRES_PASSWORD` and
`PAYSTACK_SECRET_KEY` to backend services and is managed outside Git.
The supplied API/payment URLs target production and the Paystack key is a live key.
`NEXT_PUBLIC_GOOGLE_MAPS_API_KEY` must also be supplied to frontend Docker builds,
since Next.js public variables are embedded at build time. If the Harbor
project becomes private, configure a read-only image-pull Secret in `apps-dev`.

Check memory after bringing up each service. This VM is for a reduced development
setup; capacity for the complete stack has not been established.

## Verification

```sh
kubectl get applications -n argocd
kubectl get pods -n apps-dev
kubectl get deployments -n apps-prod
kubectl top nodes
curl --fail https://registry.siricondos.com/api/v2.0/health
curl --fail -o /dev/null https://argo.siricondos.com/
```

Reference: [Traefik ACME](https://doc.traefik.io/traefik/reference/install-configuration/tls/certificate-resolvers/acme/)
and [ApplicationSet modification policies](https://argo-cd.readthedocs.io/en/stable/operator-manual/applicationset/Controlling-Resource-Modification/).

## Harbor startup

`harbor.service` starts the existing Compose containers after the logging container
is healthy, avoiding the boot-time logging dependency failure observed after resize.
It assumes `/root/harbor/harbor` and the `harbor-log` container already exist.
Install it in `/etc/systemd/system/harbor.service`, then run
`systemctl daemon-reload` and `systemctl enable --now harbor.service`.
This does not replace a Harbor data/database backup.
