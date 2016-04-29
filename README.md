# wordpress-ex

This is a set of configuration files adapted from [OpenShift/origin examples](https://github.com/openshift/origin/tree/master/examples) that work with OpenShift 3 to create a wordpress application, and follow-on steps to demonstrate the use of persistent volumes.

- [Getting Started](#Getting-Started)
- [Create a WordPress Project](#Create-a-WordPress-Project)
- [Start Blogging](#Start-Blogging)
- [Delete the Project](#Delete-the-Project)
- [Restore the Project](#Restore-the-Project)
- [Restart the Blog](#Restart-the-Blog)

<a name="Getting-Started"></a>
## Getting Started

### Storage Provisioning Using NFS Persistent Volumes

OpenShift expects that storage volumes are provisioned by system administrator outside of OpenShift. As subsequent step, the system admin then tells OpenShift about these volumes by creating Persistent Volumes objects. Wearing your "sysadmin" hat, for this example we will create NFS Persistent Volumes named `pv0001` and `pv0002`.

`oc login` with admin privileges and `git clone` this repository.

#### NFS Provisioning

We'll be creating NFS exports on the local machine.  The instructions below are for Fedora.  The provisioning process may be slightly different based on linux distribution or the type of NFS server being used.

Create two NFS exports, each of which will become a Persistent Volume in the cluster.

```
# the directories in this example can grow unbounded
# use disk partitions of specific sizes to enforce storage quotas
$ mkdir -p /home/data/pv0001
$ mkdir -p /home/data/pv0002

# security needs to be permissive currently, but the export will soon be restricted
# to the same UID/GID that wrote the data
$ chmod -R 777 /home/data/

# Add these two lines to /etc/exports
/home/data/pv0001 *(rw,sync)
/home/data/pv0002 *(rw,sync)

# Enable the new exports without bouncing the NFS service
$ exportfs -a

```

#### SELinux Security

By default, SELinux does not allow writing from a pod to a remote NFS server. The NFS volume mounts correctly, but is read-only.

To enable writing in SELinux on each node:

```
# -P makes the bool persistent between reboots.
$ setsebool -P virt_use_nfs 1
```

#### NFS Persistent Volumes

Each NFS export becomes its own Persistent Volume in the cluster.

```
# Create the persistent volumes for NFS.
$ oc create -f wordpress-ex/pv-1.yaml
$ oc create -f wordpress-ex/pv-2.yaml
$ oc get pv

NAME      LABELS    CAPACITY     ACCESSMODES   STATUS      CLAIM     REASON
pv0001    <none>    1073741824   RWO,RWX       Available             
pv0002    <none>    5368709120   RWO           Available             

```

Now the volumes are ready to be used by applications in the cluster. Fore more information on the lifecycle of persistent volumes read the [recycling policy](http://kubernetes.io/docs/user-guide/persistent-volumes/#recycling-policy).

### Adjusting Security Constraints for Third-Party Docker Images

The Wordpress Dockerhub image binds Apache to port 80 in the container, which requires root access. We can allow that
in this example, but those wishing to run a more secure cluster will want to ensure their images don't require root access (e.g, bind to high number ports, don't chown or chmod dirs, etc)

Allow Wordpress to bind to port 80 by editing the restricted security context restraint.  Change "runAsUser" from ```MustRunAsRange``` to ```RunAsAny```.

```
$ oc edit scc restricted

apiVersion: v1
groups:
- system:authenticated
kind: SecurityContextConstraints
metadata:
  creationTimestamp: 2015-06-26T14:01:03Z
  name: restricted
  resourceVersion: "59"
  selfLink: /api/v1/securitycontextconstraints/restricted
  uid: c827a79d-1c0b-11e5-9166-d4bed9b39058
runAsUser:
  type: MustRunAsRange  <-- change to RunAsAny
seLinuxContext:
  type: MustRunAs
```

Changing the restricted security context as shown above allows the Wordpress container to bind to port 80.  

## Create a WordPress Project

Since that "sysadmin" fellow has already deployed some Persistent Volumes, you can continue as an application developer and actually use these volumes to store some MySQL and Wordpress data.

`oc login` as a regular user and create a new project.

```
oc new-project project-alpha
```

### Establish Persistent Volumes Claims

Create claims for storage. The claims in this example carefully match the volumes created above.

```
$ oc create -f wordpress-ex/pvc-wp.yaml
$ oc create -f wordpress-ex/pvc-mysql.yaml
$ oc get pvc

NAME          STATUS    VOLUME    CAPACITY   ACCESSMODES   AGE
claim-mysql   Bound     pv0002    5Gi        RWO           17s
claim-wp      Bound     pv0001    1Gi        RWO,RWX       28s
```

### Launch MySQL

Launch the MySQL pod.

```
$ oc create -f wordpress-ex/pod-mysql.yaml
```

After a few moments, MySQL will be running and accessible via the pod's IP address.  We don't know what the IP address
will be and we wouldn't want to hard-code that value in any pod that wants access to MySQL.  

Create a service in front of MySQL that allows other pods to connect to it by name.

```
# This allows the pod to access MySQL via a service name instead of hard-coded host address
$ oc create -f wordpress-ex/service-mysql.yaml
```

### Launch WordPress

We use the MySQL service defined above in our Wordpress pod defined by "pod-wordpress.yaml".  The variable WORDPRESS_DB_HOST is set to the name
 of our MySQL service.

Because the Wordpress pod and MySQL service are running in the same namespace, we can reference the service by name.  We
can also access a service in another namespace by using the name and namespace: ```mysql.another_namespace```.  The fully qualified
name of the service would also work: ```mysql.<namespace>.svc.cluster.local```

```
- name: WORDPRESS_DB_HOST
  # this is the name of the mysql service fronting the mysql pod in the same namespace
  # expands to mysql.<namespace>.svc.cluster.local  - where <namespace> is the current namespace
  value: mysql
```

Launch the Wordpress pod and its corresponding service.

```
$ oc create -f wordpress-ex/pod-wordpress.yaml
$ oc create -f wordpress-ex/service-wp.yaml
$ oc get svc

NAME            LABELS                                    SELECTOR         IP(S)            PORT(S)
mysql           name=mysql                                name=mysql       172.30.115.137   3306/TCP
wpfrontend      name=wpfrontend                           name=wordpress   172.30.170.55    5055/TCP
```

Check the status of the pods to see if they've gone from "ContainerCreating" to "Running".

```
$ oc get all

NAME         CLUSTER-IP      EXTERNAL-IP   PORT(S)    AGE
mysql        172.30.115.137  <none>        3306/TCP   7m
wpfrontend   172.30.170.55                 5055/TCP   4m
NAME         READY           STATUS        RESTARTS   AGE
mysql        1/1             Running       0          26m
wordpress    1/1             Running       2          4m
```

<a name="Start-Blogging"></a>
## Start Blogging

In your browser, visit 172.30.170.55:5055 (your IP address will vary, depending on your set-up you may need to create a route -- this can be done in the OpenShift web console).

The Wordpress install process will lead you through setting up the blog.

<a name="Delete-the-Project"></a>
## Delete the Project

After customizing your blog let's try deleting & restoring the entire project. 

Note: if a persistent volume persistentVolumeReclaimPolicy set to `Recycle` instead of `Retain` then deleting the project will automatically delete data on the persistent volumes. If volumes are set to `Recycle`, data on the volumes should be backed up manually before deleting a project. Otherwise run `oc edit pv <pv-name>` to make adjustments.

```
oc delete project/project-alpha
```

<a name="Restore-the-Project"></a>
## Restore the Project

Create a new project, repeating steps from above.

```
oc new-project project-beta
```

Create claims for storage.

```
$ oc create -f wordpress-ex/pvc-wp.yaml
$ oc create -f wordpress-ex/pvc-mysql.yaml
$ oc get pvc

NAME          STATUS    VOLUME    CAPACITY   ACCESSMODES   AGE
claim-mysql   Bound     pv0002    5Gi        RWO           17s
claim-wp      Bound     pv0001    1Gi        RWO,RWX       28s
```

Note: if you are working with extra volumes and the pvc does not bind to the intended pv, please note that improvements are coming in this arena ([https://github.com/kubernetes/kubernetes/pull/24682](https://github.com/kubernetes/kubernetes/pull/24682)). In the mean time, copy your data to the bound volume.

Launch the MySQL and WordPress pods and services.

```
$ oc create -f wordpress-ex/pod-mysql.yaml
$ oc create -f wordpress-ex/service-mysql.yaml
$ oc create -f wordpress-ex/pod-wordpress.yaml
$ oc create -f wordpress-ex/service-wp.yaml
```

Check the status of the pods to see if they've gone from "ContainerCreating" to "Running".

```
$ oc get all

NAME         CLUSTER-IP      EXTERNAL-IP   PORT(S)    AGE
mysql        172.30.115.137  <none>        3306/TCP   7m
wpfrontend   172.30.170.55                 5055/TCP   4m
NAME         READY           STATUS        RESTARTS   AGE
mysql        1/1             Running       0          26m
wordpress    1/1             Running       0          4m
```

<a name="Restart-the-Blog"></a>
## Restart the Blog

In your browser, visit the address for wpfrontend or create a route with the web console.

The Wordpress site should look how you left it.

