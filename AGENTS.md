This is the app of apps repository for my personal and some professional projects.

# Workspace structure
.
├── access-key
├── app-of-apps.yml
├── argocd-infra-app.yaml
├── argocd-ingress
│   └── argocd-ingress.yaml
├── cert-manager.yaml
├── cluster-issuer
│   └── cert-manager-cluster-issuer.yaml
├── cluster-issuer-app.yaml
├── external-secrets
│   ├── aws-credentials-secret.yaml
│   ├── aws-secret-store.yaml
│   ├── immoly-db-external-secret.yaml
│   ├── joy-alemazung-strapi-app-external-secret.yaml
│   ├── joy-alemazung-strapi-external-secret.yaml
│   ├── README.md
│   └── SETUP.md
├── external-secrets-config-app.yaml
├── external-secrets-operator-app.yaml
├── hetzner-ccm-app.yaml
├── immoly
│   ├── db-secret.yaml
│   ├── env-configmap.yaml
│   ├── immoly-app-deployment.yaml
│   ├── immoly-app-ingress-resource.yaml
│   ├── immoly-app-service.yaml
│   ├── immoly-migration-job.yaml
│   ├── immoly-namespace.yaml
│   ├── immoly-postgres-deployment.yaml
│   ├── immoly-postgres-service.yaml
│   ├── immoly-volume-persistentvolumeclaim.yaml
│   └── migrations-deployment.yaml
├── immoly-app.yaml
├── ingress-nginx-app.yaml
├── joy-alemazung
│   ├── alemazung-app-deployment.yaml
│   ├── alemazung-app-ingress-resource.yaml
│   ├── alemazung-app-service.yaml
│   └── alemazung-namespace.yaml
├── joy-alemazung-app.yaml
├── joy-alemazung-cms
│   ├── strapi-cm0-configmap.yaml
│   ├── strapi-cm2-configmap.yaml
│   ├── strapi-cm4-configmap.yaml
│   ├── strapi-cm5-configmap.yaml
│   ├── strapi-data-persistentvolumeclaim.yaml
│   ├── strapidb-deployment.yaml
│   ├── strapi-deployment.yaml
│   ├── strapi-ingress-resource.yaml
│   ├── strapi-postgres-service.yaml
│   ├── strapi-secret.yaml
│   ├── strapi-service.yaml
│   └── strapi-uploads-pvc.yaml
├── joy-alemazung-cms-app.yaml
├── schluesselmomente
│   ├── schluesselmomente-backend-deployment.yaml
│   ├── schluesselmomente-be-service.yaml
│   ├── schluesselmomente-deployment.yaml
│   ├── schluesselmomente-fe-service.yaml
│   ├── schluesselmomente-ingress-redirect-resource.yaml
│   ├── schluesselmomente-ingress-resource.yaml
│   └── schluesselmomente-namespace.yaml
├── schluesselmomente-app.yaml
└── secret-access-key

