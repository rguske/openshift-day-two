# Red Hat OpenShift Day-2 Operations

⚠️ WIP

## Identity Provider `htpasswd`

[Docs: Configuring an htpasswd identity provider](https://docs.redhat.com/en/documentation/openshift_container_platform/4.17/html/authentication_and_authorization/configuring-identity-providers#configuring-htpasswd-identity-provider)

Step 1: Create an htpasswd file to store the user and password information:

`htpasswd -c -B -b users.htpasswd rguske <password>`

Add a new user to the file:

`htpasswd -bB users.htpasswd rbohne 'r3dh4t1!'`

`htpasswd -bB users.htpasswd devuser 'r3dh4t1!'`

Remove an existing user:

`htpasswd -D users.htpasswd <username>`

Replacing an updated `users.htpasswd` file:

`oc create secret generic htpass-secret --from-file=htpasswd=users.htpasswd --dry-run=client -o yaml -n openshift-config | oc replace -f -`

Step 2: Create a Kubernetes secret:

`oc create secret generic htpass-secret-rguske --from-file=htpasswd=<path_to_rguske.htpasswd> -n openshift-config`

`oc create secret generic htpass-secret-devuser --from-file=htpasswd=<path_to_devuser.htpasswd> -n openshift-config`

This can also be done using the OpenShift User Interface:

![configure-oauth-passd](assets/oauth-passwd.png)

### Updating User

`oc get secret htpass-secret -ojsonpath={.data.htpasswd} -n openshift-config | base64 --decode > users.htpasswd`

## Configure Permissions

[Docs: Using RBAC to define and apply permissions](https://docs.redhat.com/en/documentation/openshift_container_platform/4.17/html/authentication_and_authorization/using-rbac#authorization-overview_using-rbac)

![rbac](assets/rbac.png)

Add cluster-wide admin priviledges to e.g. user rguske:

```yaml
kind: ClusterRoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: rguske-cluster-admin
subjects:
  - kind: User
    apiGroup: rbac.authorization.k8s.io
    name: rguske
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-admin
```

Alternatively via the WebUi:

![rolebinding](assets/rolebinding.png)

## Make a Control-Plane Node `scheduable`

Red Hat KB6148012 - [How to schedule pod on master node where scheduling is disabled?](https://access.redhat.com/solutions/6148012)

```code
oc get scheduler cluster -oyaml

apiVersion: config.openshift.io/v1
kind: Scheduler
metadata:
  creationTimestamp: "2025-01-28T15:20:20Z"
  generation: 1
  name: cluster
  resourceVersion: "542"
  uid: 59f6fef1-e88a-484a-8e3c-fa38e6e300b3
spec:
  mastersSchedulable: false
  policy:
    name: ""
status: {}
```

Edit the `scheduler` CR and configure the spec: `mastersSchedulable: true`

```code
oc get nodes
NAME                  STATUS   ROLES                         AGE     VERSION
ocp1-h5ggj-master-0   Ready    control-plane,master,worker   2d19h   v1.30.6
ocp1-h5ggj-master-1   Ready    control-plane,master,worker   2d19h   v1.30.6
ocp1-h5ggj-master-2   Ready    control-plane,master,worker   2d19h   v1.30.6
ocp1-h5ggj-worker-0   Ready    worker                        2d18h   v1.30.6
ocp1-h5ggj-worker-1   Ready    worker                        2d18h   v1.30.6
```

## Troubleshooting

### Gathering Logs

[Creating must-gather with more details for specific components in OCP 4
](https://access.redhat.com/solutions/5459251)

Data Collection Audit logs:

`oc adm must-gather -- /usr/bin/gather_audit_logs`

Default must-gather including the audit logs:

`oc adm must-gather -- '/usr/bin/gather && /usr/bin/gather_audit_logs'`

OCPV:

`oc adm must-gather --image-stream=openshift/must-gather --image=registry.redhat.io/container-native-virtualization/cnv-must-gather-rhel[8,9]:[operator_version]`

The [8,9] should be replaced based on the version of OCP 4.12 uses rhel8, and OCP 4.13 and later uses rhel9.
The [operator_version] tag should be in format v4.y.z.

Examples - 4.17: `oc adm must-gather --image-stream=openshift/must-gather --image=registry.redhat.io/container-native-virtualization/cnv-must-gather-rhel8:v4.17.4`

```code
oc adm must-gather \
 --image-stream=openshift/must-gather \
 --image=registry.redhat.io/container-native-virtualization/cnv-must-gather-rhel9:v4.17.4 \
 --image=registry.redhat.io/workload-availability/node-healthcheck-must-gather-rhel9:v0.9.0
```

[How to generate a sosreport within nodes without SSH in OCP 4](https://access.redhat.com/solutions/4387261)

```code
oc get nodes
NAME                  STATUS   ROLES                         AGE     VERSION
ocp1-h5ggj-master-0   Ready    control-plane,master,worker   2d19h   v1.30.6
ocp1-h5ggj-master-1   Ready    control-plane,master,worker   2d19h   v1.30.6
ocp1-h5ggj-master-2   Ready    control-plane,master,worker   2d19h   v1.30.6
ocp1-h5ggj-worker-0   Ready    worker                        2d18h   v1.30.6
ocp1-h5ggj-worker-1   Ready    worker                        2d18h   v1.30.6
```

Then, create a debug session with oc debug node/<node name> (in this example oc debug node/node-1). The debug session will spawn a pod using the tools image from the release (which doesn't contain sos):

```code
oc debug node/ocp1-h5ggj-master-0
```

```code
chroot /host bash
[root@ocp1-h5ggj-master-0 /]#  cat /etc/redhat-release
Red Hat Enterprise Linux CoreOS release 4.17
```

```code
$ toolbox

Trying to pull registry.redhat.io/rhel9/support-tools:latest...
Getting image source signatures
Checking if image destination supports signatures
Copying blob facf1e7dd3e0 done   |
Copying blob a0e56de801f5 done   |
Copying blob ec465ce79861 done   |
Copying blob cbea42b25984 done   |
Copying config a627accb68 done   |
Writing manifest to image destination
Storing signatures
a627accb682adb407580be0d7d707afbcb90abf2f407a0b0519bacafa15dd409
Spawning a container 'toolbox-root' with image 'registry.redhat.io/rhel9/support-tools'
Detected RUN label in the container image. Using that as the default...
ebf4dd2b82bf8ebeab55291c8ca195b61e13c9fc5d8dfb095f5fdcbcdabae2df
toolbox-root
Container started successfully. To exit, type 'exit'.
```

`sosreport -e openshift -k crio.all=on -k crio.logs=on  -k podman.all=on -k podman.logs=on --all-logs`

## Nested Virtualization

[How to set the CPU model to Passthrough in OpenShift Virtualization?](https://access.redhat.com/solutions/7069612)

```yaml
oc create -f - <<EOF
apiVersion: kubevirt.io/v1
kind: VirtualMachine
metadata:
  annotations:
  labels:
    app: rhel9-pod-bridge
    kubevirt.io/dynamic-credentials-support: "true"
  name: rhel9-pod-bridge
spec:
  dataVolumeTemplates:
    - apiVersion: cdi.kubevirt.io/v1beta1
      kind: DataVolume
      metadata:
        name: rhel9-pod-bridge
      spec:
        sourceRef:
          kind: DataSource
          name: rhel9
          namespace: openshift-virtualization-os-images
        storage:
          accessModes:
            - ReadWriteMany
          storageClassName: thin-csi
          resources:
            requests:
              storage: 30Gi
  running: false
  template:
    metadata:
      annotations:
        vm.kubevirt.io/flavor: tiny
        vm.kubevirt.io/os: rhel9
        vm.kubevirt.io/workload: server
        kubevirt.io/allow-pod-bridge-network-live-migration: ""
      labels:
        kubevirt.io/domain: rhel9-pod-bridge
        kubevirt.io/size: tiny
    spec:
      domain:
        cpu:
          model: host-passthrough
          cores: 1
          sockets: 1
          threads: 1
        devices:
          disks:
            - disk:
                bus: virtio
              name: rootdisk
            - disk:
                bus: virtio
              name: cloudinitdisk
          interfaces:
            - bridge: {}
              name: default
        machine:
          type: pc-q35-rhel9.2.0
        memory:
          guest: 1.5Gi
      networks:
        - name: default
          pod: {}
      terminationGracePeriodSeconds: 180
      volumes:
        - dataVolume:
            name: rhel9-pod-bridge
          name: rootdisk
        - cloudInitNoCloud:
            userData: |-
              #cloud-config
              user: cloud-user
              password: redhat
              chpasswd: { expire: False }
          name: cloudinitdisk
EOF
```

Other sources:

[OpenShift Virtualization reports no nodes are available, cannot start VMs](https://access.redhat.com/solutions/5106121)

[Nested virtualization in OpenShift Virtualization](https://access.redhat.com/solutions/6692341)

Enable features on vSphere:

![nested-v](assets/nested-v-on-vsphere.png)

## Replacing the default ingress certificate

Prerequisites:

* You must have a wildcard certificate for the fully qualified .apps subdomain and its corresponding private key. Each should be in a separate PEM format file.
* The private key must be unencrypted. If your key is encrypted, decrypt it before importing it into OpenShift Container Platform.
* The certificate must include the subjectAltName extension showing *.apps.<clustername>.<domain>.
* The certificate file can contain one or more certificates in a chain. The wildcard certificate must be the first certificate in the file. It can then be followed with any intermediate certificates, and the file should end with the root CA certificate.
* Copy the root CA certificate into an additional PEM format file.
* Verify that all certificates which include -----END CERTIFICATE----- also end with one carriage return after that line.

Create a config map that includes only the root CA certificate used to sign the wildcard certificate:

```yaml
oc apply -f - <<EOF
apiVersion: v1
kind: ConfigMap
metadata:
  name: user-ca-bundle
  namespace: openshift-config
data:
  ca-bundle.crt: |
    # MyPrivateCA (root.crt)
    -----BEGIN CERTIFICATE-----
   zzzzz
    -----END CERTIFICATE-----
EOF
```

Update the cluster-wide proxy configuration with the newly created config map:

```code
oc patch proxy/cluster \
     --type=merge \
     --patch='{"spec":{"trustedCA":{"name":"user-ca-bundle"}}}'
```

Create a secret that contains the wildcard certificate chain and key:

```code
oc create secret tls ocp1-wildcard-cert \
     --cert='/Users/rguske/Downloads/ocp1.rguske/chain.crt' \
     --key='/Users/rguske/Downloads/ocp1.rguske/key.key' \
     -n openshift-ingress
```

Update the Ingress Controller configuration with the newly created secret:

// Replace the secret name

```code
oc patch ingresscontroller.operator default \
     --type=merge -p \
     '{"spec":{"defaultCertificate": {"name": "ocp1-wildcard-cert"}}}' \
     -n openshift-ingress-operator
```

Watch the ClusterOperator (`co`) for the status update.

## Login Customizations

### Customizing the web console in OpenShift Container Platform

[Docs - Customizing the web console in OpenShift Container Platform](https://docs.openshift.com/container-platform/4.17/web_console/customizing-the-web-console.html)

```code
oc create configmap console-custom-logo --from-file /path/to/console-custom-logo.png -n openshift-config
```

oc create configmap console-custom-logo --from-file '/Users/rguske/Documents/ironman.jpg' -n openshift-config

Edit the web console’s Operator configuration to include customLogoFile and customProductName:

`oc edit consoles.operator.openshift.io cluster`

```yaml
apiVersion: operator.openshift.io/v1
kind: Console
metadata:
  name: cluster
spec:
  customization:
    customLogoFile:
      key: ironman.jpg
      name: console-custom-logo
    customProductName: My Console
```

Once the Operator configuration is updated, it will sync the custom logo config map into the console namespace, mount it to the console pod, and redeploy.

Validate: `oc get clusteroperator console`

### Customizing the login/provider page

[Docs - Customizing the login page](https://docs.openshift.com/container-platform/4.17/web_console/customizing-the-web-console.html)

Run the following commands to create templates you can modify:

`oc adm create-login-template > login.html`

Alternatively, adjust the existing `login.html` and or `provider.html`.

Export the existing `login.html` and `provider.html`:

`POD=$(oc get pods -n openshift-authentication -o name | head -n 1)`

`oc exec -n openshift-authentication "$POD" -- cat /var/config/system/secrets/v4-0-config-system-ocp-branding-template/login.html > login.html`

`oc exec -n openshift-authentication "$POD" -- cat /var/config/system/secrets/v4-0-config-system-ocp-branding-template/providers.html > providers.html`

Choose an image ewhich you'd like to use for the replacement. Encode the the image into `base64`. [Base64 Guru](https://base64.guru/converter/encode/image) helps.

Replace the base64 value in the `login.html`. Search for `background-image:url(data:image/`, pay attention to the file format (png, svg, jpg), adjust it if necessary and replace the base64 value of the image.

Create the secrets:

```code
oc -n openshift-config get secret
NAME                                      TYPE                             DATA   AGE
etcd-client                               kubernetes.io/tls                2      8d
htpasswd-dm9mt                            Opaque                           1      6d1h
initial-service-account-private-key       Opaque                           1      8d
pull-secret                               kubernetes.io/dockerconfigjson   1      8d
webhook-authentication-integrated-oauth   Opaque                           1      8d
```

`oc create secret generic login-template --from-file=login.html -n openshift-config`

`oc create secret generic providers-template --from-file=providers.html -n openshift-config`

Edit the `oauth` CR:

`oc edit oauths cluster`

```yaml
apiVersion: config.openshift.io/v1
kind: OAuth
metadata:
  name: cluster
# ...
spec:
  templates:
    error:
        name: error-template
    login:
        name: login-template
    providerSelection:
        name: providers-template
```

After editing the CR, the pods within the `openshift-authentication` namespace will be redeployed.

```code
oc -n openshift-authentication get pods -w
NAME                              READY   STATUS    RESTARTS   AGE
oauth-openshift-8c7859b9f-fwsnl   1/1     Running   0          6m55s
oauth-openshift-8c7859b9f-kp8rw   1/1     Running   0          7m53s
oauth-openshift-8c7859b9f-qw7wl   1/1     Running   0          7m25s
oauth-openshift-8c7859b9f-kp8rw   1/1     Terminating   0          8m42s
oauth-openshift-664fbb9d49-r5bzk   0/1     Pending       0          0s
oauth-openshift-664fbb9d49-r5bzk   0/1     Pending       0          0s
oauth-openshift-8c7859b9f-kp8rw    0/1     Terminating   0          9m8s
oauth-openshift-664fbb9d49-r5bzk   0/1     Pending       0          26s
oauth-openshift-664fbb9d49-r5bzk   0/1     Pending       0          26s
oauth-openshift-664fbb9d49-r5bzk   0/1     ContainerCreating   0          26s
oauth-openshift-8c7859b9f-kp8rw    0/1     Terminating         0          9m8s
oauth-openshift-8c7859b9f-kp8rw    0/1     Terminating         0          9m8s
oauth-openshift-664fbb9d49-r5bzk   0/1     ContainerCreating   0          27s
oauth-openshift-664fbb9d49-r5bzk   0/1     Running             0          27s
oauth-openshift-664fbb9d49-r5bzk   1/1     Running             0          28s
```

![provider-image](assets/provider-image.png)