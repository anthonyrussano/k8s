CHART_VERSION="4.12.0-beta.0"
APP_VERSION="1.12.0-beta.0"

mkdir ./kubernetes/ingress/controller/nginx/manifests/

helm template ingress-nginx ingress-nginx \
--repo https://kubernetes.github.io/ingress-nginx \
--version ${CHART_VERSION} \
--namespace ingress-nginx \
> ./ingress/controller/nginx/manifests/nginx-ingress.${APP_VERSION}.yaml