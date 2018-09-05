---
layout: layout.pug
navigationTitle:  Deploying Services using a Custom Marathon with Security Features
title: Deploying Services using a Custom Marathon with Security Features
menuWeight: 40
excerpt: Using an advanced, non-native instance of Marathon
enterprise: true

---

This topic describes how to deploy a non-native instance of Marathon with isolated roles, reservations, quotas, and security features. The advanced non-native Marathon procedure should only be used if you require [secrets](/1.12/security/ent/secrets/) or fine-grain ACLs, otherwise use the [basic procedure](/1.12/deploying-services/marathon-on-marathon/basic/).

For this procedure, we are assuming that you have obtained an enterprise version of Marathon from your sales representative (<sales@mesosphere.io>). This custom tarball contains Marathon plus secrets and auth plugins. These additional plugins allow you to use secrets and fine-grained access control in your non-native Marathon folder hierarchy.

**Prerequisites:**

-  DC/OS and DC/OS CLI [installed](/1.12/installing/oss/).
-  [DC/OS Enterprise CLI 0.4.14 or later](/1.12/cli/enterprise-cli/#ent-cli-install).
-  A private Docker registry that each private DC/OS agent can access over the network. You can follow [these](/1.12/deploying-services/private-docker-registry/) instructions for how to set up in Marathon, or use another option such as [DockerHub](https://hub.docker.com/), [Amazon EC2 Container Registry](https://aws.amazon.com/ecr/), and [Quay](https://quay.io/)).
-  Custom non-native Marathon (version 0.11.0 or newer) image [deployed in your private Docker registry](/1.12/deploying-services/private-docker-registry#tarball-instructions). Contact your sales representative or <sales@mesosphere.io> for obtain the enterprise Marathon image file.
-  You must be logged in as a superuser.
-  SSH access to the cluster.

<a name="variables-in-example"></a>
**Variables in this example**

Throughout this page, you will be instructed to invoke commands or take actions that differ according to your setup. For the parts that differ, we are using the `${VARIABLE}` notation, that should be replaced with the appropriate value that fits in your case.

The following table contains all the variables used in this page:

| Variable | Description |
|--------------------------|--------------------------------------------|
| `${MY_USER_ID}` | The username that you use to login to your cluster. That's the username that you were prompted during `dcos cluster setup`. |
| `${AGENT_ID}` | The ID of an agent in your DC/OS cluster. You can obtain an agent ID using the `dcos node` command. |
| `${MESOS_ROLE}` | The name of the [Mesos Role](https://mesos.apache.org/documentation/latest/roles/) that the new Marathon instance will use. This should be a valid [Mesos role name](https://mesos.apache.org/documentation/latest/roles/#invalid-role-names), for example `"marathon_ee"`. |
| `${SERVICE_ACCOUNT_ID}` | The name of the [Service Account](/1.12/security/ent/service-auth/) that Marathon will use to communicate with the other services in DC/OS. The name should include only lowercase alphabet, uppercase alphabet, numbers, `@`, `.`, `\`, `_`, and `-`. For example `"marathon_user_ee"` |
| `${BEGIN_PORT}` - `${END_PORT}` | A port range that will be reserved on the agent, for the custom Marathon instance. Check the output of the `dcos node --json | jq '.[] | {id, ports: .resources.ports}'` command to find out what is a valid range for your agents. |
| `${MARATHON_INSTANCE_NAME}` | The name of your non-native instance. This should be a valid [Marathon service name](https://mesosphere.github.io/marathon/docs/application-basics.html), for example `"mom_ee"`. |
| `${SERVICE_ACCOUNT_SECRET}` | The path of the secret in the [Secret Store](/1.12/security/ent/secrets/) that holds the private key that Marathon will use along with the `${SERVICE_ACCOUNT_ID}` account to authenticate on DC/OS. This the name **must not** contain the character `/`. For example `"marathon_user_ee_secret"` |
| `${DOCKER_REGISTRY_SECRET}` | The path of the secret in the [Secret Store](/1.12/security/ent/secrets/) that holds the credentials for fetching the Marathon Docker image from the private registry. This the name **must not** contain the character `/`. For example `"registry_secret"`. |
| `${PRIVATE_KEY}` | The path to the private key file (in your local file system). Feel free to pick any file name ending with `.pem`. |
| `${PUBLIC_KEY}` | The path to the public key file (in your local file system). Feel free to pick any file name ending with `.pem` |
| `${MARATHON_IMAGE}` | The name of the Marathon image **in your private repository**, for example `private-repo/marathon-dcos-ee`. |
| `${MARATHON_TAG}` | The Docker image tag of the Marathon version that you want to deploy. For example `v1.5.11_1.10.2` (version 0.11.0 or newer).  |

**Note:** If you are working on a MacOS or Linux machine, you can pre-define all the above variables in your terminal session and just copy and paste the snippets in your terminal:

```
set -a
MY_USER_ID="..."
AGENT_ID="..."
MESOS_ROLE="..."
SERVICE_ACCOUNT_ID="..."
BEGIN_PORT="..."
END_PORT="..."
MARATHON_INSTANCE_NAME="..."
SERVICE_ACCOUNT_SECRET="..."
DOCKER_REGISTRY_SECRET="..."
PRIVATE_KEY="..."
PUBLIC_KEY="..."
MARATHON_IMAGE="..."
MARATHON_TAG="..."
```


# Step 1: Preparation

For the following steps, we are assuming that you have already:

1. Pushed the enterprise image of Marathon to your private registry [(Instructions)](/1.12/deploying-services/private-docker-registry#tarball-instructions), under the name `${MARATHON_IMAGE}:${MARATHON_TAG}`.
1. Stored your private Docker credentials in the secrets store [(Instructions)](/1.12/deploying-services/private-docker-registry/#referencing-private-docker-registry-credentials-in-the-secrets-store-enterprise), under the name `${DOCKER_REGISTRY_SECRET}`.

    **Warning:** The name of the secret should either be in the root path (ex. `/some-secret-name`) or prefixed with the name of your app (ex. `/${MARATHON_INSTANCE_NAME}/some-secret-name`). Failing to do so, will make root Marathon unable to read the secret value and will fail to launch your custom Marathon-on-Marathon instance.

# Step 2: Reserve Resources (Optional) {.tabs}
In this step, we are going to define a [Mesos Role](https://mesos.apache.org/documentation/latest/roles/) for our new Marathon instance, and reserve some dedicated resources for it. This step is optional, but it's recommended if you want your new marathon instance to have a minimum resource guarantee for it's tasks.

**Important:** If you choose not to reserve resources, you should use `MESOS_ROLE="*"` in any further occurrence of this variable.

You can choose between [static](#static-reservations) reservations, [dynamic](#dynamic-reservations) reservations or [quotas](#quotas).

| Method | Description |
|--------|-------------|
| Static | Dedicates entire agent(s) to the new Marathon |
| Dynamic | Dedicates a portion of agent(s) resources to the new Marathon |
| Quotas | Ensures a minimum resource allocation to the new Marathon, regardless of where it's found. |

## Static Reservations
A _static_ reservation is achieved by changing the default Mesos Role of a particular agent in order to dedicate all of its resources to the new Marathon instance.

**Warning:** This procedure kills all running tasks on your node.

1.  [SSH](/1.12/administering-clusters/sshcluster/) to your private agent node.

   ```bash
   dcos node ssh --master-proxy --mesos-id=${AGENT_ID}
   ```
1.  Navigate to `/var/lib/dcos` and create a file named `mesos-slave-common` with these contents, where `${MESOS_ROLE}` is the name of your role.

    ```bash
    MESOS_DEFAULT_ROLE="${MESOS_ROLE}"
    ```
1.  Stop the private agent node:

    ```bash
    sudo sh -c 'systemctl kill -s SIGUSR1 dcos-mesos-slave && systemctl stop dcos-mesos-slave'
    ```

1.  Add the node back to your cluster.

    1.  Reload the systemd configuration.

        ```bash
        ﻿⁠⁠sudo systemctl daemon-reload
        ```

    1.  Remove the `latest` metadata pointer on the agent node:

        ```bash
        ⁠⁠⁠⁠sudo rm /var/lib/mesos/slave/meta/slaves/latest
        ```

    1.  Start your agents with the newly configured attributes and resource specification⁠⁠.

        ```bash
        sudo systemctl start dcos-mesos-slave
        ```

        **Tip:** You can check the status with this command:

        ```bash
        sudo systemctl status dcos-mesos-slave
        ```

1.  Repeat these steps for each additional node.

## Dynamic Reservations
A _dynamic_ [reservation](https://mesos.apache.org/documentation/latest/reservation/#dynamic-reservation) is reserving a portion of an agent's resources to a role. To achieve this we are going to use the Mesos API which will be a less intrusive method to your cluster.

```bash
curl -i -k \
      -H "Authorization: token=`dcos config show core.dcos_acs_token`" \
      -H "Content-Type: application/json" \
      -H "Accept: application/json" \
      -d '{
  "type": "RESERVE_RESOURCES",
  "reserve_resources": {
    "agent_id": {
      "value": "'${AGENT_ID}'"
    },
    "resources": [
      {
        "type": "SCALAR",
        "name": "cpus",
        "reservation": {
          "principal": "'${MY_USER_ID}'"
        },
        "role": "'${MESOS_ROLE}'",
        "scalar": {
          "value": 1.0
        }
      },
      {
        "type": "SCALAR",
        "name": "mem",
        "reservation": {
          "principal": "'${MY_USER_ID}'"
        },
        "role": "'${MESOS_ROLE}'",
        "scalar": {
          "value": 512.0
        }
      },
      {
        "type": "RANGES",
        "name": "ports",
        "reservation": {
          "principal": "'${MY_USER_ID}'"
        },
        "role": "'${MESOS_ROLE}'",
        "ranges": {
          "begin": "['${BEGIN_PORT}]'",
          "end": "['${END_PORT}']"
        }
      }
    ]
  }
}' \
      -X POST "`dcos config show core.dcos_url`/mesos/api/v1"
```

## Quotas
Since both _static_ and _dynamic_ reservations are pinned to a particular agent, they are susceptible to the individual agents failing. This can either be mitigated by reserving enough resources on multiple agent nodes, or by using [Quotas](https://mesos.apache.org/documentation/latest/quota/) instead.

A _Quota_ is simply a mechanism for guaranteeing that a role will receive a certain minimum amount of resources.

**Warning:** You can only use `SCALAR` resources with _Quotas_. This means that if you want to reserve a port range you have to use _static_ or _dynamic Reservations_.

```bash
curl -i -k \
      -H "Authorization: token=`dcos config show core.dcos_acs_token`" \
      -H "Content-Type: application/json" \
      -H "Accept: application/json" \
      -d '{
  "role": "'${MESOS_ROLE}'",
  "guarantee": [
    {
      "type": "SCALAR",
      "name": "cpus",
      "scalar": { "value": 1.0 }
    },
    {
      "type": "SCALAR",
      "name": "mem",
      "scalar": { "value": 512.0 }
    }
  ]
}' \
      -X POST "`dcos config show core.dcos_url`/mesos/quota"
```


# Step 3: Create a Marathon Service Account
In this step, a Marathon [Service Account](/1.12/security/ent/service-auth/) is created. This account will be used by Marathon to authenticate on the rest of DC/OS components. The permissions later given to this account are going to define what Marathon is allowed to do.

Depending on your [security mode](/1.12/security/ent/#security-modes), a Marathon Service Account is either optional or required.

| Security Mode | Marathon Service Account |
|---------------|--------------------------|
| Permissive | Optional |
| Strict | Required |

1.  Create a 2048-bit RSA private-public key pair (`${PRIVATE_KEY}` and `${PUBLIC_KEY}`) and save each value into a separate file within the current directory.

    With the following command, we create a pair of private and public keys. The public key will be used to create the Marathon service account. The private key we will store in the [secret store](/1.12/security/ent/secrets/) and later passed to Marathon so it can authorize itself using this account. 

    ```bash
    dcos security org service-accounts keypair ${PRIVATE_KEY} ${PUBLIC_KEY}
    ```

1.  Create a new service account called `${SERVICE_ACCOUNT_ID}`, with the public key specified (`${PUBLIC_KEY}`).

    ```bash
    dcos security org service-accounts create -p ${PUBLIC_KEY} -d "Marathon-on-Marathon User" ${SERVICE_ACCOUNT_ID}
    ```


# Step 4: Create a Service Account Secret
In this step, a secret is created for the Marathon service account and stored in the Secret Store.

  The secret created (`${SERVICE_ACCOUNT_SECRET}`) will contain the private key (`${PRIVATE_KEY}`) and the name of the service account (`${SERVICE_ACCOUNT_ID}`). This information will be used by Marathon to authenticate against DC/OS.

  ## {.tabs}

  ### Permissive

  ```bash
  dcos security secrets create-sa-secret ${PRIVATE_KEY} ${SERVICE_ACCOUNT_ID} ${SERVICE_ACCOUNT_SECRET}
  ```

  ### Strict

  ```bash
  dcos security secrets create-sa-secret --strict ${PRIVATE_KEY} ${SERVICE_ACCOUNT_ID} ${SERVICE_ACCOUNT_SECRET}
  ```

  #### Recommendations

  * Make sure that your secret is in place, using the following command:

     ```bash
     dcos security secrets list /
     ```

  *  Review your secret to ensure that it contains the correct service account ID, private key, and `login_endpoint` URL. If you're in `strict` it should be HTTPS, in `permissive` mode it should be HTTP. If the URL is incorrect, try [upgrading the DC/OS Enterprise CLI](/1.12/cli/enterprise-cli/#ent-cli-upgrade), deleting the secret, and recreating it. 

      You can use this commands to view the contents (requires [jq 1.5 or later](https://stedolan.github.io/jq/download) installed):

      ```bash
      dcos security secrets get ${SERVICE_ACCOUNT_SECRET} --json | jq -r .value | jq
      ```

  *  Delete the private key file from your file system to prevent bad actors from using the private key to authenticate into DC/OS.


# Step 5: Assign Permissions (Strict mode only)
In this step, permissions are assigned to the Marathon-on-Marathon instance. Permissions are required in strict mode and are ignored in other security modes.

| Security Mode | Permissions |
|---------------|----------------------|
| Permissive | Not available |
| Strict | Required |

All CLI commands can also be executed via the [IAM API](/1.12/security/ent/iam-api).

Grant service account `${SERVICE_ACCOUNT_ID}` permission to launch Mesos tasks that will execute as Linux user `nobody`.

To allow executing tasks as a different Linux user, replace `nobody` with that user's Linux user ID. For example, to launch tasks as Linux user `bob`, replace `nobody` with `bob` below.

Note that the `nobody` and `root` users exist on all agents by default, but if a custom `bob` user is specified it has to be manually created (using the `adduser` or a similar utility) on every agent that tasks can be executed on.

```bash
dcos security org users grant ${SERVICE_ACCOUNT_ID} dcos:mesos:master:task:user:nobody create --description "Tasks can execute as Linux user nobody"
dcos security org users grant ${SERVICE_ACCOUNT_ID} dcos:mesos:master:framework:role:${MESOS_ROLE} create --description "Controls the ability of ${MESOS_ROLE} to register as a framework with the Mesos master"
dcos security org users grant ${SERVICE_ACCOUNT_ID} dcos:mesos:master:reservation:role:${MESOS_ROLE} create --description "Controls the ability of ${MESOS_ROLE} to reserve resources"
dcos security org users grant ${SERVICE_ACCOUNT_ID} dcos:mesos:master:volume:role:${MESOS_ROLE} create --description "Controls the ability of ${MESOS_ROLE} to access volumes"
dcos security org users grant ${SERVICE_ACCOUNT_ID} dcos:mesos:master:reservation:principal:${SERVICE_ACCOUNT_ID} delete --description "Controls the ability of ${SERVICE_ACCOUNT_ID} to reserve resources"
dcos security org users grant ${SERVICE_ACCOUNT_ID} dcos:mesos:master:task:app_id:/ create --description "Controls the ability to launch tasks"
dcos security org users grant ${SERVICE_ACCOUNT_ID} dcos:mesos:master:volume:principal:${SERVICE_ACCOUNT_ID} delete --description "Controls the ability of ${SERVICE_ACCOUNT_ID} to access volumes"
```


# Step 6: Install a Non-Native Marathon Instance with Assigned Role {.tabs}
In this step, a non-native Marathon instance is installed on DC/OS with the Mesos role assigned.

1.  Create a custom JSON config that will be used to install the custom non-native Marathon instance. The JSON file contents vary according to your [security mode](/1.12/security/ent/#security-modes).

    Make sure you replace all the `${VARIABLES}` in the JSON file with the correct values for your case.

    Alternatively, if you have exposed the values for all these variables in your terminal session (as explained in the [Variables Section](#variables-in-example)) you can use the following command to automatically replace all the variables with the correct values. It will save the result file in `marathon-filled.json`:

    ```bash
    perl -p -e 's/\$\{(\w+)\}/(exists $ENV{$1}?$ENV{$1}:"missing variable $1")/eg' < marathon.json > marathon-filled.json
    ```

    ## {.tabs}

    ### Permissive

    Use the following JSON template if you are running a cluster in `permissive` security mode. Don't forget to **replace** all the environment variables that follow the `${VARIABLES}` format:

    ```json
    {
      "id":"/${MARATHON_INSTANCE_NAME}",
      "cmd":"cd $MESOS_SANDBOX && LIBPROCESS_PORT=$PORT1 && /marathon/bin/start --default_accepted_resource_roles \"*,${MESOS_ROLE}\" --enable_features \"vips,task_killing,external_volumes,secrets,gpu_resources\" --framework_name ${MARATHON_INSTANCE_NAME} --hostname $LIBPROCESS_IP --http_port $PORT0 --master zk://master.mesos:2181/mesos --max_instances_per_offer 1 --mesos_leader_ui_url /mesos --mesos_role ${MESOS_ROLE}  --zk zk://master.mesos:2181/universe/${MARATHON_INSTANCE_NAME} --mesos_user nobody --mesos_authentication --mesos_authentication_principal ${SERVICE_ACCOUNT_ID}",
      "user":"nobody",
      "cpus":2,
      "mem":4096,
      "disk":0,
      "instances":1,
      "constraints":[  
         [  
            "hostname",
            "UNIQUE"
         ]
      ],
      "container":{  
         "type": "MESOS",
         "docker":{  
            "image":"${MARATHON_IMAGE}:${MARATHON_TAG}",
            "pullConfig": {
                "secret": "docker-registry-credential"
            }
         },
         "volumes":[  
            {  
               "containerPath":"/opt/mesosphere",
               "hostPath":"/opt/mesosphere",
               "mode":"RO"
            }
         ]
      },
      "env":{  
         "JVM_OPTS":"-Xms256m -Xmx2g",
         "DCOS_STRICT_SECURITY_ENABLED":"false",
         "DCOS_SERVICE_ACCOUNT_CREDENTIAL_TOFILE":{  
            "secret":"service-credential"
         },
         "MESOS_AUTHENTICATEE":"com_mesosphere_dcos_ClassicRPCAuthenticatee",
         "MESOS_MODULES":"file:///opt/mesosphere/etc/mesos-scheduler-modules/dcos_authenticatee_module.json",
         "MESOS_NATIVE_JAVA_LIBRARY":"/opt/mesosphere/lib/libmesos.so",
         "MESOS_VERBOSE":"true",
         "GLOG_v":"2",
         "PLUGIN_ACS_URL":"http://master.mesos",
         "PLUGIN_AUTHN_MODE":"dcos/jwt+anonymous",
         "PLUGIN_FRAMEWORK_TYPE":"marathon"
      },
      "healthChecks":[  
         {  
            "path":"/ping",
            "protocol":"HTTP",
            "portIndex":0,
            "gracePeriodSeconds":1800,
            "intervalSeconds":10,
            "timeoutSeconds":5,
            "maxConsecutiveFailures":3,
            "ignoreHttp1xx":false
         }
      ],
      "secrets":{  
         "service-credential":{  
            "source":"${SERVICE_ACCOUNT_SECRET}"
         },
         "docker-registry-credential": {
            "source":"${DOCKER_REGISTRY_SECRET}"
         }
      },
      "labels":{  
         "DCOS_SERVICE_NAME":"${MARATHON_INSTANCE_NAME}",
         "DCOS_SERVICE_PORT_INDEX":"0",
         "DCOS_SERVICE_SCHEME":"http"
      },
      "portDefinitions":[  
         {  
            "port":0,
            "name":"http"
         },
         {  
            "port":0,
            "name":"libprocess"
         }
      ]
    }
    ```

    ### Strict

    Use the following JSON template if you are running a cluster in `strict` security mode. Don't forget to **replace** all the environment variables that follow the `${VARIABLES}` format:

    ```json
    {  
      "id":"/${MARATHON_INSTANCE_NAME}",
      "cmd":"cd $MESOS_SANDBOX && LIBPROCESS_PORT=$PORT1 && /marathon/bin/start --default_accepted_resource_roles \"*,${MESOS_ROLE}\" --enable_features \"vips,task_killing,external_volumes,secrets,gpu_resources\" --framework_name ${MARATHON_INSTANCE_NAME} --hostname $LIBPROCESS_IP --http_port $PORT0 --master zk://master.mesos:2181/mesos --max_instances_per_offer 1 --mesos_leader_ui_url /mesos --mesos_role ${MESOS_ROLE}  --zk zk://master.mesos:2181/universe/${MARATHON_INSTANCE_NAME} --mesos_user nobody --mesos_authentication --mesos_authentication_principal ${SERVICE_ACCOUNT_ID}",
      "user":"nobody",
      "cpus":2,
      "mem":4096,
      "disk":0,
      "instances":1,
      "constraints":[  
         [  
            "hostname",
            "UNIQUE"
         ]
      ],
      "container":{  
         "type": "MESOS",
         "docker":{  
            "image":"${MARATHON_IMAGE}:${MARATHON_TAG}",
            "pullConfig": {
                "secret": "docker-registry-credential"
            }
         },
         "volumes":[  
            {  
               "containerPath":"/opt/mesosphere",
               "hostPath":"/opt/mesosphere",
               "mode":"RO"
            }
         ]
      },
      "env":{  
         "JVM_OPTS":"-Xms256m -Xmx2g",
         "DCOS_STRICT_SECURITY_ENABLED":"true",
         "DCOS_SERVICE_ACCOUNT_CREDENTIAL_TOFILE":{  
            "secret":"service-credential"
         },
         "MESOS_AUTHENTICATEE":"com_mesosphere_dcos_ClassicRPCAuthenticatee",
         "MESOS_MODULES":"file:///opt/mesosphere/etc/mesos-scheduler-modules/dcos_authenticatee_module.json",
         "MESOS_NATIVE_JAVA_LIBRARY":"/opt/mesosphere/lib/libmesos.so",
         "MESOS_VERBOSE":"true",
         "GLOG_v":"2",
         "PLUGIN_ACS_URL":"https://master.mesos",
         "PLUGIN_AUTHN_MODE":"dcos/jwt",
         "PLUGIN_FRAMEWORK_TYPE":"marathon"
      },
      "healthChecks":[  
         {  
            "path":"/",
            "protocol":"HTTP",
            "portIndex":0,
            "gracePeriodSeconds":1800,
            "intervalSeconds":10,
            "timeoutSeconds":5,
            "maxConsecutiveFailures":3,
            "ignoreHttp1xx":false
         }
      ],
      "secrets":{  
         "service-credential":{  
            "source":"${SERVICE_ACCOUNT_SECRET}"
         },
         "docker-registry-credential": {
            "source":"${DOCKER_REGISTRY_SECRET}"
         }
      },
      "labels":{  
         "DCOS_SERVICE_NAME":"${MARATHON_INSTANCE_NAME}",
         "DCOS_SERVICE_PORT_INDEX":"0",
         "DCOS_SERVICE_SCHEME":"http"
      },
      "portDefinitions":[  
         {  
            "port":0,
            "name":"http"
         },
         {  
            "port":0,
            "name":"libprocess"
         }
      ]
    }
    ```

1.  Deploy the Marathon instance, passing the JSON file you created in the previous step.

    ```bash
    dcos marathon app add marathon-filled.json
    ```

# Step 7: Grant User Access to Non-Native Marathon
By now, your new Marathon instance is accessible only by the DC/OS Superusers. In order to give access to regular users, you need to explicitly give them access permissions, according to your cluster security mode: 

  ## {.tabs}

  ### Permissive

  For each `${USER_ACCOUNT}` that you want to give access, if you are running a cluster in `permissive` security mode, depending on what kind of permissions you want to give:

  - To give **full** permissions to any service (or a group) that runs on the newly created Marathon:

    ```bash
    # Access to the new Marathon service under `/service/<name>` URL
    dcos security org users grant ${USER_ACCOUNT} dcos:adminrouter:service:${MARATHON_INSTANCE_NAME} full
    dcos security org users grant ${USER_ACCOUNT} dcos:service:marathon:${MARATHON_INSTANCE_NAME}:services:/ full

    # Access to the Mesos tasks and their sandbox
    dcos security org users grant ${USER_ACCOUNT} dcos:adminrouter:ops:mesos full
    dcos security org users grant ${USER_ACCOUNT} dcos:adminrouter:ops:slave full

    # (Optionally) Access to the Marathon instance that runs on the root 
    # Marathon and can be controlled via the DC/OS UI
    dcos security org users grant ${USER_ACCOUNT} dcos:adminrouter:service:marathon full
    dcos security org users grant ${USER_ACCOUNT} dcos:service:marathon:marathon:services:/${MARATHON_INSTANCE_NAME} full
    ```

  - To give permissions to an **individual** service (or a group) that runs on the newly created Marathon and is named `${CHILD_SERVICE_NAME}`:

    ```bash
    # Access to the new Marathon service under `/service/<name>` URL
    dcos security org users grant ${USER_ACCOUNT} dcos:adminrouter:service:${MARATHON_INSTANCE_NAME} full
    dcos security org users grant ${USER_ACCOUNT} dcos:service:marathon:${MARATHON_INSTANCE_NAME}:services:/${CHILD_SERVICE_NAME} full

    # Access to the Mesos tasks and their sandbox
    dcos security org users grant ${USER_ACCOUNT} dcos:adminrouter:ops:mesos full
    dcos security org users grant ${USER_ACCOUNT} dcos:adminrouter:ops:slave full

    # (Optionally) Access to the Marathon instance that runs on the root 
    # Marathon and can be controlled via the DC/OS UI
    dcos security org users grant ${USER_ACCOUNT} dcos:adminrouter:service:marathon full
    dcos security org users grant ${USER_ACCOUNT} dcos:service:marathon:marathon:services:/${MARATHON_INSTANCE_NAME} full
    ```

  ### Strict

  For each `${USER_ACCOUNT}` that you want to give access, if you are running a cluster in `strict` security mode, depending on what kind of permissions you want to give:

  - To give **full** permissions to any service (or a group) that runs on the newly created Marathon:

    ```bash
    # Access to the new Marathon service under `/service/<name>` URL
    dcos security org users grant ${USER_ACCOUNT} dcos:adminrouter:service:${MARATHON_INSTANCE_NAME} full
    dcos security org users grant ${USER_ACCOUNT} dcos:service:marathon:${MARATHON_INSTANCE_NAME}:services:/ full

    # Access to the Mesos tasks and their sandbox
    dcos security org users grant ${USER_ACCOUNT} dcos:adminrouter:ops:mesos full
    dcos security org users grant ${USER_ACCOUNT} dcos:adminrouter:ops:slave full
    dcos security org users grant ${USER_ACCOUNT} dcos:mesos:agent:executor:app_id:/ read
    dcos security org users grant ${USER_ACCOUNT} dcos:mesos:agent:framework:role:${MESOS_ROLE} read
    dcos security org users grant ${USER_ACCOUNT} dcos:mesos:agent:sandbox:app_id:/ read
    dcos security org users grant ${USER_ACCOUNT} dcos:mesos:agent:task:app_id:/ read
    dcos security org users grant ${USER_ACCOUNT} dcos:mesos:master:executor:app_id:/ read
    dcos security org users grant ${USER_ACCOUNT} dcos:mesos:master:framework:role:${MESOS_ROLE} read
    dcos security org users grant ${USER_ACCOUNT} dcos:mesos:master:task:app_id:/ read

    # (Optionally) Access to the Marathon instance that runs on the root 
    # Marathon and can be controlled via the DC/OS UI
    dcos security org users grant ${USER_ACCOUNT} dcos:adminrouter:service:marathon full
    dcos security org users grant ${USER_ACCOUNT} dcos:service:marathon:marathon:services:/${MARATHON_INSTANCE_NAME} full
    ```

  - To give permissions to an **individual** service (or a group) that runs on the newly created Marathon and is named `${CHILD_SERVICE_NAME}`:

    ```bash
    # Access to the new Marathon service under `/service/<name>` URL
    dcos security org users grant ${USER_ACCOUNT} dcos:adminrouter:service:${MARATHON_INSTANCE_NAME} full
    dcos security org users grant ${USER_ACCOUNT} dcos:service:marathon:${MARATHON_INSTANCE_NAME}:services:/${CHILD_SERVICE_NAME} full

    # Access to the Mesos tasks and their sandbox
    dcos security org users grant ${USER_ACCOUNT} dcos:adminrouter:ops:mesos full
    dcos security org users grant ${USER_ACCOUNT} dcos:adminrouter:ops:slave full
    dcos security org users grant ${USER_ACCOUNT} dcos:mesos:agent:executor:app_id:/${CHILD_SERVICE_NAME} read
    dcos security org users grant ${USER_ACCOUNT} dcos:mesos:agent:framework:role:${MESOS_ROLE} read
    dcos security org users grant ${USER_ACCOUNT} dcos:mesos:agent:sandbox:app_id:/${CHILD_SERVICE_NAME} read
    dcos security org users grant ${USER_ACCOUNT} dcos:mesos:agent:task:app_id:/${CHILD_SERVICE_NAME} read
    dcos security org users grant ${USER_ACCOUNT} dcos:mesos:master:executor:app_id:/${CHILD_SERVICE_NAME} read
    dcos security org users grant ${USER_ACCOUNT} dcos:mesos:master:framework:role:${MESOS_ROLE} read
    dcos security org users grant ${USER_ACCOUNT} dcos:mesos:master:task:app_id:/${CHILD_SERVICE_NAME} read

    # (Optionally) Access to the Marathon instance that runs on the root 
    # Marathon and can be controlled via the DC/OS UI
    dcos security org users grant ${USER_ACCOUNT} dcos:adminrouter:service:marathon full
    dcos security org users grant ${USER_ACCOUNT} dcos:service:marathon:marathon:services:/${MARATHON_INSTANCE_NAME} full
    ```


# Step 8: Access the Non-Native Marathon Instance
In this step, you log in as a authorized user to the non-native Marathon DC/OS service.

1.  Launch the non-native Marathon interface at: `http://<master-public-ip>/service/${SERVICE_ACCOUNT_ID}/`.

1.  Enter your username and password and click **LOG IN**.

    ![Log in DC/OS](/1.12/img/gui-installer-login-ee.gif)

    You are done!

    ![Marathon on Marathon](/1.12/img/mom-marathon-gui.png)


# Next Steps

- You can configure the DC/OS CLI to interact with your non-native Marathon instance using the following command:
    
    ```sh
    dcos config set marathon.url \
      $(dcos config show core.dcos_url)/service/${MARATHON_INSTANCE_NAME}
    ```

    Now any future `dcos marathon ...` commands will target your new marathon instance.

    To undo this change, use the following command:

    ```sh
    dcos config unset marathon.url
    ```


# Known pitfalls

- When launching docker containers user _nobody_ does not have enough rights to execute the command e.g. stating _nginx_ as _nobody_ will fail because _nobody_ does not have the right  to write logs to /var/log (as nginx needs). 

- User `nobody` has different UIDs on different systems (99 on coreos, 65534 on ubuntu). Depending on the agent's distribution you might need to modify the container image so that UIDs match! (the same goes if you use the user _bob_)

- If using custom users e.g. _bob_, this user has to exist on the agent and within the container.

- When using a new user e.g. _bob_ don't forget to give Marathon service account permissions to run tasks with this user 

