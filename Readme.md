# Traefik JWT Decode

![GitHub tag (latest by date)](https://img.shields.io/github/v/tag/SimonSchneider/traefik-jwt-decode)
![Docker Image Size (latest by date)](https://img.shields.io/docker/image-size/simonschneider/traefik-jwt-decode)
[![Coverage](http://gocover.io/_badge/github.com/SimonSchneider/traefik-jwt-decode/decoder)](http://gocover.io/github.com/SimonSchneider/traefik-jwt-decode/decoder)
[![Go Report Card](https://goreportcard.com/badge/github.com/SimonSchneider/traefik-jwt-decode)](https://goreportcard.com/report/github.com/SimonSchneider/traefik-jwt-decode)

[Traefik Forward auth](https://docs.traefik.io/middlewares/forwardauth/)
implementation that decodes and validates JWT (JWS) tokens and populates
headers with configurable claims from the token.
The tokens are validated using jwks, checked for expiration and cached.

If the token is invalid, ie. can't be verified or is expired `traefik-jwt-decode`
will respond with a `UNAUTHORIZED 401`.

If the token is valid `traefik-jwt-decode` will respond with a `OK 200` and
headers mapped from the claims of the token and an additional (configurable) `jwt-token-validated: true` header. 
Traefik should be configured to forward these headers via the `authResponseHeaders` which forwards them to the
end destination.

If no token is present on the request and `AUTH_HEADER_REQUIRED` is `true`, `traefik-jwt-decode` will return 401.

If no token is present on the request and `AUTH_HEADER_REQUIRED` is `false`, `traefik-jwt-decode` will return 200 and set the header `jwt-token-validated: false`.

## Installation and usage

### Minimal with helm

The below example will deploy `traefik-jwt-decode` into kubernetes which 
will map the claims `email` and `scopes` into the headers `jwt-token-email` and
`jwt-token-scopes`.

It will then create a traefik forwardAuth middleware that forwards the 
`jwt-token-validated`, `jwt-token-email` and `jwt-token-scopes` to the upstream service.
```
cd _helm

helm install traefik-jwt-decode traefik-jwt-decode \
  --set env.JWKS_URL="https://www.googleapis.com/oauth2/v3/certs" \
  --set env.CLAIM_MAPPINGS="email:jwt-token-email,scopes:jwt-token-scopes"

cat <<EOF >> traefik-auth-resource.yaml
apiVersion: traefik.containo.us/v1alpha1
kind: Middleware
metadata:
  name: jwt-decode-auth
spec:
  forwardAuth:
    address: http://traefik-jwt-decode:8080
    authResponseHeaders:
      - jwt-token-validated
      - jwt-token-email
      - jwt-token-scopes
EOF

kubectl apply -f traefik-auth-resource.yaml
```

### Configuring and running the docker image:

minimal (with claimMapping env variable)
```
docker run \
  -e CLAIM_MAPPINGS="claim-123:header-123,claim-456:header-456" \
  -e JWKS_URL="https://www.googleapis.com/oauth2/v3/certs" \
  -p 8080:8080 \
  simonschneider/traefik-jwt-decode:latest
```

minimal (with claim file):
```
echo "{ \"claim-123\": \"header-123\" }" > config.json

docker run \
  -v $(pwd)/config.json:/config.json \
  -e JWKS_URL="https://www.googleapis.com/oauth2/v3/certs" \
  -p 8080:8080 \
  simonschneider/traefik-jwt-decode:latest
```

### Configuration reference

required configurations:
```
JWKS_URL
url pointing at the jwks json file (https://auth0.com/docs/tokens/concepts/jwks)
```

default configurations
```
CLAIM_MAPPING_FILE_PATH    = config.json
AUTH_HEADER_KEY            = Authorization
TOKEN_VALIDATED_HEADER_KEY = jwt-token-validated
AUTH_HEADER_REQUIRED       = false
PORT                       = 8080
LOG_LEVEL                  = info                = trace | debug | info | warn | crit
LOG_TYPE                   = json                = json | pretty
MAX_CACHE_KEYS             = 10000
CACHE_ENABLED              = true
FORCE_JWKS_ON_START        = true
```

optional configurations
```
CLAIM_MAPPINGS=claim1:header1,claim2:header2
set up claim mappings by env, on the format
the above corresponds to the json

{
  "claim1": "header1",
  "claim2": "header2"
}
```

## Papercup setup

### More than one JWKS_URL
Because we're having services running in dev cluster that authenticate via the production Auth0 tenant, in dev setup we're passing JWKS_URL as a list of URLs, and we check against each of them until we get a result or fail.
This allows us to have prod authorisations pass in dev contexts (ex: Stapler-V2).
The idea here is to always pass in first the dev tenant then the prod one. 
Importantly, this should only be done in dev cluster, not in prod!!!

### So how is this used as an auth service?

At present, we have two instances of filter-paper: `filter-paper-prod` and `filter-paper-dev`.  
`filter-paper-prod` is responsible for authentication using the `papercup` auth0 [tenant](https://auth0.com/docs/glossary#T), while `filter-paper-dev` is responsible for authentication using the `papercup-dev` auth0 tenant.  
This would mean that if a frontend service required an authenticated request to a backend service behind `filter-paper` authentication, we would need to ensure that filter-paper's `env.JWKS_URL` corresponds to the correct tenant against which the user received a signed jwt.  


### How to set this up?
We actually require two pieces of infra to be setup to enable filter paper:

Firstly we need to set up traefik to use this as a [ForwardAuth middleware](https://doc.traefik.io/traefik/middlewares/overview/#configuration-example).
When doing this, we want to set up the `spec.forwardAuth.address` to be the same `host` in filter-paper's k8s `{stage}.yaml` file.  
Another thing to mention here, is that the downstream services may not only require filter-paper to forward or terminate a request based on a valid jwt, but also could require certain pieces of information from the jwt to be forwarded (for example, the auth0 user id or roles).  
Information as to how to accomplish this can be found [here](https://doc.traefik.io/traefik/middlewares/http/forwardauth/#authresponseheaders).  
At the time of writing, we have middlewares setup for this [here](https://github.com/papercup-ai/charts/tree/main/setup/middlewares).

Secondly, we need to actually deploy the `filter-paper` server. At present, CI is set up for both prod and dev instances to deploy to the relevant k8s cluster.
