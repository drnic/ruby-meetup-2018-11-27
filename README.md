# Knative examples for Ruby Meetup

## Build demo images

```shell
docker build -t drnic/meetup:v1 meetup-v1
docker build -t drnic/meetup:v2 meetup-v2

docker push drnic/meetup:v1
docker push drnic/meetup:v2
```

## Serving

### Deploying v1

```console
$ knctl deploy --service meetup --image drnic/meetup:v1

$ knctl curl -s meetup
Hello v1!
```

### Local DNS routing to remote cluster

`knctl curl` is necessary because the generated routes aren't real. We can use `kwt` to map all Knative routes from local machine to remote Kubernetes cluster:

```shell
sudo -E kwt net start --dns-map-exec='knctl dns-map'
```

In another terminal:

```shell
curl meetup.default.example.com
```

Or can access website in local browser now.

### Deploying v2

Knative does not require explicitly tagged images, but we can use them:

```console
$ knctl deploy --service meetup --image drnic/meetup:v2

$ knctl curl -s meetup
Hello v2!
```

The difference between `:v1` and `:v2` is an environment variable `TARGET`. We can override this on our next deploy:

```console
$ knctl deploy --service meetup --image drnic/meetup:v2 --env TARGET="revision 3"
$ knctl curl -s meetup
Hello revision 3!
```

### Revisions are explicit

```console
$ knctl revisions list
Service  Name          Tags      Annotations  Conditions  Age  Traffic
meetup   meetup-00003  latest    -            4 OK / 4    48s  100% -> meetup.default.example.com
~        meetup-00002  previous  -            4 OK / 4    1m   -
~        meetup-00001  -         -            4 OK / 4    10m  -
```

### Rollout of traffic to new revisions

```console
$ knctl deploy --service meetup --image drnic/meetup:v2 \
    --env TARGET="revision 4" \
    --managed-route=false
$ knctl rollout --route meetup -p meetup:previous=80% -p meetup:latest=20%

$ knctl revisions list
Service  Name          Tags      Annotations  Conditions  Age  Traffic
meetup   meetup-00004  latest    -            4 OK / 4    1m   20% -> meetup.default.example.com
~        meetup-00003  previous  -            4 OK / 4    3m   80% -> meetup.default.example.com
~        meetup-00002  -         -            2 OK / 4    3m   -
~        meetup-00001  -         -            2 OK / 4    12m  -
```

Later:

```console
$ knctl rollout --route meetup -p meetup:previous=50% -p meetup:latest=50%
$ knctl revisions list
Service  Name          Tags      Annotations  Conditions  Age  Traffic
meetup   meetup-00004  latest    -            4 OK / 4    4m   50% -> meetup.default.example.com
~        meetup-00003  previous  -            4 OK / 4    6m   50% -> meetup.default.example.com
...
```

Finally:

```console
$ knctl rollout --route meetup -p meetup:latest=100%
$ knctl revisions list
Revisions

Service  Name          Tags      Annotations  Conditions  Age  Traffic
meetup   meetup-00004  latest    -            4 OK / 4    5m   100% -> meetup.default.example.com
~        meetup-00003  previous  -            4 OK / 4    6m   -
...
```
