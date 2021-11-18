---
layout: post
title:  "k3s on a Raspberry Pi 4 at home, Part 2"
date:   2021-11-12 12:00:00 +0100
tags:   raspberrypi k3s
---

This is part 2 of our small series of posts, where we install Kubernetes (in the form of k3s) on a Raspberry Pi 4 (or similar device) at home.

* TOC Placeholder
{:toc}

## Some kubernetes basics

Feel free to skip this chapter if you already used kubernetes.

You use the `kubectl` command to interact with your Kubernetes cluster.

K8s resources are divided into **namespaces**. If you do not specify a namespace, the _default_ namespace will be used. Usually you want to separate different applications into different namespaces.

The hardware (or VM, in some cases) where k8s runs on is called a **node**. Usually you have at least 3 nodes for redundancy, but we only have one.

Apps run in **pods**. These pods can be created manually, but usually they are created automatically by a **deployment**, which defines not only a pod but also rules on how many of that same pod will be deployed and how and when they will be restarted.

Usually Apps listen on some port for network connections. Since these ports (and the nodes they run on) can change, the ports are abstracted into **services**. Services make apps accessible to other apps in the cluster.

To make a service accessible from _outside_ the cluster, you use an **ingress**. An ingress also attaches a hostname to the service, and in our case, has TLS certificates attached.

To store sensitive data, like credentials, you can use **secrets**. This keeps them away from your deployments and pods, and can optionally be used for segregation of duty.

For repeated scheduled tasks, k8s has **cronjobs**. They define a cron-like schedule and a template for a **job**. When the schedule triggers, a new job is created from the template, and it is this job that does the actual work. Old jobs that ran successfully are automatically cleaned up after a while.

To query all objects of a certain type, you use the following command

    kubectl [ -n namespace ] get { object }

If you want to query only a single object, you can specify it after the command, in one of two possible ways:

    kubectl [ -n namespace ] get { object } { identifier }
    kubectl [ -n namespace ] get { object }/{ identifier }

You can optionally add the flag `-o wide` to get more columns, or `-o yaml` or `-o json` to specify the output format. Also note that the object itself can be singular or plural it it stands alone, but must be singular when used with the second `{ object }/{ identifier }` syntax.

    kubectl -n my-namespace get pods
    kubectl -n my-namespace get pod my-pod-01 -o wide
    kubectl -n my-namespace get pod/my-pod-01 -o wide

You can specify the namespace that you want to query with `-n namespace`. Or you can specify `-A` or `--all-namespaces` to query a certain object type across all namespaces.

    kubectl get --all-namespaces pods

The `-n namespace`, `-A` and `--all-namespaces` option can be used for all the following commands, so they will not be mentioned again.

If you want to get some more details about the object, you can use `kubectl describe`. The remaining syntax is the same as above.

    kubectl describe { object } { identifier }
    kubectl describe { object }/{ identifier }

Unless otherwise specified, the two different syntaxes, `{ object } { identifier }` and `{ object }/{ identifier }`, are available for all commands, so only one will be mentioned in the examples below.

You can create simple objects (usually namespaces) via

    kubectl create namespace { identifier }

You can also delete objects:

    kubectl delete { object }/{ identifier }

Deleting a namespace will delete all resources in that namespace as well, so be careful not to delete anything you might need later.
{: .warning}

And you can in-place edit most objects, although you cannot change all properties of them. For some things you need to delete and re-create them:

    kubectl edit { object }/{ identifier }

Some objects (mainly pods and jobs) have logs that you can view. To do that use the following command:

    kubectl logs { object }/{ identifier }

The logs command is one of the few commands where you have to use the `{ object }/{ identifier }` syntax. If you don't specify an object type, it will default to pods.

The most important command, however, is `kubectl apply`. In practice, you mainly use this command to interact with the objects of your cluster. The apply command takes a YAML file and creates or updates the resource (or resources) defined in this file. When you later edit and change the file, you just use `kubectl apply` again.

    kubectl apply -f { filename.yaml }

The last command, which we will only briefly mention (because we will need it in a later part of the series), is `kubectl exec` which is used to execute a program (often a shell) inside a running container. If the pod of the application sonsists of only one container, you can also specify the pod where you want to execute a command, but this is just a shortcut: you never execute a command in a _pod_, you always execute them in a _container_.

    kubectl exec -it { name-of-pod } -- { name-of-command }

For example, to run a shell in an nginx container, you could use the following command

    kubectl exec -it nginx -- /bin/bash

