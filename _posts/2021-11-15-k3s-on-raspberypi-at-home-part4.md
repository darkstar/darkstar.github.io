---
layout: post
title:  "k3s on a Raspberry Pi 4 at home, Part 4"
date:   2021-11-15 12:00:00 +0100
tags:   raspberrypi k3s
---

This is part four of our series of setting up k3s on a Raspberry Pi at home. In this part we will focus on getting WordPress up and running.

* TOC Placeholder
{:toc}

## Scenario 2: Creating the WordPress backend infrastructure

For our second scenario, we will need a 2-tier architecture: The WordPress webserver needs a database in the backend to store its data.

First we will create a new namespace for all our WordPress workloads.

    pi@raspberrypi:~ $ kubectl create namespace wp
    namespace/wp created

For MySQL (or rather MariaDB), we will need a root password. For simplicity, we will use the same random password for the WordPress user later. Save the following as `wp-secret.yaml`:

{% highlight yaml %}
apiVersion: v1
kind: Secret
metadata:
  namespace: wp
  name: mariadb-secret
stringData:
  password: 843j5lkj3ufk2j4n
{% endhighlight %}

This password will be used by the respective Pods later, and the administrator doesn't have to remember it since both sides (frontend and backend) will both pull it from this secret.

The next part we need is the MariaDB backend itself. This, again, needs a volume to store its database files, for which we will use another fixed hostPath-type volume, but also the secret we just created, which we can pull into an environment variable. Create the file `wp-mariadb-deployment.yaml` with the following content:

{% highlight yaml %}
apiVersion: apps/v1
kind: Deployment
metadata:
  name: wordpress-mariadb
  namespace: wp
  labels:
    app: wordpress
spec:
  replicas: 1
  selector:
    matchLabels:
      app: wordpress
      tier: mariadb
  strategy:
    type: Recreate
  template:
    metadata:
      labels:
        app: wordpress
        tier: mariadb
    spec:
      containers:
      - name: mariadb
        image: mariadb:10.6
        env:
        - name: MARIADB_ROOT_PASSWORD
          valueFrom:
            secretKeyRef:
              name: mariadb-secret
              key: password
        ports:
        - containerPort: 3306
          name: mariadb
        volumeMounts:
        - mountPath: /var/lib/mysql
          name: mariadb-data
      volumes:
      - name: mariadb-data
        hostPath:
          path: /k8s-data/wp.dark.star/mariadb
          type: Directory
{% endhighlight %}

This deployment looks similar to our previous deployment. We only have one replica (this is important, otherwise we would have two different, diverging databases in the backend), we are using the MariaDB 10.6 container from DockerHub, and we populate the MariaDB root password from our secret. We also define the local directory where the database files are stored. This directory must be created first. And while we're at it, we can also create the directory for the frontend that we will need later:

    pi@raspberrypi:~ $ sudo mkdir -p /k8s-data/wp.dark.star/mariadb
    pi@raspberrypi:~ $ sudo mkdir -p /k8s-data/wp.dark.star/wordpress

The service that goes along with the deployment is again pretty straightforward (`wp-mariadb-service.yaml`):

{% highlight yaml %}
apiVersion: v1
kind: Service
metadata:
  name: wordpress-mariadb
  namespace: wp
  labels:
    app: wordpress
spec:
  ports:
    - protocol: TCP
      port: 3306
  selector:
    app: wordpress
    tier: mariadb
  clusterIP: None
{% endhighlight %}

The only interesting thing in the service definition is the `clusterIP: none` line, which makes sure that our service will not receive a stable cluster IP.

This is called a _headless service_ in Kubernetes. What Kubernetes does in this case is, it creates DNS aliases as for regular services, but the endpoints are the IP addresses of the pods themselves instead of the single service IP. This saves a little bit of iptables trickery, as the whole virtual Cluster-IP does not need to be set up. If another pod requests the DNS entry of the service, they will receive all DNS entries of all backend pods, and need to do load balancing/client selection themselves, rather than having the service do it for them. This is described very well [on this page][headless_service].

[headless_service]: https://stackoverflow.com/questions/52707840/what-is-a-headless-service-what-does-it-do-accomplish-and-what-are-some-legiti

## Create the actual database and a user for WordPress

A small "problem" with WordPress is, that is it not able to create a database in the backend by itself. So the administrator has to create an empty database first, and also create a user with full access to the database.

Yes, there are other ways of creating an initial database, and we will be using a different method in the next part. But since this is a "how I learned about k8s"-style post, I thought it would be interesting to also include the more complicated ways I came up with for solving some issues, instead of just providing a clean, easy way.
{: .notice }

Our Database is already up and running, but it's empty, so how can we create a database inside that running pod?

Of course with the `kubectl exec` command. But what if we do not know the root-password that was used for the MariaDB database? With a few tricks, we can get that out of the Kubernetes secret. This is why we installed the `jq` tool back in part 1.

    pi@raspberrypi:~ $ PASSWORD=$(kubectl -n wp get secret mariadb-secret -o json|jq -r '.data.password' | base64 -d)
    pi@raspberrypi:~ $ POD=$(kubectl get pods -n wp -l tier=mariadb -o json | jq -r '.items[0].metadata.name')
    pi@raspberrypi:~ $ echo $POD
    wordpress-mariadb-5fc5c5ddbc-qnwnd

You can check the password in a similar way. Generally, getting k8s resources as json and querying them via `jq` is a good way of extracting particular data in a structured way.

