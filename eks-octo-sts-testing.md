# octo-sts Github triggers via external entities testing

## About

We want to stop using unreliable GitHub Actions `cron schedule` to trigger scheduled workflows for various platform activities.

Looking at both EKS Cronjobs and AWS Lambda functions to manage the actual timing of these reliably:

`external cron > authenticated with GH > trigger workflow_dispatch`

### Helloworld workflow

https://github.com/ministryofjustice/sw-ecr-testing/blob/main/.github/workflows/hello-world.yml

### Chainguard config (this will change to reflect entity and OIDC issuer)
https://github.com/ministryofjustice/sw-ecr-testing/blob/main/.github/chainguard/hello-world-trigger.sts.yaml

## Cloud Platform Pod Test

- Create a service account

```
$ kubectl create sa octo-sts-test -n [test-namespace]
```

- Apply the deployment contained in this repo

```
kubectl apply -f deploy/deploy.yaml -n [test-namespace]
```

- Exec into the pod

```
kubectl exec -it -n [test-namespace] deployment/octo-sts-test -- /bin/sh
```

- Manually test octo-sts auth flow:

```
# Read projected token from deployment volume path

TOKEN=$(cat /var/run/secrets/octo-sts.dev/token)

# Exchange with octo-sts.dev with scope and identity

TOKEN=$(cat /var/run/secrets/octo-sts.dev/token)

RESPONSE=$(curl -sf -H "Authorization: Bearer ${TOKEN}" \
"https://octo-sts.dev/sts/exchange?scope=ministryofjustice/sw-ecr-testing&identity=hello-world-trigger")

GH_TOKEN=${RESPONSE#*\"token\":\"}
GH_TOKEN=${GH_TOKEN%%\"*}

# Trigger the hello-world.yml workflow via GH API

curl -sf -X POST \
-H "Authorization: Bearer ${GH_TOKEN}" \
-H "Accept: application/vnd.github+json" \
"https://api.github.com/repos/ministryofjustice/sw-ecr-testing/actions/workflows/hello-world.yml/dispatches" \
-d '{"ref":"main"}'
```