Note that you need to know what binaries are available inside the container, and the full path to them. If your container does not have a shell anywhere, you can of course not execute one.

## Installing k3s on the RaspberryPi

We are now ready to install and configure k3s on the RaspberryPi. There is not much to do as most of it is automated.

The official installation method, by using `curl -sL https://get.k3s.io | sudo sh -`, is a dangerous practice and while it is safe in this case, it is best avoided.
{: .warning}

First, we fetch the install script:

    curl -sLo install-k3s.sh https://get.k3s.io

then execute it as root:

    pi@raspberrypi:~ $ sudo sh install-k3s.sh
    [INFO]  Finding release for channel stable
    [INFO]  Using v1.21.5+k3s2 as release
    [INFO]  Downloading hash https://github.com/k3s-io/k3s/releases/download/v1.21.5+k3s2/sha256sum-rm64.txt
    [INFO]  Downloading binary https://github.com/k3s-io/k3s/releases/download/v1.21.5+k3s2/k3s
    [INFO]  Verifying binary download
    [INFO]  Installing k3s to /usr/local/bin/k3s
    [INFO]  Skipping installation of SELinux RPM
    [INFO]  Creating /usr/local/bin/kubectl symlink to k3s
    [INFO]  Creating /usr/local/bin/crictl symlink to k3s
    [INFO]  Creating /usr/local/bin/ctr symlink to k3s
    [INFO]  Creating killall script /usr/local/bin/k3s-killall.sh
    [INFO]  Creating uninstall script /usr/local/bin/k3s-uninstall.sh
    [INFO]  env: Creating environment file /etc/systemd/system/k3s.service.env
    [INFO]  systemd: Creating service file /etc/systemd/system/k3s.service
    [INFO]  systemd: Enabling k3s unit
    Created symlink /etc/systemd/system/multi-user.target.wants/k3s.service → /etc/systemd/system/k3s.service.
    [INFO]  systemd: Starting k3s

The script installs an alias for the `kubectl` command, however, by default that is only callable by root:

    pi@raspberrypi:~ $ kubectl get nodes
    WARN[2021-11-12T14:25:04 +01:00] Unable to read /etc/rancher/k3s/k3s.yaml, please start server with
        --write-kubeconfig-mode to modify kube config permissions
    error: error loading config file "/etc/rancher/k3s/k3s.yaml": open /etc/rancher/k3s/k3s.yaml: permission denied

To let regular users run `kubectl`, we need to allow read-access to the k3s config file. This can be done by setting an option in the systemd environment file for the k3s service:

    pi@raspberrypi:~ $ echo "K3S_KUBECONFIG_MODE=0644" | sudo tee /etc/systemd/system/k3s.service.env
    K3S_KUBECONFIG_MODE=0644
    pi@raspberrypi:~ $ sudo systemctl restart k3s

Now, a `kubectl get nodes` should return one node, even when run as non-root user:

    pi@raspberrypi:~ $ kubectl get nodes
    NAME            STATUS   ROLES                  AGE     VERSION
    raspberrypi     Ready    control-plane,master   6m51s   v1.21.5+k3s2

## Create the Dynamic DNS Updater cronjobs

We could just set up regular cronjobs in Linux to update our Dynamic DNS, say, every hour. But where would be the fun in that? Especially since you can just as easily (well, almost) create Cronjobs in k8s too. So let's do that instead.

First we create a new namespace that will pick up everything we need for our cronjobs. This is always a good idea as it allows a clean separation between unrelated things in k8s.

Create the following file and save it as `dyndns-namespace.yaml`:

{% highlight yaml %}
apiVersion: v1
kind: Namespace
metadata:
  name: dyndns-updater
{% endhighlight %}

Apply it with `kubectl apply -f dyndns-namespace.yaml`:

    pi@raspberrypi:~ $ kubectl apply -f dyndns-namespace.yaml
    namespace/dyndns-updater created

Note that you could have also done a simple `kubectl create namespace dyndns-updater`, but we want everything to be repeatable later, so it's good if we have all configuration changes as yaml files.

Now we create the two secrets which will hold our DynDNS hosts and passwords. We can do this as two separate files, or in one file. Create a file named `dyndns-secrets.yaml` with the following content:

{% highlight yaml %}
apiVersion: v1
kind: Secret
metadata:
  namespace: dyndns-updater
  name: dyndns-web-secret