Now we can run the MariaDB command line client tool, `mysql`, inside the MariaDB pod:

    pi@raspberrypi:~ $ kubectl exec -it -n wp ${POD} -- mysql -u root --password=${PASSWORD}
    Welcome to the MariaDB monitor.  Commands end with ; or \g.
    Your MariaDB connection id is 2106
    Server version: 10.6.4-MariaDB-1:10.6.4+maria~focal mariadb.org binary distribution
    
    Copyright (c) 2000, 2018, Oracle, MariaDB Corporation Ab and others.
    
    Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.
    
    MariaDB [(none)]>

That was easy! Press Ctrl+D or type "quit" to leave the CLI tool. To actually create a new user with our password, we either need to actually show (and copy) the password, or we need to execute a non-interactive command. For the purpose of this tutorial, we will go with the second option to avoid repeating the password, but you might go with the first option if you wish.

The SQL command to create a new user with a particular password is `create user "<name>" identified by "<password>";`. We will call our user `wp-user`. To execute this against the database, run the following command:

    pi@raspberrypi:~ $ kubectl exec -it -n wp ${POD} -- mysql -u root --password=${PASSWORD} \
         -e "create user \"wp-user\" identified by \"${PASSWORD}\";"

Take note of the escaped quotes and the `;` at the end. If everything was successful you should not get any output from that command.

If you prefer to execute the command from the MariaDB shell (see above), the syntax is similar but a bit easier:

    MariaDB [(none)]> create user "wp-user" identified by "<password>";

Now that we have a user, we still need a database, and we need to grant the user access to that database. You can, again, either use the MariaDB shell for that, or use a non-interactive `kubectl exec`:

    pi@raspberrypi:~ $ kubectl exec -it -n wp ${POD} -- mysql -u root --password=${PASSWORD}
    MariaDB [(none)]> create database wordpress;
    MariaDB [(none)]> use wordpress;
    Database changed
    MariaDB [wordpress]> grant all privileges on wordpress.* to "wp-user";
    Query OK, 0 rows affected (0.00 sec)

This is everything you need to create a new database called `wordpress`, create a user called `wp-user`, and give that user full access to the database.

## Create the WordPress web frontend

Now for the actual Web frontend, we again need a deployment and a service.

Here's the `wp-wordpress-service.yaml` file. Did you know that you can apply manifests (aka. YAML files) in any order, and Kubernetes will take care of all your dependencies? So let's start with the service first, this time:

{% highlight yaml %}
apiVersion: v1
kind: Service
metadata:
  name: wordpress
  namespace: wp
  labels:
    app: wordpress
spec:
  ports:
    - protocol: TCP
      port: 80
  selector:
    app: wordpress
    tier: frontend
{% endhighlight %}

Since we do not yet have an app that matches this services's selector, it will wait until it can find one. So now let's look at the deployment, `wp-wordpress-deployment.yaml`:

{% highlight yaml %}
apiVersion: apps/v1
kind: Deployment
metadata:
  name: wordpress
  namespace: wp
  labels:
    app: wordpress
spec:
  selector:
    matchLabels:
      app: wordpress
      tier: frontend
  strategy:
    type: Recreate
  template:
    metadata:
      labels:
        app: wordpress
        tier: frontend
    spec:
      containers:
      - image: wordpress:5-apache
        name: wordpress
        env:
        - name: WORDPRESS_DB_HOST
          value: wordpress-mariadb
        - name: WORDPRESS_DB_NAME
          value: wordpress
        - name: WORDPRESS_DB_PASSWORD
          valueFrom:
            secretKeyRef:
              name: mariadb-secret
              key: password
        - name: WORDPRESS_DB_USER
          value: wp-user
        ports:
        - containerPort: 80
          name: wordpress
        volumeMounts:
        - name: wordpress-persistent-storage
          mountPath: /var/www/html
      volumes:
      - name: wordpress-persistent-storage
        hostPath:
          path: /k8s-data/wp.dark.star/wordpress
          type: Directory
{% endhighlight %}

This deployment uses the WordPress container in version 5 with the Apache2 web server, as evidenced by the use of the image name `wordpress:5-apache`. We also populate some environment variables, some with fixed data, and one (the password) with data from our secret from before. Also, the volume mounted at `/var/www/html`, which is where the WordPress container stores/expects all of its web site data, will be backed by a local path on our USB stick.

    pi@raspberrypi:~ $ kubectl apply -f wp-wordpress-deployment.yaml
    deployment/wordpress created

Now check the service to see that it is up

    pi@raspberrypi:~ $ kubectl -n wp get service/wordpress -o wide
    NAME        TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)   AGE   SELECTOR
    wordpress   ClusterIP   10.43.88.122   <none>        80/TCP    10d   app=wordpress,tier=frontend

Now all that's left is the ingress, this time we will use the LetsEncrypt _prod_ server immediately, without going through the staging first to test it.

Here is the ingress definition, the file `wp-wordpress-ingress.yaml`:

{% highlight yaml %}
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: wp-nginx-ingress
  namespace: wp
  annotations:
    kubernetes.io/ingress.class: "traefik"
    cert-manager.io/cluster-issuer: letsencrypt-prod
spec:
  rules:
  - host: wp.dark.star
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: wordpress
            port:
              number: 80
  tls:
  - hosts:
    - wp.dark.star
    secretName: wp-dark-star-tls
{% endhighlight %}

Apply this to the cluster and wait a few seconds for the certificate enrollment to complete:

    pi@raspberrypi:~ $ kubectl apply -f wp-wordpress-ingress.yaml
    ingress/wp-nginx-ingress created
    pi@raspberrypi:~ $ kubectl get -n wp ingress
    NAME               CLASS    HOSTS          ADDRESS       PORTS     AGE
    wp-nginx-ingress   <none>   wp.dark.star   192.168.1.3   80, 443   36s

Try accessing your site, it should work with both `http://wp.dark.star` and `https://wp.dark.star`.

In the next part we will set up a NextCloud instance on our RaspberryPi.

