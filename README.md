# Working with Kubernetes Secrets and ConfigMaps

Kubernetes has two types of objects that can be used to inject configuration data into a container when it starts up: Secrets and ConfigMaps.  Secrets and ConfigMaps behave similarly in Kubernetes, both in the way they are created, and in that both can be exposed inside a container as either mounted files or volumes, or as environment variables.

We will explore both Secrets and ConfigMaps with a real-world situation:

Consider running the [Official MariaDB container image](https://hub.docker.com/_/mariadb) in Kubernetes.  Some configuration is expected to get the container to run.  The image required an environment variable to be set for one of `MYSQL_ROOT_PASSWORD`, `MYSQL_ALLOW_EMPTY_PASSWORD`, or `MYSQL_RANDOM_ROOT_PASSWORD`, in order to initialize the database.  It also allows for extensions to the MySQL Configuration file `my.cnf` by placing custom config files in `/etc/mysql/conf.d`.

You could build a custom image, setting the environment variables and copying the configuration files into it, thereby creating a bespoke container image. However, it is considered practice to create and use generic images and add configuration to the containers themselves, and this is a perfect use-case for ConfigMaps and Secrets.  The `MYSQL_ROOT_PASSWORD` can be set in a Secret and added to the container as an environment variable, and the configuration files can be stored in a ConfigMap and mounted into the container as a files on startup.

Let's try it out!


## Secrets

Secrets are a Kubernetes object intended for storing a small amount of sensitive data.  It is worth noting that Secrets are stored base64-encoded within Kubernetes, so they are not wildly secure.  You should make sure to have appropriate [Role-base access controls](https://kubernetes.io/docs/reference/access-authn-authz/rbac/) (or RBAC) to protect access to secrets. Even so, extremely sensitive secret data should probably be stored using something like [HashiCorp Vault](https://www.vaultproject.io/).  For the root password of a MariaDB database, however, they are just fine.


### Creating a Secret manually

To create the Secret containing the `MYSQL_ROOT_PASSWORD`, we need to first pick a password, and convert it to base64:

```
# The root password will be "KubernetesRocks!"
$ echo -n 'KubernetesRocks!' | base64
S3ViZXJuZXRlc1JvY2tzIQ==
```

Make a note of the encoded string.  We need it to create the YAML file for the Secret:

```
apiVersion: v1
kind: Secret
metadata:
  name: mariadb-root-password
type: Opaque
data:
  password: S3ViZXJuZXRlc1JvY2tzIQ==
```

Save that file as `mysql-secret.yaml` and create the secret in Kubernetes with the `kubectl apply` command:

```
$ kubectl apply -f mysql-secret.yaml
secret/mariadb-root-password created
```


### View the newly created Secret

Now that the secret is created, you can use `kubectl describe` to see it.

```
$ kubectl describe secret mariadb-root-password
Name:         mariadb-root-password
Namespace:    secrets-and-configmaps
Labels:       <none>
Annotations:
Type:         Opaque

Data
====
password:  16 bytes
```

Note the `Data` field contains the key we set in the YAML: `password`.  The value assigned to that key is the password we created, but it is not shown in the output.  Instead, the size of the value is shown in its place - in this case 16 bytes.

You can also use `kubectl edit secret <secretname>` to view and edit the secret.  Editing the secret we created shows something like:

```
# Please edit the object below. Lines beginning with a '#' will be ignored,
# and an empty file will abort the edit. If an error occurs while saving this file will be
# reopened with the relevant failures.
#
apiVersion: v1
data:
  password: S3ViZXJuZXRlc1JvY2tzIQ==
kind: Secret
metadata:
  annotations:
    kubectl.kubernetes.io/last-applied-configuration: |
      {"apiVersion":"v1","data":{"password":"S3ViZXJuZXRlc1JvY2tzIQ=="},"kind":"Secret","metadata":{"annotations":{},"name":"mariadb-root-password","namespace":"secrets-and-configmaps"},"type":"Opaque"}
  creationTimestamp: 2019-05-29T12:06:09Z
  name: mariadb-root-password
  namespace: secrets-and-configmaps
  resourceVersion: "85154772"
  selfLink: /api/v1/namespaces/secrets-and-configmaps/secrets/mariadb-root-password
  uid: 2542dadb-820a-11e9-ae24-005056a1db05
type: Opaque
```

Again, you can see the `data` field with the `password` key, and this time the base64-encoded secret.


### Decode the Secret

Let's say we need to view the secret in plain text. For example, we can verify that the secret was created with the correct content by decoding it.

It is easy to decode the secret by extracting the value and piping it to base64.  In this case we will use the output format `-o jsonpath=<path>` to extract only the secret value using a JSONPath template.

```
# Returns the base64 encoded secret string
$ kubectl get secret mariadb-root-password -o jsonpath='{.data.password}'
S3ViZXJuZXRlc1JvY2tzIQ==

# Pipe it to `base64 -d -` to decode:
$ kubectl get secret mariadb-root-password -o jsonpath='{.data.password}' | base64 -d -
KubernetesRocks!
```

### Another way to create Secrets

You can also create Secrets directly using the `kubectl create secret` command.  The MariaDB image also allows for setting up a regular database user with a password by setting the `MYSQL_USER` and `MYSQL_PASSWORD` environment variables.  A Secret can hold more than one key/value pair, so we can create a single Secret to hold both strings.  As a bonus, by using `kubectl create secret`, we can let Kubernetes mess with base64 for us, so we don't have to.

```
$ kubectl create secret generic mariadb-user-creds \
      --from-literal=MYSQL_USER=kubeuser\
      --from-literal=MYSQL_PASSWORD=kube-still-rocks
secret/mariadb-user-creds created
```

Note the use of `--from-literal`.  That sets the key name and the value all in one.  We can pass as many `--from-literal` arguments as needed, to create one or more key/value pairs in the secret.

You can validate that the username and password were created and stored correctly with the `kubectl get secrets` command, again:

```
# Get the username
$ kubectl get secret mariadb-user-creds -o jsonpath='{.data.MYSQL_USER}' | base64 -d -
kubeuser

# Get the password
$ kubectl get secret mariadb-user-creds -o jsonpath='{.data.MYSQL_PASSWORD}' | base64 -d -
kube-still-rocks
```

## ConfigMaps

ConfigMaps are similar to Secrets.  They can be created in the same ways, and can be shared in the containers the same ways.  The only big difference between them is the base64 encoding obfuscation.  ConfigMaps are intended to non-sensitive data - configuration data - like config files and environment variables, and are a great way to create customized running services from generic container images.

### Create a ConfigMap

ConfigMaps can be created the same ways Secrets are.  You can manually create a YAML representation of the ConfigMap and load it into Kubernetes, or you can use the `kubectl create configmap` command.  In the next example, we'll create a ConfigMap using the latter method, but this time, instead of passing literal strings as we did with `--from-literal=<key>=<string>` for the Secret above, we'll create a ConfigMap from an existing file - a MySQL config intended for `/etc/mysql/conf.d` in the container. For this exercise, we will override the "max_allowed_packet" setting for MariaDB, set to 16M by default, with this config file.

First, create a file named `max_allowed_packet.cnf` with the following content:

```
[mysqld]
max_allowed_packet = 64M
```

This will override the default setting in the my.cnf file and set "max_allowed_packet" to 64M.

Once the file is created, we can create a ConfigMap named "mariadb-config" using the `kubectl create configmap` command that contains the file:

```
$ kubectl create configmap mariadb-config --from-file=max_allowed_packet.cnf
configmap/mariadb-config created
```

Just like Secrets, ConfigMaps store one or more key/value pairs in their "Data" hash of the object.  By default, using `--from-file=<filename>` as we did above, the contents of the file will be stored as the value, and the name of the file will be stored as the key.  This is convenient from an organization viewpoint.  However, the keyname can be explicitly set too.  For example, if we'd used `--from-file=max-packet=max_allowed_packet.cnf` when we created the ConfigMap, the key would be "max-packet" rather than the filename.  If we had multiple files to store in the ConfigMap, we could add each of them with an additional `--from-file=<filename>` argument.


### View the new ConfigMap and read the data

As already mentioned, ConfigMaps are not meant to store sensitive data, so the data is not encoded when the ConfigMap is created.  This makes it easy to view and validate the data, and each to edit it directly if desired.

Validate that the ConfigMap we just created did, in fact, get created:

```
$ kubectl get configmap mariadb-config
NAME             DATA      AGE
mariadb-config   1         9m
```

You can see the contents of the ConfigMap with the `kubectl describe` command.  Note that the full contents of the file is visible, and that the keyname is in fact the file name, `max_allowed_packet.cnf`.

```
$ kubectl describe cm mariadb-config
Name:         mariadb-config
Namespace:    secrets-and-configmaps
Labels:       <none>
Annotations:  <none>


Data
====
max_allowed_packet.cnf:
----
[mysqld]
max_allowed_packet = 64M

Events:  <none>
```

A ConfigMap can be edited live within Kubernetes with the `kubectl edit` command.  Doing so will open a buffer with your default editor, showing the contents of the ConfigMap as YAML.  When you save your changes, the changes will be immediately live in Kubernetes.  While not really the _best_ practice, it can be handy for testing things in development.

Let's say we really wanted a "max_allowed_packet" value of 32M instead of the default 16M or the 64M in the `max_allowed_packet.cnf` file.  Use `kubectl edit configmap mariadb-config` to edit the value:

```
$ kubectl edit configmap mariadb-config

# Please edit the object below. Lines beginning with a '#' will be ignored,
# and an empty file will abort the edit. If an error occurs while saving this file will be
# reopened with the relevant failures.
#
apiVersion: v1

data:
  max_allowed_packet.cnf: |
    [mysqld]
    max_allowed_packet = 32M
kind: ConfigMap
metadata:
  creationTimestamp: 2019-05-30T12:02:22Z
  name: mariadb-config
  namespace: secrets-and-configmaps
  resourceVersion: "85609912"
  selfLink: /api/v1/namespaces/secrets-and-configmaps/configmaps/mariadb-config
  uid: c83ccfae-82d2-11e9-832f-005056a1102f
```

After saving the change, verify the data has been updated:

```
# Note the '.' in max_allowed_packet.cnf needs to be escaped
$ kubectl get configmap mariadb-config -o "jsonpath={.data['max_allowed_packet\.cnf']}"

[mysqld]
max_allowed_packet = 32M
```


## Using Secrets and ConfigMaps

Secrets and ConfigMaps can be mounted as environment variables or as files within a container.  For the MariaDB container, we will need to mount the secrets as environment variables, and the ConfigMap as a file.  First, though, we need to write a Deployment for MariaDB, so we have something to work with.  Create a file named "mariadb-deployment.yaml" with the following:

```
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: mariadb
  name: mariadb-deployment
spec:
  replicas: 1
  selector:
    matchLabels:
      app: mariadb
  template:
    metadata:
      labels:
        app: mariadb
    spec:
      containers:
      - name: mariadb
        image: docker.io/mariadb:10.4
        ports:
        - containerPort: 3306
          protocol: TCP
        volumeMounts:
        - mountPath: /var/lib/mysql
          name: mariadb-volume-1
      volumes:
      - emptyDir: {}
        name: mariadb-volume-1
```

This is a bare-bones Kubernetes Deployment of the offical MariaDB 10.4 image from Docker Hub.  Now, let's add our Secrets and ConfigMap.


### Adding the Secrets to the Deployment as environment variables

We have two secrets that need to be added to the Deployment:

1.  mariadb-root-password (with one key/value pair)
2.  mariadb-user-creds (with two key/value pairs)

For the mariadb-root-password secret, we will specify the secret and the key we want by adding an `env` list/array to the container spec in the Deployment, and setting the environment variable value to the value of the key in our secret.  In this case, the list contains only a single entry, for the variable `MYSQL_ROOT_PASSWORD`.

```
 env:
   - name: MYSQL_ROOT_PASSWORD
     valueFrom:
       secretKeyRef:
         name: mariadb-root-password
         key: password
```

Note that the name of the object is the name of the environment variable that is added to the container.  The `valueFrom` field defines `secretKeyRef` as the source from which the environment variable will be set; ie: it will use the value from the "password" key in the "mariadb-root-password" secret we set earlier.

Add this section to the definition for the "mariadb" container in the mariadb-deployment.yaml file.  It should look something like this:


```
 spec:
   containers:
   - name: mariadb
     image: docker.io/mariadb:10.4
     env:
       - name: MYSQL_ROOT_PASSWORD
         valueFrom:
           secretKeyRef:
             name: mariadb-root-password
             key: password
     ports:
     - containerPort: 3306
       protocol: TCP
     volumeMounts:
     - mountPath: /var/lib/mysql
       name: mariadb-volume-1
```

In this way we have explicitly set the variable to the value of a specific key from our secret.  This method can also be used with ConfigMaps by using `configMapRef` instead of `secretKeyRef`.

It is also possible to set environment variables from _all_ key/value pairs in a secret or config map, automatically using the key name as the environment variable name and the key's value as the environment variable's value.  By using `envFrom` rather than `env` in the container spec, we can set the `MYSQL_USER` and `MYSQL_PASSWORD` from the "mariadb-user-creds" secret we created earlier, all in one go:

```
 envFrom:
 - secretRef:
     name: mariadb-user-creds
```

`envFrom` is a list of sources for Kubernetes to take environment variables.  Again, we're using `secretRef`, this time to specify "mariadb-user-creds" as the source of the environment variables.  That's it!  All the keys and values in the secret will be added as environment variables in the container.

You should now have a container spec that looks like this:

```
spec:
  containers:
  - name: mariadb
    image: docker.io/mariadb:10.4
    env:
      - name: MYSQL_ROOT_PASSWORD
        valueFrom:
          secretKeyRef:
            name: mariadb-root-password
            key: password
    envFrom:
    - secretRef:
        name: mariadb-user-creds
    ports:
    - containerPort: 3306
      protocol: TCP
    volumeMounts:
    - mountPath: /var/lib/mysql
      name: mariadb-volume-1
```

__Note:__  We could have just added the "mysql-root-password" secret to the `envFrom` list and let it be parsed as well, as long as the "password" key was named "MYSQL_ROOT_PASSWORD" instead.  There is not a way to manually specify the environment variable name with `envFrom` the way we did with `env`.

As mentioned above, you can use both `env` and `envFrom` to share ConfigMap key/value pairs with a container as well.  In the case of our ConfigMap "mariadb-config", though, we have an entire file stored as the value to our key, and the file needs to exist in the filesystem of the container for MariaDB to be able to use it.  Luckily, both Secrets and ConfigMaps be the source of Kubernetes "volumes" and mounted into the containers, instead of using a filesystem or block device as the volume to be mounted.

Our mariadb-deployment.yaml already has a volume and volumeMount specified, an `emptyDir` (effectively, temporary or ephemeral) volume being mounted to `/var/lib/mysql` to store the MariaDB data:

```
<...>

  volumeMounts:
  - mountPath: /var/lib/mysql
    name: mariadb-volume-1

<...>

volumes:
- emptyDir: {}
name: mariadb-volume-1

<...>
```

_Note:_ This is not a production configuration.  When the pod restarts, the data in the `emptyDir` volume is lost.  It is primarily used for development or when the contents of the volume do not need to be persistent.

We can add our ConfigMap as a source by adding it to the volume list, and then adding a volumeMount for it to the container definition:


```
<...>

  volumeMounts:
  - mountPath: /var/lib/mysql
    name: mariadb-volume-1
  - mountPath: /etc/mysql/conf.d
    name: mariadb-config

<...>

volumes:
- emptyDir: {}
  name: mariadb-volume-1
- configMap:
    name: mariadb-config
    items:
      - key: max_allowed_packet.cnf
        path: max_allowed_packet.cnf
  name: mariadb-config-volume

<...>
```

The `volumeMount` is pretty self-explanitory here - create a volume mount the "mariadb-config-volume" volume (specified in the `volumes` list, below it) to the path `/etc/mysql/conf.d`.

Then, in the `volumes` list, `configMap` tells Kubernetes to use the "mariadb-config" ConfigMap, taking the contents of the key "max_allowed_packet.cnf" and mounting it to the path "max_allowed_packed.cnf". The name of the volume itself is "mariadb-config-volume", which was referenced in the `volumeMounts` above.

_Note:_ The `path` from the `configMap` is the name of a file that will contain the contents of the key's value.  In this case, our key was a file name, too, but it does not need to be.  Note also that `items` is a list, so multiple keys can be referenced and their values mounted as files. These files will all be created in the `mountPath` of the `volumeMount` specified above: `/etc/mysql/conf.d`.


### Creating a MariaDB instance from the Deployment

At this point we should have enough to create a MariaDB instance.  There are two Secrets, one holding the `MYSQL_ROOT_PASSWORD` and another storing the `MYSQL_USER` and `MYSQL_PASSWORD` environment variables to be added to the container.  We also have a ConfigMap holding the contents of a MySQL config file that overrides the `max_allowed_packed` value from its default setting.

We also have a "mariadb-deployment.yaml" file that describes a Kubernetes deployment of a pod with a MariaDB container, adds the Secrets as environment variables and the ConfigMap as a volume mounted file in the container.  It should look like this:

```
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: mariadb
  name: mariadb-deployment
spec:
  replicas: 1
  selector:
    matchLabels:
      app: mariadb
  template:
    metadata:
      labels:
        app: mariadb
    spec:
      containers:
      - image: docker.io/mariadb:10.4
        name: mariadb
        env:
          - name: MYSQL_ROOT_PASSWORD
            valueFrom:
              secretKeyRef:
                name: mariadb-root-password
                key: password
        envFrom:
        - secretRef:
            name: mariadb-user-creds
        ports:
        - containerPort: 3306
          protocol: TCP
        volumeMounts:
        - mountPath: /var/lib/mysql
          name: mariadb-volume-1
        - mountPath: /etc/mysql/conf.d
          name: mariadb-config
      volumes:
      - emptyDir: {}
        name: mariadb-volume-1
      - configMap:
          name: mariadb-config
          items:
            - key: max_allowed_packet.cnf
              path: max_allowed_packet.cnf
        name: mariadb-config-volume
```

Create a new MariaDB instance from the YAML file with the `kubectl create` command:

```


TODO: Conclusion