stringData:
  hostname: web.dark.star
  password: 1secret234!
---
apiVersion: v1
kind: Secret
metadata:
  namespace: dyndns-updater
  name: dyndns-wp-secret
stringData:
  hostname: wp.dark.star
  password: 2secret567!
{% endhighlight %}

Make sure you copy the correct passwords from your DNS hoster's website. Note also the names of these two secrets, `dyndns-web-secret` and `dyndns-wp-secret`, by which we will reference them later. Each secret contains two key-value pairs, with `hostname` and `password` as keys. This will also become important soon.

Apply these secrets to your cluster:

    pi@raspberrypi:~ $ kubectl apply -f dyndns-secrets.yaml
    secret/dyndns-web-secret created
    secret/dyndns-wp-secret created

Storing passwords as secrets instead of hardcoding them in scripts or deployments later on has the benefit of improved security: You can restrict who will be able to access those secrets, and a developer who wants to use these secrets does not necessarily have to know them to be able to use them.
Currently, though, we are doing all kubernetes configuration as global k3s admin, so we have access to everything anyway. In a production environment you would want to differentiate access for different kinds of users or developers. But storing the secrets separately still has benefits in our case, as you cannot "accidentally" read the secrets (try `kubectl get secret dyndns-web-secret -n dyndns-updater -o yaml`), and you can edit and modify the cronjob later without anyone else who might be looking over your shoulder seeing any sensitive information.

Now we will create two CronJobs, one for each host. Save the following file as `dyndns-web-job.yaml`:

{% highlight yaml %}
apiVersion: batch/v1
kind: CronJob
metadata:
  namespace: dyndns-updater
  name: dyndns-web-job
spec:
  schedule: "13 * * * *"
  jobTemplate:
    spec:
      template:
        spec:
          containers:
          - name: cron
            image: busybox
            imagePullPolicy: IfNotPresent
            env:
              - name: SECRET_HOSTNAME
                valueFrom:
                  secretKeyRef:
                    name: dyndns-web-secret
                    key: hostname
              - name: SECRET_PASSWORD
                valueFrom:
                  secretKeyRef:
                    name: dyndns-web-secret
                    key: password
              - name: URL
                value: "https://dynamicdns.key-systems.net/update.php"
            command:
            - /bin/sh
            - -c
            - wget -O - "${URL}?hostname=${SECRET_HOSTNAME}&password=${SECRET_PASSWORD}&ip=auto"
          restartPolicy: OnFailure
{% endhighlight %}

Here, we basically define the `dyndns-web-job` CronJob to run at thirteen minutes past every hour (the `schedule:` key), and when it runs it will instantiate a new BusyBox container with three environment variables set (two from our secret above, and one set statically just to make the command shorter later on).
Also, the actual command we run in that container will be overwritten by `/bin/sh -c wget ...`, which will just launch wget instead of running an interactive BusyBox-shell.

That particular wget call with `-O -` will output all that it receives to stdout, where we can inspect it later for troubleshooting.

Apply this cronjob to your cluster too:

    pi@raspberrypi:~ $ kubectl apply -f dyndns-web-job.yaml
    cronjob.batch/dyndns-web-job created

Now, to see if this works, we might have to wait quite some time until it is triggered the next time. Or we can manually trigger it by creating a Job from the CronJob:

    pi@raspberrypi:~ $ kubectl -n dyndns-updater create job --from=cronjob/dyndns-web-job testjob1
    job.batch/testjob1 created

To see if the job went through, let's just describe it:

    pi@raspberrypi:~$ kubectl -n dyndns-updater describe job/testjob1
    Name:           testjob1
    Namespace:      dyndns-updater
    Selector:       controller-uid=219fa210-0e7a-4f42-926b-ddf4a73edc9e
    Labels:         controller-uid=219fa210-0e7a-4f42-926b-ddf4a73edc9e
                    job-name=testjob1
    Annotations:    cronjob.kubernetes.io/instantiate: manual
    Parallelism:    1
    Completions:    1
    Start Time:     Fri, 12 Nov 2021 16:42:55 +0100
    Completed At:   Fri, 12 Nov 2021 16:42:59 +0100
    Duration:       4s
    Pods Statuses:  0 Running / 1 Succeeded / 0 Failed
    Pod Template:
      Labels:  controller-uid=219fa210-0e7a-4f42-926b-ddf4a73edc9e
               job-name=testjob1
      Containers:
       cron:
        Image:      busybox
        Port:       <none>
        Host Port:  <none>
        Command:
          /bin/sh
          -c
          wget -O - "${URL}?hostname=${SECRET_HOSTNAME}&password=${SECRET_PASSWORD}&ip=auto"
        Environment:
          SECRET_HOSTNAME:  <set to the key 'hostname' in secret 'dyndns-web-secret'>  Optional: false
          SECRET_PASSWORD:  <set to the key 'password' in secret 'dyndns-web-secret'>  Optional: false
          URL:              https://dynamicdns.key-systems.net/update.php
        Mounts:             <none>
      Volumes:              <none>
    Events:
      Type    Reason            Age   From            Message
      ----    ------            ----  ----            -------
      Normal  SuccessfulCreate  51s   job-controller  Created pod: testjob1-dplw5
      Normal  Completed         47s   job-controller  Job completed

