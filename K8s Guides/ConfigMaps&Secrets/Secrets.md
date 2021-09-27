# Secrets

   ## Short overview

   Secrets, in many ways, are similar to ConfigMaps, with the biggest difference being in holding confidential data such as passwords, tokens, keys, etc. They are created independently of the pods that use them, so there is less risk of accidental/forceful exposure of confidential data.

   ---

   > Kubernetes Secrets, by default, are stored unencrypted in the API server's underlying data store(etcd). Anyone who is authorized to create a Pod in a namespace can use that access to read any Secret in that namespace, even with indirect access such as the ability to create a Deployment.

   > More details about securing secrets in [documentation](https://kubernetes.io/docs/concepts/configuration/secret/)

   ---

   There are three main ways of using Secrets with a Pod:

   * As files in a volume mounted on one or more of its containers
   * As container environment variable.
   * By the kubelet when pulling images for the Pod.

   > The name of a Secret object must be a valid DNS subdomain name.

   ## Types of Secret

   | BuiltinType                            | Usage                                     |
   | -------------------------------------- |:-----------------------------------------:|
   | Opaque                                 | arbitrary user-defined data               |
   | `kubernetes.io/service-account-token`  | service account token                     |
   | `kubernetes.io/dockercfg`              | serialized ~/.dockercfg                   |
   | `kubernetes.io/dockerconfigjson`       | serialized ~/.docker/config.json file     |
   | `kubernetes.io/basic-auth`             | credentials for basic file authentication |
   | `kubernetes.io/ssh-auth`               | credentials for SSH authentication        |
   | `kubernetes.io/tls`                    | data for a TLS client or server           |
   | `bootstrap.kubernetes.io/token`        | bootstrap token data                      |

   ### **Opaque secrets**

   `Opaque` is the default Secret type if omitted from a Secret configuration file. When you create a Secret using `kubectl`, you will use the `generic` subcommand to indicate an `Opaque` Secret type. For example, the following command creates an empty Secret of type `Opaque`.

   ```sh
   kubectl create secret generic empty-secret
   kubectl get secret empty-secret
   ```
   Output:

   ```
   NAME           TYPE     DATA   AGE
   empty-secret   Opaque   0      2m6s
   ```

   The `DATA` column shows the number of data items stored in the Secret. In this case, 0 means we have created an empty Secret.

   ### **Sevice account token Secrets**

   Main goal of this type of Secret is to store a token that indentifies a service account.

   ```yaml
   apiVersion: v1
   kind: Secret
   metadata:
     name: secret-sa-sample
     annotations:
       kubernetes.io/service-account.name: "sa-name" # annotation must be set to an existing service account name
   type: kubernetes.io/service-account-token
   data:
     # You can include additional key value pairs as you do with Opaque Secrets
     extra: YmFyCg==
   ```

   When creating a `Pod`, Kubernetes automatically creates a service account Secret and automatically modifies your Pod to use this Secret. The service account token Secret contains credentials for accessing the API.

   ### **Docker config Secrets**

   You can config Secrets via Docker, with one of the following `type` values to store the credentials for accessing a Docker registry for images:

   * `kubernetes.io/dockercfg`
   * `kubernetes.io/dockerconfigjson`
   
   Below is an example for a `kubernetes.io/dockercfg` type:

   ```yaml
   apiVersion: v1
   kind: Secret
   metadata:
     name: secret-dockercfg
   type: kubernetes.io/dockercfg
   data: # data must contain `/dockerconfigjson` key in wich file is provided as a base64 encoded string
     .dockercfg: |
           "<base64 encoded ~/.dockercfg file>"
   ```

   > Instead of base64 encoding, you can choose to use the `stringData` filed instead.

   When you create these types of Secrets using a manifest, the API server checks whether the expected key does exists in the data field, and it verifies if the value provided can be parsed as a valid JSON. When you create these types of Secrets using a manifest, the API server checks whether the expected key does exists in the `data` field, and it verifies if the value provided can be parsed as a valid JSON.

   When you do not have a Docker config file, or you want to use kubectl to create a Docker registry Secret, you can do:

   ```shell
   kubectl create secret docker-registry secret-tiger-docker \
   --docker-username=tiger \
   --docker-password=pass113 \
   --docker-email=tiger@acme.com
   ```

   This command creates a Secret of type `kubernetes.io/dockerconfigjson`. If you dump the .dockerconfigjson content from the data field, you will get the following JSON content

  ```json
    {
      "auths": {
        "https://index.docker.io/v1/": {
          "username": "tiger",
          "password": "pass113",
          "email": "tiger@acme.com",
          "auth": "dGlnZXI6cGFzczExMw=="
        }
      }
    }
   ```

   ### **Basic authentication Secret**

   The `kubernetes.io/basic-auth` type is provided for storing credentials needed for basic authentication. The `data` field must contain following two keys:

   * `username`
   * `password` > can be pass or token

   Example:

   ```yaml
   apiVersion: v1
   kind: Secret
   metadata:
     name: secret-basic-auth
   type: kubernetes.io/basic-auth
   stringData:
     username: admin
     password: t0p-Secret
   ```

   You can create an `Opaque` for credentials used for basic authentication.

   ---

   > However, using the builtin Secret type helps unify the formats of your credentials and the API server does verify if the required keys are provided in a Secret configuration.

   ### **SSH authentication secrets**

   The builtin type `kubernetes.io/ssh-auth` is provided for storing data used in SSH authentication. 

   Example:

   ```yaml
   apiVersion: v1
   kind: Secret
   metadata:
     name: secret-ssh-auth
   type: kubernetes.io/ssh-auth
   data: # a key-value pair must be provided in ssh-privatekey
     ssh-privatekey: |
             MIIEpQIBAAKCAQEAulqb/Y ...
   ```

   You can create an Opaque for credentials used for SSH authentication.

   ---

   > However, using the builtin Secret type helps unify the formats of your credentials and the API server does verify if the required keys are provided in a Secret configuration.

   > Caution: SSH private keys do not establish trusted communication between an SSH client and host server on their own. A secondary means of establishing trust is needed to mitigate "man in the middle" attacks, such as a known_hosts file added to a ConfigMap.

   ### **TLS secrets**

   Kubernetes provides a builtin Secret type kubernetes.io/tls for storing a certificate and its associated key that are typically used for TLS. This data is primarily used with TLS termination of the Ingress resource, but may be used with other resources or directly by a workload.

   Example:

   ```yaml
   apiVersion: v1
   kind: Secret
   metadata:
     name: secret-tls
   type: kubernetes.io/tls
   data: # tls.crt and tls.key must be provided
     tls.crt: |
           MIIC2DCCAcCgAwIBAgIBATANBgkqh ...
     tls.key: |
           MIIEpgIBAAKCAQEA7yn3bRHQ5FHMQ ...
   ```

   You can create an `Opaque` for credentials used for TLS server and/or client.

   > However, using the builtin Secret type helps ensure the consistency of Secret format in your project.

   When creating a TLS Secret using `kubectl`, you can use the `tls` subcommand as shown in the following example:

   ```shell
   kubectl create secret tls my-tls-secret \
     --cert=path/to/cert/file \
     --key=path/to/key/file
   ```

   The public/private key pair must exist beforehand. The public key certificate for --cert must be .PEM encoded (Base64-encoded DER format), and match the given private key for --key. The private key must be in what is commonly called PEM private key format, unencrypted. In both cases, the initial and the last lines from PEM (for example, --------BEGIN CERTIFICATE----- and -------END CERTIFICATE---- for a certificate) are not included.

   ## Creating, editing, using Secrets

   ### **Creating Secrets**

   * create Secret using kubectl command
   * create Secret from config YAML file
   * create Secret using [kustomize](https://kubernetes.io/docs/tasks/configmap-secret/managing-secret-using-kustomize/)

   ### **Editing Secrets**

   You can edit Secret with following command: `kubectl edit secrets mysecret`

   ### **Using Secrets**
   
   Secrets can be mounted as data volumes or exposed as environment variables to be used by a container in a Pod. Secrets can also be used by other parts of the system, without being directly exposed to the Pod. For example, Secrets can hold credentials that other parts of the system should use to interact with external systems on your behalf.

   > Due to amount of information and its value - better to read [documentation](https://kubernetes.io/docs/concepts/configuration/secret/#using-secrets)

   ## Immutable Secrets

   Immutable Secrets and Immutable ConfigMaps are in nutshell - one thing. They have same function and protect from same problems. This feature is controlled by the `ImmutableEphemeralVolumes` feature gate, which is enabled by default since v1.19. You can create an immutable Secret by setting the `immutable` field to `true`.

   Example:

   ```yaml
   apiVersion: v1
   kind: Secret
   metadata:
     ...
   data:
     ...
   immutable: true
   ```

   > Once a Secret or ConfigMap is marked as immutable, it is not possible to revert this change nor to mutate the contents of the data field. You can only delete and recreate the Secret. 

   ## Secret and Pod lifetime interaction

   When a Pod is created by calling the Kubernetes API, there is no check if a referenced secret exists. Once a Pod is scheduled, the kubelet will try to fetch the secret value. If the secret cannot be fetched because it does not exist or because of a temporary lack of connection to the API server, the kubelet will periodically retry. It will report an event about the Pod explaining the reason it is not started yet. Once the secret is fetched, the kubelet will create and mount a volume containing it. None of the Pod's containers will start until all the Pod's volumes are mounted.

   ---

   # EOF