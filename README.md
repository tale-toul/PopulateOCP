# Overview

This playbook and collection of roles will deploy a set of applications to an OpenShift cluster.  The intention is to quickly deploy some **non production** loads to a cluster for testing purposes.

The playbook is compatible with OCP 3 and 4 versions, although some of the applications may not be, so the playbook itself will decice if the particular application must be deployed or not depending on the version of OpenShift. 

Some of the applications will use persistent storage by creating PVCs, so the cluster is expected to have a default storage class defined.

The roles try to be idempotent, so additional executions of the playbook will not modify the resources that have not been changed.

Most of the work is done using the [k8s](https://docs.ansible.com/ansible/latest/modules/k8s_module.html#k8s-module) ansible modules family: k8s; k8s_info; k8s_auth   

The projects created by the playbook are:  cakephp; imageuploader; rocket; todoapp (only for OCP 3)

# How to use the playbook

## Deploy projects

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

The calls to the OpenShift API are authenticated using an API key, this key is obtained at the beginning of the playbook and revoked at the end.  To obtain the api key a valid user and password with access to the OCP cluster are required by the playbook, no special permissions are required.  The credentials are expected to be found in a file called **secret** in the same directory where the populate-ocp.yml playbook is:

```
ocp_user: <username>
ocp_pass: <password>
```
This file can be encrypted with **ansible-vault**:

```
$ ansible-vault encrypt secret
```

Finally run the playbook, in the exampel a file with the encryption keys is passed via the `--vault-id` parameter:

```
$ ansible-playbook populate-ocp.yml --vault-id key.txt
```

## Remove projects

The playbook can also be used to remove all or some of the projects prevously deployed.  This behavior is controlled by a group of variables, one for each project, defined in the **vars** section at the beginning of the populate-ocp.yml playbook.  

The default value of all these variables is **present** which causes the projects and its components to be created, changing any of the variables to **absent** will cause that particular project and all its resources to be removed:

```
  vars:
    cakephp_state: "present"
    imageuploader_state: "present"
    rocket_state: "present"
    todo_state: "present"
```

For example to remove the cakephp and todoapp projects use the following command:

```
$ ansible-playbook populate-ocp.yml -e cakephp_state=absent -e todo_state=absent --vault-id key.txt
```

## Security considerations

Given that this are test cases not meant for production systems, security is weak. 

* The passwords defined for the cakephp applications are defined in the clear: database_password; cakephp_secret_token; cakephp_security_salt; github_webhook_secret.  They should be defined in a "_vaulted_" file.

## Idempotency when removing projects

The tasks in the roles of all projects are divided in to _groups_:  The first one is mainly made of the task _create project_; and the second one is made of the tasks that create and verify the resources in the project.

* The project creation is always executed both for creation and destruction of the projects, its result is saved in a variable.  A conditional clause is defined to define when this task fails, this is needed because a _normal_ user without special privileges will fail in deleting a project that does not currently exist, and this may happend if the playbook is run more than onece with state=absent for any project.  The conditional defines that the task fails only when the result of the task is **failed** and the next two conditions don't happend: the project was being removed and the creation status is 403 (forbidden); if the task fails but the the project was being removed and the status code is 403, the result is not considered a failure:

```
...
  register: _project_creation
  failed_when: _project_creation.failed == true and not (imageuploader_state == "absent" and _project_creation.status == 403)
...
```

* The next group of tasks is grouped in a block and only runs when the project is being created, because OpenShift will take care of destroying all the resources in a project when the project itself has been deleted.  This grouping of tasks saves execution time and also avoids spurious errors due to a tasks trying to delete a resource that has already been deleted by OpenShift.