We see at the end that the job completed successfully. We can also request the actual logs by using `kubectl -n dyndns-updater logs job/testjob1` but that might include the hostname and password in clear text, as it will be present in the URL that wget might logs to stdout (not in our case, but if you add a `-v` option to the wget command line for troubleshooting, it would be present):

    pi@raspberrypi:~$ kubectl -n dyndns-updater logs job/testjob1
    wget: note: TLS certificate validation not implemented
    [RESPONSE]
    code = 200
    description = Command completed successfully
    queuetime = 0
    runtime = 0.099
    EOF

The cronjob will automatically clean up (i.e. delete) old jobs and only keep the most recent 3 (per default) for you to inspect, should the jobs suddenly start to fail. You can see the list of recent jobs by using `kubectl get all -n dyndns-updater`:

    pi@raspberrypi:~ $ kubectl get all -n dyndns-updater
    NAME                                READY   STATUS      RESTARTS   AGE
    pod/testjob1-dplw5                  0/1     Completed   0          90m
    pod/dyndns-web-job-27278931-5rn33   0/1     Completed   0          60s
    pod/dyndns-web-job-27278893-9n9d2   0/1     Completed   0          41s
    
    NAME                           SCHEDULE     SUSPEND   ACTIVE   LAST SCHEDULE   AGE
    cronjob.batch/dyndns-web-job   13 * * * *   False     0        41s             92m
    
    NAME                                COMPLETIONS   DURATION   AGE
    job.batch/testjob1                  1/1           4s         90m
    job.batch/dyndns-web-job-27278931   1/1           1s         60s
    job.batch/dyndns-web-job-27278893   1/1           1s         41s


As an excercise, you will now create the second cronjob definition for the `wp.dark.star` subdomain. Copy the file `dyndns-web-job.yaml` to `dyndns-wp-job.yaml` and edit it. Change the cronjob's name in the metadata section, the schedule in the spec section (we don't want to run both commands at the same time), and change all references to `dyndns-web-secret` so that they reference `dyndns-wp-secret` instead. You can leave the rest of the file as-is. Apply your file to the cluster, create a test run and inspect the result of your cronjob.
{: .tip}

## Install cert-manager for automatic LetsEncrypt certificates

Now we need to install the cert-manager, which will be used to automatically generate Let's Encrypt certificates for our ingresses. Note that even though Traekik, the default network stack in k3s, can technically do that as well, it is not as sophisticated as cert-manager (for example, Traefik stores the certificates in regular files instead of Kubernetes secrets).

First we will create a new namespace for the cert-manager. Save this as `cert-manager-namespace.yaml` and apply it to your cluster:

{% highlight yaml %}
apiVersion: v1
kind: Namespace
metadata:
  name: cert-manager
{% endhighlight %}

    pi@raspberrypi:~ $ kubectl apply -f cert-manager-namespace.yaml
    namespace/cert-manager created

Since cert-manager is a rather big project, we will be downloading the YAML file to deploy it directly from its GitHub repository. However, that YAML file is preconfigured for the x86_64 architecture, so we need to modify it for arm64 to be usable on the RaspberryPi. The following two commands accomplish this:

    curl -sLO https://github.com/jetstack/cert-manager/releases/download/v1.3.1/cert-manager.yaml
    sed -r 's/(image:.*):(v.*)$/\1-arm64:\2/g' cert-manager.yaml > cert-manager-arm64.yaml

The file `cert-manager-arm64.yaml` will have all image names replaced by their arm64 variants. After this you can delete the `cert-manager.yaml` file as it will not be needed anymore.

