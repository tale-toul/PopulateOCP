The tasks will be run on localhost, but the URL for the OCP server API will be provided.

The CA cert from the server is required:

* For OCP 4 extract the CA of the ingress and save it to a file (https://access.redhat.com/solutions/4505101):

```
$ oc rsh -n openshift-authentication <oauth-openshift-pod> cat /run/secrets/kubernetes.io/serviceaccount/ca.crt > ingress-ca.crt
```

* For OCP 3

Get the following file from any master: /etc/origin/master/ca-bundle.crt


To use the k8s ansible module family the python openshift client is required, this can be installed from EPEL repository (https://fedoraproject.org/wiki/EPEL). If using RHEL as ansible control host, additionally the server-extras and server-optinal repos must be enabled:

```
# subscription-manager repos --enable rhel-7-server-optional-rpms --enable rhel-7-server-extras-rpms
```

Check the version of python that ansible uses:

```
# ansible --version
```

Finally install the python openshift client package for the python 2 or 3 version:

```
# yum install python2-openshift
```

