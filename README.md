# Overview

This playbook and collection of roles will deploy a set of applications to an OpenShift cluster.  The intention is to quickly deploy some **non production** loads to a cluster for testing purposes.

The playbook is compatible with OCP 3 and 4 versions, although some of the applications may not be, so the playbook itself will decice if the particular application must be deployed or not depending on the version of OpenShift. 

Some of the applications will use persistent storage by creating PVCs, so the cluster is expected to have a default storage class defined.

The roles try to be idempotent, so additional executions of the playbook will not redeploy resources that have not been changed.

Most of the work is done using the [k8s](https://docs.ansible.com/ansible/latest/modules/k8s_module.html#k8s-module) ansible modules family: k8s; k8s_info; k8s_auth   

The projects created by the playbook are:  cakephp; imageuploader; rocket; todoapp (only for OCP 3)

# How to use the playbook

This playbook does not require an inventory file since all taks are run on localhost, however the k8s modules use the parameter **host** to specify the OpenShift cluster API entry point, this API URL can be obtained running the following command from an already logged in session with the OCP cluster:

```
$ oc whoami --show-server
https://master.ocpext.example.com:443
```
Add the URL to the variable **api_entrypoint** in the **vars** section in the file **populate-ocp.yaml**:

```
  vars:
    api_entrypoint: "https://master.ocpext.example.com:443"
```

The CA cert from the OCP server is required to verify the SSL connections with the API, get the certificate using the following instructions and copy it in the same directory where the populate-ocp.yml playbook is:

* For OCP 4 extract the CA cert from the ingress controller and save it to a file (https://access.redhat.com/solutions/4505101):

```
$ oc rsh -n openshift-authentication <oauth-openshift-pod> cat /run/secrets/kubernetes.io/serviceaccount/ca.crt > ingress-ca.crt
```

* For OCP 3 get the following file from any master: /etc/origin/master/ca-bundle.crt

Assign the name of the file to the variable **api_ca_cert** in the **vars** section in the file **populate-ocp.yaml**:

```
  vars:
    api_ca_cert: "ca-bundle.crt"
```

To use the k8s ansible module family the _python openshift client_ is required to be installed in the control host, this can be installed from [EPEL repository](https://fedoraproject.org/wiki/EPEL). If the ansible control host is using RHEL, additionally the server-extras and server-optinal repos must be enabled:

```
# subscription-manager repos --enable rhel-7-server-optional-rpms --enable rhel-7-server-extras-rpms
```

Check the python version that ansible uses:

```
# ansible --version
ansible 2.9.4
...
  python version = 2.7.5 (default, Jun 11 2019, 14:33:56) [GCC 4.8.5 20150623 (Red Hat 4.8.5-39)]
```

Finally install the python openshift client package for python 2 or 3 version, the same as the one used by ansible:

```
# yum install python2-openshift
```

The calls to the OpenShift API are authenticated using an API key, this key is obtained at the beginning of the playbook and revoked at the end.  To obtain the api key a valid user and password are required by the playbook, the credentials are expected to be found in a file called **secret** in the same directory where the populate-ocp.yml playbook is:

```
ocp_user: <username>
ocp_pass: <password>
```
This file can be encrypted with **ansible-vault**:

```
$ ansible-vault encrypt secret
```

## Security considerations

Given that this are test cases not meant for production systems, security is weak. 

* The passwords defined for the cakephp applications are defined in the clear: database_password; cakephp_secret_token; cakephp_security_salt; github_webhook_secret.  They should be defined in a "_vaulted_" file.