If you are _not_ using a RaspberryPi, you will need the original file, and that file only. In that case, skip the `sed` command above and continue with the `cert-manager.yaml` file.
{: .notice}

We are using cert-manager version 1.3.1 above, as that is what I originally tested with. It should be no problem to replace that with the most recent 1.4 or 1.5 version, and probably even with 1.6. I will test that in my lab setup and change the above instructions accordingly after I successfully tested a newer version.
{: .notice}

Now, install cert-manager into the cluster. Many objects will be created and it will take a while util all pods are up and running:

    pi@raspberrypi:~ $ kubectl apply -f cert-manager-arm64.yaml 
    customresourcedefinition.apiextensions.k8s.io/certificaterequests.cert-manager.io created
    customresourcedefinition.apiextensions.k8s.io/certificates.cert-manager.io created
    customresourcedefinition.apiextensions.k8s.io/challenges.acme.cert-manager.io created
    customresourcedefinition.apiextensions.k8s.io/clusterissuers.cert-manager.io created
    customresourcedefinition.apiextensions.k8s.io/issuers.cert-manager.io created
    customresourcedefinition.apiextensions.k8s.io/orders.acme.cert-manager.io created
    namespace/cert-manager configured
    serviceaccount/cert-manager-cainjector created
    serviceaccount/cert-manager created
    serviceaccount/cert-manager-webhook created
    clusterrole.rbac.authorization.k8s.io/cert-manager-cainjector created
    clusterrole.rbac.authorization.k8s.io/cert-manager-controller-issuers created
    clusterrole.rbac.authorization.k8s.io/cert-manager-controller-clusterissuers created
    clusterrole.rbac.authorization.k8s.io/cert-manager-controller-certificates created
    clusterrole.rbac.authorization.k8s.io/cert-manager-controller-orders created
    clusterrole.rbac.authorization.k8s.io/cert-manager-controller-challenges created
    clusterrole.rbac.authorization.k8s.io/cert-manager-controller-ingress-shim created
    clusterrole.rbac.authorization.k8s.io/cert-manager-view created
    clusterrole.rbac.authorization.k8s.io/cert-manager-edit created
    clusterrole.rbac.authorization.k8s.io/cert-manager-controller-approve:cert-manager-io created
    clusterrole.rbac.authorization.k8s.io/cert-manager-webhook:subjectaccessreviews created
    clusterrolebinding.rbac.authorization.k8s.io/cert-manager-cainjector created
    clusterrolebinding.rbac.authorization.k8s.io/cert-manager-controller-issuers created
    clusterrolebinding.rbac.authorization.k8s.io/cert-manager-controller-clusterissuers created
    clusterrolebinding.rbac.authorization.k8s.io/cert-manager-controller-certificates created
    clusterrolebinding.rbac.authorization.k8s.io/cert-manager-controller-orders created
    clusterrolebinding.rbac.authorization.k8s.io/cert-manager-controller-challenges created
    clusterrolebinding.rbac.authorization.k8s.io/cert-manager-controller-ingress-shim created
    clusterrolebinding.rbac.authorization.k8s.io/cert-manager-controller-approve:cert-manager-io created
    clusterrolebinding.rbac.authorization.k8s.io/cert-manager-webhook:subjectaccessreviews created
    role.rbac.authorization.k8s.io/cert-manager-cainjector:leaderelection created
    role.rbac.authorization.k8s.io/cert-manager:leaderelection created
    role.rbac.authorization.k8s.io/cert-manager-webhook:dynamic-serving created
    rolebinding.rbac.authorization.k8s.io/cert-manager-cainjector:leaderelection created
    rolebinding.rbac.authorization.k8s.io/cert-manager:leaderelection created
    rolebinding.rbac.authorization.k8s.io/cert-manager-webhook:dynamic-serving created
    service/cert-manager created
    service/cert-manager-webhook created
    deployment.apps/cert-manager-cainjector created
    deployment.apps/cert-manager created
    deployment.apps/cert-manager-webhook created
    mutatingwebhookconfiguration.admissionregistration.k8s.io/cert-manager-webhook created
    validatingwebhookconfiguration.admissionregistration.k8s.io/cert-manager-webhook created

You can check the status of the cert-manager pods and see if they are all running:

    pi@raspberrypi:~ $ kubectl get pods -n cert-manager
    NAME                                       READY   STATUS    RESTARTS   AGE
    cert-manager-cainjector-64c949654c-2ls5r   1/1     Running   0          101s
    cert-manager-7dd5854bb4-6j4k8              1/1     Running   0          101s
    cert-manager-webhook-6b57b9b886-mfktj      1/1     Running   0          101s

