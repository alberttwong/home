# Deploying JupyterHub to OpenShift

This is entirely based on the work of Graham Dupleton <gdumplet@redhat.com> who has been very helpful in getting this
set up. Mostly it is based on instructions found in his
[jupyterhub-quickstart](https://github.com/jupyter-on-openshift/jupyterhub-quickstart)
GitHub repo.

What is deployed is:

* s2i builders for basic, scipy and tensorflow notebook images and for Jupyter Hub
* JupyterHub using a Postgresql database
* Notebooks and hub support JupyterLab interface
* SSO using Keycloak

## Prerequistes

OpenShift cluster with:

* Dynamic volume provisioning (e.g. using GlusterFS)
* Red Hat SSO (Keycloak) running with a suitable realm (e.g. `openrisknet`)

## Deploy

### new project
```
oc new-project jupyter
```

### Build Jupyter images
```
oc create -f https://raw.githubusercontent.com/jupyter-on-openshift/jupyter-notebooks/master/images.json
```
This takes about 15 mins.

Patch the build configs to enable JupyterLab support:
```
oc patch bc/s2i-minimal-notebook --patch '{"spec":{"resources":{"limits":{"memory":"3Gi"}}}}'
oc patch bc/s2i-scipy-notebook --patch '{"spec":{"resources":{"limits":{"memory":"3Gi"}}}}'
oc patch bc/s2i-tensorflow-notebook --patch '{"spec":{"resources":{"limits":{"memory":"3Gi"}}}}'

oc set env bc/s2i-minimal-notebook JUPYTER_INSTALL_LAB=true

oc start-build s2i-minimal-notebook
oc start-build s2i-scipy-notebook
oc start-build s2i-tensorflow-notebook
```
The images will be rebuilt. Wait for this to finish.

TODO: can this be improved so that the Jupyter images only need to be built once?

## Build JupyterHub image
Deploy the s2i builder for the JupyterHub image: 
```
oc create -f https://raw.githubusercontent.com/jupyter-on-openshift/jupyterhub-quickstart/master/images.json
```
Wait for the build to complete. Takes about 3 mins.

### Set up SSO

In Keycloak go to the appropriate realm (e.g. `openrisknet`) and add `jupyterhub` as a new client.
Specify `confidential` as the `Access Type`.

### Deploy JupyterHub templates

Deploy the JupyterHub templates:
```
oc create -f https://raw.githubusercontent.com/jupyter-on-openshift/jupyterhub-quickstart/master/templates.json
```

### Edit templates

These templates need to be edited as the liveness and readiness probes are not tolerant of slow starting containers and after
they have failed seems to cause further problems (e.g. patching the deployment config once it is running to modify these probes is too late).

For the `jupyterhub-deployer` template modify the Deployment config for the jupyterhub-db to add these to the livenessProbe:
```
failureThreshold: 10
initialDelaySeconds: 60
periodSeconds: 15
timeoutSeconds: 1
```
and this or the readinessProbe:
```
failureThreshold: 3
initialDelaySeconds: 90
periodSeconds: 30
timeoutSeconds: 1
```

Modify the other templates similarly if they are being used.

TODO: push those changes back to Graham's project to avoid this being necessary.

TODO: look into deploying Jupyterhub into an existing Postgres database (e.g. the one in the openrisknet-infra project). 

### JupyterHub Configuration

Create the jupyterhub_config.py configuration file:
```
c.JupyterHub.spawner_class = 'wrapspawner.ProfilesSpawner'

c.KubeSpawner.start_timeout = 180
c.KubeSpawner.http_timeout = 120

c.KubeSpawner.environment = dict(JUPYTER_ENABLE_LAB='true')

c.ProfilesSpawner.profiles = [
    (
        "Minimal Notebook (CentOS 7 / Python 3.5)",
        's2i-minimal-notebook',
        'kubespawner.KubeSpawner',
        dict(singleuser_image_spec='s2i-minimal-notebook:3.5')
    ),
    (
        "SciPy Notebook (CentOS 7 / Python 3.5)",
        's2i-scipy-notebook',
        'kubespawner.KubeSpawner',
        dict(singleuser_image_spec='s2i-scipy-notebook:3.5')
    ),
    (
        "Tensorflow Notebook (CentOS 7 / Python 3.5)",
        's2i-tensorflow-notebook',
        'kubespawner.KubeSpawner',
        dict(singleuser_image_spec='s2i-tensorflow-notebook:3.5')
    )
]

# authentication

import os
os.environ['OAUTH2_TOKEN_URL'] = 'https://sso.prod.openrisknet.org/auth/realms/openrisknet/protocol/openid-connect/token' 
os.environ['OAUTH2_AUTHORIZE_URL'] = 'https://sso.prod.openrisknet.org/auth/realms/openrisknet/protocol/openid-connect/auth' 
os.environ['OAUTH2_USERDATA_URL'] = 'https://sso.prod.openrisknet.org/auth/realms/openrisknet/protocol/openid-connect/userinfo'
os.environ['OAUTH2_TLS_VERIFY'] = '1'
os.environ['OAUTH2_USERNAME_KEY'] = 'preferred_username'

from oauthenticator.generic import GenericOAuthenticator
c.JupyterHub.authenticator_class = GenericOAuthenticator
c.OAuthenticator.client_id = 'jupyterhub'
c.OAuthenticator.client_secret = '<client-secret>'
c.OAuthenticator.oauth_callback_url = 'https://jupyterhub-jupyter.prod.openrisknet.org/hub/oauth_callback' 
c.OAuthenticator.tls_verify = True
c.OAuthenticator.login_service = 'OpenRiskNet SSO'

# storage
c.KubeSpawner.user_storage_pvc_ensure = True
c.KubeSpawner.pvc_name_template = '%s-nb-{username}' % c.KubeSpawner.hub_connect_ip
c.KubeSpawner.user_storage_capacity = '1Gi'
c.KubeSpawner.volumes = [
    {
        'name': 'data',
        'persistentVolumeClaim': {
            'claimName': c.KubeSpawner.pvc_name_template
        }
    }
]
c.KubeSpawner.volume_mounts = [
    {
        'name': 'data',
        'mountPath': '/opt/app-root/src'
    }
]
```
You must replace the correct value for the `c.OAuthenticator.client_secret` property, and maybe some other values. 

TODO: describe the contents of this file.

TODO: work our how to change the login message from "Sign in with GenericOAUth2" to something more meaningful.

TODO: work out how to specify the need for specific role(s) for authorisation.

### Deploy

Deploy it using:
```
oc new-app --template jupyterhub-deployer --param JUPYTERHUB_CONFIG="`cat jupyterhub_config.py`"
```

## Delete
Delete the deployment (imagestreams will remain):
```
oc delete all,configmap,pvc,serviceaccount,rolebinding --selector app=jupyterhub
```

Or delete everything:
```
oc delete project jupyter
```

## Database backups
These example templates provide backups of your JupyterHub database
using the Informatics Matters backup container image.

>   For the following recipe to work the JupyterHub database must be configured
    to permit remote access to the admin account, i.e. a password must be
    provided in the database Pod's `POSTGRESQL_ADMIN_PASSWORD` environment
    variable. If a password is not set you can run `psql -U postgres` from
    within the Pod and change the password with the `\password` meta command
    of psql.

-   [PVC](backup-pvc.yaml)
-   [CronJob](backup.yaml)

The backup image provides rich control of actions
through a number of container environment variables.

The backup essentially creates a compressed SQL file from the
result of running a `pg_dumpall` command.

The example template has a backup schedule that results in the
backup running at **00:07** each day whilst maintaining a history of
**4** of the most recent backups.

>   If you want to change the total number of backups that are held set the
    `BACKUP_COUNT` parameter, with `-p BACKUP_COUNT=<n>`.

To deploy the backup, using an OpenShift login on the Jupyter project,
first create the storage and then start the CronJob. You will need
the admin password of the database and (optionally) the host and
admin user (defaulted to `jupyterhub-db` and `jupyterhub` respectively).

So, if the admin password is `xyz123` and you're using the default Jupyter
database host service name and admin username you can deploy with the
following sequence of commands, after logging onto OpenShift as an admin user:

```
oc project jupyter
oc process -f backup-pvc.yaml | oc create -f -
oc process -f backup.yaml -p PGADMINPASS=xyz123 | oc create -f -
```

Or, with a different host, the backup can be deployed with: -

```
oc process -f backup.yaml -p PGADMINPASS=xyz123 \
    -p PGHOST=jupyterhub-db | oc create -f -
```

### Database recovery
A recovery image can be launched using the example Job template.

-   [Job](recovery.yaml)

This template, which has the same database user parameters as the backup
template, and attempts to recover the database from the latest backup can
be run with the following command: -

```
oc process -f recovery.yaml -p PGADMINPASS=xyz123 \
    -p PGHOST=jupyterhub-db | oc create -f -
```

>   It is important to ensure that the backup CronJob is not running
    or will not run when you recover the database.
 