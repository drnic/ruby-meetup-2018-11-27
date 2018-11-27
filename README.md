# Knative examples for Ruby Meetup

## Build demo images

```shell
docker build -t drnic/meetup:v1 myapp-v1
docker build -t drnic/meetup:v2 myapp-v2

docker push drnic/meetup:v1
docker push drnic/meetup:v2
```

## Serving

### Deploying v1

```console
$ knctl deploy --service meetup --image index.docker.io/drnic/meetup:v1

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
$ knctl deploy --service meetup --image index.docker.io/drnic/meetup:v2

$ knctl curl -s meetup
Hello v2!
```

The difference between `:v1` and `:v2` is an environment variable `TARGET`. We can override this on our next deploy:

```console
$ knctl deploy --service meetup --image index.docker.io/drnic/meetup:v2 --env TARGET="revision 3"
$ knctl curl -s meetup
Hello revision 3!
```

### Revisions are explicit

```console
$ knctl revisions list
...
```

### Rollout of traffic to new revisions

```console
$ knctl deploy --service meetup --image index.docker.io/drnic/meetup:v2 \
    --env TARGET="revision 4" \
    --managed-route=false
$ knctl rollout --route meetup -p meetup:previous=80% -p meetup:latest=20%
$ knctl routes show --route meetup
Percent  Revision      Service  Domain
80%      meetup-00003  -        meetup.default.knative.starkandwayne.com
20%      meetup-00004  -        meetup.default.knative.starkandwayne.com
```

Mapping knative routes to local machine:

```shell
sudo -E kwt net start --dns-map-exec='knctl dns-map'
```

In another terminal:

```console
$ curl meetup.default.knative.starkandwayne.com
Hello My message!
$ curl meetup.default.knative.starkandwayne.com
Hello My next message!
```