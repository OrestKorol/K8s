# Practical use

   ## Container enviroment variables

   Create a coresponding YAML configuration file:

   ```yaml
   apiVersion: v1
   kind: Secret
   metadata:
     name: mysecret
   type: Opaque
   data:
     USER_NAME: YWRtaW4=
     PASSWORD: MWYyZDFlMmU2N2Rm
   ```

   Apply changes `kubectl apply -f mysecret.yaml`

   Create a Pod which will use created Secret. Use `envFrom` to define all of the Secret's data as container environment variables.

   ```yaml
   apiVersion: v1
   kind: Pod
   metadata:
     name: secret-test-pod
   spec:
     containers:
       - name: test-container
         image: k8s.gcr.io/busybox
         command: [ "/bin/sh", "-c", "env" ]
         envFrom:
        - secretRef:
            name: mysecret
     restartPolicy: Never
   ```

   ## Pod with ssh keys

   Create secret with ssh key by running `kubectl create secret generic ssh-key-secret --from-file=ssh-privatekey=/path/to/.ssh/id_rsa --from-file=ssh-publickey=/path/to/.ssh/id_rsa.pub`  
   The output will be similar to `secret "ssh-key-secret" created`

   ---

   > You can also create a kustomization.yaml with a secretGenerator field containing ssh keys.

   >Think carefully before sending your own ssh keys: other users of the cluster may have access to the secret. Use a service account which you want to be accessible to all the users with whom you share the Kubernetes cluster, and can revoke this account if the users are compromised.

   ---

   Now we need to create pod which will reference created ssh Secret

   ```yaml
   apiVersion: v1
   kind: Pod
   metadata:
     name: secret-test-pod
     labels:
       name: secret-test
   spec:
     volumes:
     - name: secret-volume
       secret:
         secretName: ssh-key-secret
     containers:
     - name: ssh-test-container
       image: mySshImage
       volumeMounts:
       - name: secret-volume
         readOnly: true
         mountPath: "/etc/secret-volume"
   ```
   After applying YAML file we can access key in container at  
   `/etc/secret-volume/ssh-publickey`
   `/etc/secret-volume/ssh-privatekey`

   ## Best practices

   When deploying applications that interact with the Secret API, you should limit access using [authorization policies](https://kubernetes.io/docs/reference/access-authn-authz/authorization/) such as [RBAC](https://kubernetes.io/docs/reference/access-authn-authz/rbac/).

   