# ConfigMaps

## Subtopics:
   * ConfigMap object
   * ConfigMaps and Pods
   * ConfigMaps usage
   * Immutable ConfigMaps

## ConfigMap object

   ConfigMap is an API object used to store non-confidential data in key-value pairs. There are different approaches to using ConfigMaps with Pods, such as environment variables, command-line arguments, or configuration files in a mounted volume.  
   The main purpose of ConfigMap is to store configuration for other Kubernetes objects to use. Instead of `spec` field ConfigMap has `data` and `binaryData` fields. These fields use key-value pairs as their values and both of them are optional. The `data` field contains UTF-8 strings, while the `binaryData` contains Base64-encoded strings. From Kubernetes v1.19, you can add the immutable field to ConfigMap, creating immutable ConfigMap.

   ---

   > A ConfigMap is not designed to hold large chunks of data. The data stored in a ConfigMap cannot exceed 1 MiB. If you need to store settings that are larger than this limit, you may want to consider mounting a volume or use a separate database or file service.

   ---

   > ConfigMap does not provide secrecy or encryption. If the data you want to store are confidential, use a Secret rather than a ConfigMap, or use additional (third party) tools to keep your data private.

   ---

   > The name of a ConfigMap must be a valid DNS subdomain name.

## ConfigMaps and Pods

   To use ConfigMap with Pod, you need refer later via configuration in YAML file. The Pod and the ConfigMap must be in one namespace. Example of ConfigMap below:

   ```yaml
  apiVersion: v1
  kind: ConfigMap
  metadata:
    name: game-demo
  data:
    # property-like keys; each key maps to a simple value
    player_initial_lives: "3"
    ui_properties_file_name: "user-interface.properties"

  # file-like keys
  game.properties: |
    enemy.types=aliens,monsters
    player.maximum-lives=5    
  user-interface.properties: |
    color.good=purple
    color.bad=yellow
    allow.textmode=true    
   ```

   There are four different ways to use a ConfigMap for conteiner configuration inside a Pod:
   * Inside a container command and args
   * Environment variables for a container
   * Add a file in read-only volume, for the application
   * Write code to run inside the Pod that uses the Kubernetes API to read a ConfigMap

   Pod example for ConfigMap above:

   ```yaml
  apiVersion: v1
  kind: Pod
  metadata:
    name: configmap-demo-pod
  spec:
    containers:
      - name: demo
        image: alpine
        command: ["sleep", "3600"]
        env:
          # Define the environment variable
          - name: PLAYER_INITIAL_LIVES # Notice that the case is different here
                                     # from the key name in the ConfigMap.
            valueFrom:
              configMapKeyRef:
                name: game-demo           # The ConfigMap this value comes from.
                key: player_initial_lives # The key to fetch.
          - name: UI_PROPERTIES_FILE_NAME
            valueFrom:
              configMapKeyRef:
                name: game-demo
                key: ui_properties_file_name
        volumeMounts:
        - name: config
          mountPath: "/config"
          readOnly: true
    volumes:
      # You set volumes at the Pod level, then mount them into containers inside that Pod
      - name: config
        configMap:
        # Provide the name of the ConfigMap you want to mount.
          name: game-demo
          # An array of keys from the ConfigMap to create as files
          items:
          - key: "game.properties"
            path: "game.properties"
          - key: "user-interface.properties"
            path: "user-interface.properties"
   ```

   A ConfigMap doesn't differentiate between single line property values and multi-line file-like values. What matters is how Pods and other objects consume those values.  

   For this example, defining a volume and mounting it inside the `demo` container as `/config` creates two files, `/config/game.properties` and `/config/user-interface.properties`, even though there are four keys in the ConfigMap. This is because the Pod definition specifies an `items` array in the `volumes` section. If you omit the `items` array entirely, every key in the ConfigMap becomes a file with the same name as the key, and you get 4 files.

## ConfigMaps usage

  ### Using ConfigMap consuption in a volume in a pod:

  *  Create a ConfigMap or use an existing one. Multiple Pods can reference the same ConfigMap.
  * Modify your Pod definition to add a volume under `.spec.volumes[].` Name the volume anything, and have a `.spec.volumes[].configMap.name` field set to reference your ConfigMap object.
  * Add a `.spec.containers[].volumeMounts[]` to each container that needs the ConfigMap. Specify `.spec.containers[].volumeMounts[].readOnly = true` and `.spec.containers[].volumeMounts[].mountPath` to an unused directory name where you would like the ConfigMap to appear.
  * Modify your image or command line so that the program looks for files in that directory. Each key in the ConfigMap `data` map becomes the filename under `mountPath`.

  Example:

  ```yaml
  apiVersion: v1
  kind: Pod
  metadata:
    name: mypod
  spec:
    containers:
    - name: mypod
      image: redis
      volumeMounts:
      - name: foo
        mountPath: "/etc/foo"
        readOnly: true
    volumes:
    - name: foo
      configMap:
        name: myconfigmap
  ```

  ---

  > Each ConfigMap you want to use needs to be referred to in .spec.volumes.

  ---

  > If there are multiple containers in the Pod, then each container needs its own volumeMounts block, but only one .spec.volumes is needed per ConfigMap.

## Immutable ConfigMaps

   Reasoning behind immutable ConfigMap:
   * protects you from accidental (or unwanted) updates that could cause applications outages
   * improves performance of your cluster by significantly reducing load on kube-apiserver, by closing watches for ConfigMaps marked as immutable.

   Example:

   ```yaml
   apiVersion: v1
   kind: ConfigMap
   metadata:
     ...
   data:
     ...
   immutable: true
   ```

   ---

   > Once a ConfigMap is marked as immutable, it is not possible to revert this change nor to mutate the contents of the data or the binaryData field. You can only delete and recreate the ConfigMap.

   # EOF