If one of the pods will not come up, even after a few minutes, try finding out why by using `kubectl describe` and `kubectl logs` on the pod.

## Create certificate issuers for staging and prod

Cert-manager uses so-called **Issuers** to generate certificates. These describe where/how certificates can be requested, signed and retrieved. Regular Issuers only work within a single namespace, while **ClusterIssuers** work within the whole cluster, which is what we want.

For testing purposes, we should use the Let's Encrypt _staging_ server, which has no rate limitations but doesn't generate trusted certificates. When this works well enough, we can switch to the _prod_ server for valid certificates.

To create a Cluster-Issuer, copy the following YMAL into a file called `letsencrypt-issuer-staging.yaml`, and remember to change the E-Mail address to a valid one:

{% highlight yaml %}
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-staging
spec:
  acme:
    # The ACME server URL
    server: https://acme-staging-v02.api.letsencrypt.org/directory
    # Email address used for ACME registration
    email: someone@example.com
    # Name of a secret used to store the ACME account private key
    privateKeySecretRef:
      name: letsencrypt-staging-secret
    # Enable the HTTP-01 challenge provider
    solvers:
    - http01:
        ingress:
          class: traefik
{% endhighlight %}

Apply the file to your cluster and wait for it to become ready:

    pi@raspberrypi:~ $ kubectl apply -f letsencrypt-issuer-staging.yaml
    clusterissuer.cert-manager.io/letsencrypt-staging created
    
    pi@raspberrypi:~ $ kubectl get clusterissuer
    NAME                  READY   AGE
    letsencrypt-staging   True    2m8s

To test if cert-manager can correctly issue certificates, we will have it issue a test certificate for one of our domains. Save the following YAML file as `letsencrypt-test-certificate.yaml` and remember editing the hostnames:

{% highlight yaml %}
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: web-dark-star
  namespace: default
spec:
  secretName: web-dark-star-tls
  issuerRef:
    # Use the staging issuer for the test certificate
    name: letsencrypt-staging
    kind: ClusterIssuer
  commonName: web.dark.star
  dnsNames:
  - web.dark.star
{% endhighlight %}

Apply it to the cluster and check if the certificate was correctly generated.

    pi@raspberrypi:~ $ kubectl apply -f letsencrypt-test-certificate.yaml
    certificate.cert-manager.io/web-dark-star created
    
    pi@raspberrypi:~ $ kubectl get certificates
    NAME                READY   SECRET                  AGE
    web-dark-star       True    web-dark-star-tls       21s

If the READY column doesn't show `True` after a couple of seconds or minutes, you need to investigate. Start by using `kubectl describe certificate web-dark-star` and check the last lines in the log.

    ...
    Events:
      Type    Reason     Age   From          Message
      ----    ------     ----  ----          -------
      Normal  Issuing    34s   cert-manager  Issuing certificate as Secret does not exist
      Normal  Generated  32s   cert-manager  Stored new private key in temporary Secret resource "web-dark-star-m72zw"
      Normal  Requested  32s   cert-manager  Created new CertificateRequest resource "web-dark-star-b72sm"

Just walk the chain of Kubernetes resources from here on, i.e. next use `kubectl describe certificaterequest web-dark-star-b72sm`, which will probably point you towards an **order**, which leads you to a **challenge**, etc. At some point, you should be able to see an error in the log, for example _no such host_ or something similar.

If the certificate has been issued correctly, you can delete it again

    pi@raspberrypi:~ $ kubectl delete certificate web-dark-star
    certificate.cert-manager.io "web-dark-star" deleted

As an excercise, you can now create the Let's Encrypt Production Certificate Issuer. Copy the file `letsencrypt-staging-issuer.yaml` to `letsencrypt-prod-issuer.yaml` and edit it. Change the issuer's name in the metadata section to `letsencrypt-prod`, change the PrivateKeySecretRef's name in a similar way, and change the `server:` URL to `https://acme-v02.api.letsencrypt.org/directory`. Then save and apply the file to your cluster. Note that if you decide on testing it, make sure that you don't trigger any rate limits as that might lock you out of Let's Encrypt for a few hours or even days.
{: .tip}

This concludes our second part. In the next part we will set up our first actual website and make it accessible via HTTP and HTTPS. See you there!
