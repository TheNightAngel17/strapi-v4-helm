# strapi-v4-helm

Using my docker implementation of Strapi v4 to allow for a highly available Strapi service. More details to come

The basic structure of this is provided by creating a new chart via the following command:

```
https://github.com/mlx-dev/strapi-v4-helm.git
```

I'll go through and describe some key items that differ from the main files

## `chart.yaml`

Version and appVersion are based off of the main Strapi version. The appVersion has an extra number on the end for different Docker builds within the Strapi version (so if i add something but don't update versions, that will increment)

## `values.yaml`
- `replicacount` - this is set up for HA, so go a head and crank-er.
- `image`
  - `.repository` - Using my docker build of Strapi which can be found at [mlxdev/strapi-v4-docker](https://hub.docker.com/repository/docker/mlxdev/strapi-v4-docker). The codebase for this is at [mlx-dev/strapi-v4-docker](https://github.com/mlx-dev/strapi-v4-docker).
  - `.pullPolicy` - i like to make sure that i have the latest, but you can totally save some time by changing this
- `service`
  - `.type` - I use metallb to expose my services, and that uses LoadBalancer, but feel free to make this whateer you want.
  - `.port` - this sets all of the port values for everything, including the Strapi environment variable _PORT_.
- `env_secrets` - a list of values that will be set as environment variables but placed in secrets within kubernetes.
- `env` - a list of variables that will be set as environment variables as static text.
- `postgresql.auth`
  - `username` - postgres username. used to set _DATABASE_USERNAME_ environment variable.
  - `password` - password for postgres user. used to set _DATABASE_PASSWORD_ environment variable.
  - `database` - postgres database name. used to set _DATABASE_NAME_ environment variable.
- `persistence`
  - `enabled` - enables PVCs - highly recommended to borderline required for HA.
  - `accessMode` - especially with HA, this needs to be accessable to multiple pods
  - `size` - this depends on the media stored as well as amount of content types. The more you have the larger this will need to be.

## `templates\deployment.yaml`
- `spec.template.spec`
  - `.containers[0]`
    - `.env` - adding environment variables
      - loops through `.Values.env` and adds all of those
      - loops through `.Values.env_secrets` and adds all of the secretKeyRefs
      - adds _PORT_ from `.Values.service.port`
      - adds all relevent _DATABASE\_*_ values form `.Values.postgresql.auth`
    - `.ports[0].containerPort` - set from `.Values.service.port`
    - `volumeMounts` mounts the following pats to the same volume
      - `/opt/app/public/uploads` - media uploads data
      - `/opt/app/src` - content types data
  - `volumes[0]` - set for the volumeMounts above. if persistence is enabled, use PVC othersie just use emptyDir.

## `templates\persistentvolumeclaim.yaml`
sets persistent volume claim if enabled in `.Values.persistence.enabled`. Nothing crazy, just a simple PVC.


## `templates\secret.yaml`
adds new secret dependency. loops through all `.Values.env_secrets` values and adds them to the secret dependency.