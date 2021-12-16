---
layout: post
title:  "k3s on a Raspberry Pi 4 at home, Part 5"
date:   2021-11-20 12:00:00 +0100
tags:   raspberrypi k3s
---

In this part, part 5, we will try and set up NextCloud in our cluster. This is a bit more involved but we'll get there eventually.

* TOC Placeholder
{:toc}

## Scenario 3: NextCloud backend database

There are multiple ways to deploy a NextCloud instance to a Kubernetes cluster. The easiest and most flexible way would probably be the [official helm chart][nextcloud_helm_chart]. However, we will opt for the manual deployment similar to what we did in the previous two parts, just to have one more example of how to manually deploy something to a k8s cluster, without introducing another "magic" tool to do that job for us. We will probably take a look at using helm in a later part of this series.
{: .notice}

NextCloud can be deployed with an integrated SQLite database, but for our purposes, we will use, again, a MySQL (or rather MariaDB) instance as backend.

In this case we actually need a different MariaDB version as for the WordPress installation in the last part. This is because all MariaDB versions from 10.6 onwards have deprecated a feature that NextCloud relies on (InnoDB compressed rows, as shown in [this][mariadb_compressed] bug report), and this would require us to implement a workaround by manually changing the MariaDB config file.

Using the MariaDB 10.5 image is simpler, and since 10.5 is still fully supported for a while, we can use this until the NextCloud developers implement a proper migration path for the database.

Our new NextCloud instance is expected to go live on the `https://cloud.dark.star` URL. So the first thing we do is register that DNS alias with our DNS provider and create the cronjobs, the same way we did for the other domains in part 2.

Create the files `dyndns-updater-cloud-secret.yaml` and `dyndns-updater-cloud-cronjob.yaml`, taking one of the other YAML files from part 2 as base, changing all references from either `wp` or `web` (depending on what you used as base) to `cloud`, and changing the cron schedule to another minute in the hour. Then apply these to the cluster and see if they work by running the cronjob at least once, as shown in part 2.
{: .tip}

Here is `cloud-namespace.yaml` for our new namespace:

{% highlight yaml %}
apiVersion: v1
kind: Namespace
metadata:
  name: cloud
{% endhighlight %}

As for the secrets, this time we will be using one secret that contains two different passwords, one for the MariaDB root user and one for the actual `nextcloud` user that will access the database from the frontend. Here is the `cloud-secret.yaml` file:

{% highlight yaml %}
apiVersion: v1
kind: Secret
metadata:
  namespace: cloud
  name: mariadb-secret
stringData:
  rootpassword: secret123password456
  userpassword: user1password2cloud
{% endhighlight %}

Apply these two files to the cluster:

    pi@raspberrypi:~ $ kubectl apply -f cloud-namespace.yaml
    namespace/cloud created
    pi@raspberrypi:~ $ kubectl apply -f cloud-secret.yaml
    secret/mariadb-secret created

Now we need two data directories on our USB stick, one for the database backend, and one for the web frontend (and the actual files stored in NextCloud):

    pi@raspberrypi:~ $ sudo mkdir -p /k8s-data/cloud.dark.star/{db,nextcloud}

The deployment for the backend database also has some things that differ from our previous database deployment, so here is `cloud-db-deployment.yaml`:

{% highlight yaml %}
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: nextcloud
    tier: backend
  name: mariadb
  namespace: cloud
spec:
  replicas: 1
  strategy:
    type: Recreate
  selector:
    matchLabels:
      app: nextcloud
      tier: backend
  template:
    metadata:
      labels:
        app: nextcloud
        tier: backend
    spec:
      containers:
      - name: mariadb
        image: mariadb:10.5
        env:
        - name: MYSQL_DATABASE
          value: nextcloud
        - name: MYSQL_PASSWORD
          valueFrom:
            secretKeyRef:
              name: mariadb-secret
              key: userpassword
        - name: MYSQL_USER
          value: nextcloud
        - name: MYSQL_ROOT_PASSWORD
          valueFrom:
            secretKeyRef:
              name: mariadb-secret
              key: rootpassword
        ports:
        - containerPort: 3306
          name: mariadb
        volumeMounts:
        - mountPath: /var/lib/mysql
          name: mariadb-data
      restartPolicy: Always
      volumes:
      - name: mariadb-data
        hostPath:
          path: "/k8s-data/cloud.dark.star/db"
          type: Directory
{% endhighlight %}

Some things to note:
* We label the database pods with the `app: nextcloud` and `tier: backend` labels, to distinguish them from the web server frontend (see below).
* For the container image, we explicitly select MariaDB 10.5, because of the issues mentioned above.
* We set the `MYSQL_DATABASE`, `MYSQL_USER`, `MYSQL_PASSWORD` and `MYSQL_ROOT_PASSWORD` environment variables to have MariaDB create a database for us automatically on startup, and grant a user full access to it. The two passwords come from different keys in the secret we deployed before.
* The database directory will be `/k8s-data/cloud.dark.star/db`

We could have done it like in part 4 and manually created the database, but I wanted to show multiple different ways of achieving the same end result, so that's why we did it differently this time[^1].

The service for the database will again be a headless service, we don't need the internal loadbalancer for it as we are pretty sure that we always only have one database backend pod running, so here is `cloud-db-service.yaml`:

{% highlight yaml %}
apiVersion: v1
kind: Service
metadata:
  name: mariadb
  namespace: cloud
  labels:
    app: nextcloud
spec:
  ports:
    - protocol: TCP
      port: 3306
  selector:
    app: nextcloud
    tier: backend
  clusterIP: None
{% endhighlight %}

Apply it to the cluster and check (with `kubectl logs`) if the database server starts up. You can also check the directory `/k8s-data/cloud.dark.star/db` to see if there is a subdirectory with the name `nextcloud` created, which corresponds to the database of the same name that the pod automatically created when it started up.

    pi@raspberrypi:~ $ kubectl apply -f cloud-db-deployment.yaml
    deployment/mariadb created
    pi@raspberrypi:~ $ kubectl apply -f cloud-db-service.yaml
    service/mariadb created
    pi@raspberrypi:~ $ ls -l /k8s-data/cloud.dark.star/db/ | grep ^d
    drwx------ 2 999 spi      4096 Nov 20 21:56 mysql
    drwx------ 2 999 spi     12288 Nov 20 22:05 nextcloud
    drwx------ 2 999 spi      4096 Nov 20 21:56 performance_schema

With this, our backend database is up and running.

[nextcloud_helm_chart]: https://github.com/nextcloud/helm/
[mariadb_compressed]: https://github.com/nextcloud/server/issues/25436

## Creating the web frontend

Now to the frontend. This is the `cloud-nc-deployment.yaml`:

{% highlight yaml %}
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: nextcloud
    tier: frontend
  name: nextcloud
  namespace: cloud
spec:
  replicas: 1
  strategy:
    type: Recreate
  selector:
    matchLabels:
      app: nextcloud
      tier: frontend
  template:
    metadata:
      labels:
        app: nextcloud
        tier: frontend
    spec:
      containers:
      - env:
        - name: TZ
          value: Europe/Berlin
        - name: DEBUG
          value: "false"
        - name: NEXTCLOUD_URL
          value: https://cloud.dark.star
        - name: NEXTCLOUD_UPLOAD_MAX_FILESIZE
          value: 4096M
        - name: NEXTCLOUD_MAX_FILE_UPLOADS
          value: "20"
        - name: MYSQL_DATABASE
          value: nextcloud
        - name: MYSQL_HOST
          value: mariadb
        - name: MYSQL_PASSWORD
          valueFrom:
            secretKeyRef:
              name: mariadb-secret
              key: userpassword
        - name: MYSQL_USER
          value: nextcloud
        - name: APACHE_DISABLE_REWRITE_IP
          value: "1"
        - name: TRUSTED_PROXIES
          value: 10.0.0.0/8
        name: nc
        image: nextcloud
        ports:
        - containerPort: 80
          protocol: TCP
        terminationMessagePath: /dev/termination-log
        terminationMessagePolicy: File
        volumeMounts:
        - mountPath: /var/www/html
          name: nextcloud-data
      restartPolicy: Always
      volumes:
        - name: nextcloud-data
          hostPath:
            path: "/k8s-data/cloud.dark.star/nextcloud"
            type: Directory
{% endhighlight %}

Things to note:

* We use the `app: nextcloud` and `tier: frontend` labels to distinguish this deployment and pod from the database backend. This is also reflected in the selectors of the deployment itself and the service (below).
* We set some variables that configure NextCloud. The `TZ`, `DEBUG` and `NEXTCLOUD_URL` parameters should be self-explanatory.
* The next two parameters define some PHP settings for the maximum size and number of concurrent file uploads for a single request.
* There are four parameters for the database connection. The `MYSQL_HOST` is the name of the service in the backend, which will make DNS-based service discovery work. The `MYSQL_PASSWORD` is again taken from the same secret where the backend database took it from.
* The `APACHE_DISABLE_REWRITE_IP` is necessary to disable IP rewriting, which doesn't work correctly behind Traefik and an nginx ingress. Disabling IP rewrite will make NextCloud check and use the actual `X-Forwarded-For`, `X-Forwarded-Proto`, etc. headers, which will be set by the ingress.
* The `TRUSTED_PROXIES` variable will set the network range from which NextCloud will accept the X-Forwarded headers. Since our pods will always be somewhere within the 10.0.0.0/8 subnet, this is what we choose here.
* Finally we have another volume which gets mapped to a path on our USB stick.

We apply this manifest to the cluster:

    pi@raspberrypi:~ $ kubectl apply -f cloud-nc-deployment.yaml
    deployment/nextcloud created

We also again need a service, which should be quite straightforward now:

{% highlight yaml %}
apiVersion: v1
kind: Service
metadata:
  name: nextcloud
  namespace: cloud
  labels:
    app: nextcloud
spec:
  ports:
    - protocol: TCP
      port: 80
  selector:
    app: nextcloud
    tier: frontend
{% endhighlight %}

Note that we select the destination pods via both the `app` and `tier` labels, so that we can distinguish the frontend from the backend pods.

Now all we need is an ingress. We will request a production certificate for it from LetsEncrypt right away this time, without going through staging first. Save this file as `cloud-ingress.yaml`:

{% highlight yaml %}
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: cloud-nginx-ingress
  namespace: cloud
  annotations:
    kubernetes.io/ingress.class: "traefik"
    cert-manager.io/cluster-issuer: letsencrypt-prod
spec:
  rules:
  - host: cloud.dark.star
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: nextcloud
            port:
              number: 80
  tls:
  - hosts:
    - cloud.dark.star
    secretName: cloud-dark-star-tls
{% endhighlight %}

Applying this to the cluster makes our NextCloud instance available through `https://cloud.dark.star` after a few seconds.

    pi@raspberrypi:~ $ kubectl apply -f cloud-ingress.yaml
    ingress/cloud-nginx-ingress created

## Fixes for the best-practice checks in NextCloud

Nextcloud has an integrated check for configuration issues and best practices, which can be accessed in the *Settings* menu under *Administration* -> *Overview*.

### Redirects for carddav and caldav

The first thing it will complain about is that some redirects are not set up correctly:

> Your web server is not properly set up to resolve "/.well-known/caldav". Further information can be found in the documentation
>
> Your web server is not properly set up to resolve "/.well-known/carddav". Further information can be found in the documentation

This is because, even though redirection is set up for these two well-known URLs, the redirection goes to the http endpoint instead of the https endpoint. The reason is that the redirection is done by the frontend apache webserver, and since the ingress already does TLS termination, the webserver receives all its traffic on port 80 (http), and so it will also generate redirects that go to port 80:

    pi@raspberrypi:~ $ curl -v https://cloud.dark.star/.well-known/caldav
    ...
    > GET /.well-known/caldav HTTP/2
    > Host: cloud.dark.star
    > User-Agent: curl/7.64.0
    > Accept: */*
    > 
    * TLSv1.3 (IN), TLS handshake, Newsession Ticket (4):
    * Connection state changed (MAX_CONCURRENT_STREAMS == 250)!
    < HTTP/2 301
    < content-type: text/html; charset=iso-8859-1
    < date: Sat, 20 Nov 2021 21:16:41 GMT
    < location: http://cloud.dark.star/remote.php/dav/
    ...

There are two ways to solve this. One is to move the redirect to the ingress (Traefik) itself, and the other is fixing the Webserver's redirect.

I could not get the first method to work, so we will be focusing on the second method.

Edit the file `.htaccess` in the root directory of your NextCloud volume. This is `/k8s-data/cloud.dark.star/nextcloud/.htaccess` in our case. Look for the following lines:

{% highlight apache %}
<IfModule mod_rewrite.c>
  RewriteEngine on
  RewriteCond %{HTTP_USER_AGENT} DavClnt
  RewriteRule ^$ /remote.php/webdav/ [L,R=302]
  RewriteRule .* - [env=HTTP_AUTHORIZATION:%{HTTP:Authorization}]
  RewriteRule ^\.well-known/carddav /remote.php/dav/ [R=301,L]
  RewriteRule ^\.well-known/caldav /remote.php/dav/ [R=301,L]
  RewriteRule ^remote/(.*) remote.php [QSA,L]
  RewriteRule ^(?:build|tests|config|lib|3rdparty|templates)/.* - [R=404,L]
  RewriteRule ^\.well-known/(?!acme-challenge|pki-validation) /index.php [QSA,L]
  RewriteRule ^(?:\.(?!well-known)|autotest|occ|issue|indie|db_|console).* - [R=404,L]
</IfModule>
{% endhighlight %}

Change the two lines in the middle to the following:

{% highlight apache %}
  RewriteRule ^\.well-known/carddav https://%{SERVER_NAME}/remote.php/dav/ [R=301,L]
  RewriteRule ^\.well-known/caldav https://%{SERVER_NAME}/remote.php/dav/ [R=301,L]
{% endhighlight %}

This will explicitly redirect to an https URL. Note that this means that you must use carddav/caldav clients that support https, which is probably not a problem these days.

If you have any hints on how to change the ingress so that the redirects will be done by Traefik directly, please get in touch...
{: .notice}

### SVG support is not available

> Module php-imagemagick in this instance has no SVG support. For better compatibility it is recommended to install it

This is actually a missing dependency in the official docker container for NextCloud. They do not install the corresponding modules because of [security concerns][nextcloud_imagemagick_security]. The proper way to install these modules is to rebuild the container image from the Dockerfile and add the dependency there.

A quick and dirty solution is provided here, which works until the pod is destroyed and re-created

    pi@raspberrypi:~ $ POD=$(kubectl get pods -n cloud -l "tier=frontend" -o json | jq -r '.items[0].metadata.name')
    pi@raspberrypi:~ $ kubectl exec -n cloud -it ${POD} -- /bin/sh
    # apt-get update
    ...
    # apt-get install libmagickcore-6.q16-6-extra
    ...
    # rm -rf /var/lib/apt/lists
    # exit

The additional module is picked up immediately. If you press F5 in the browser, you will see the warning is gone.

### Default phone number format

> Your installation has no default phone region set. This is required to validate phone numbers in the profile settings without a country code. To allow numbers without a country code, please add `default_phone_region` with the respective ISO 3166-1 code of the region to your config file.

This warning relates to a rather new option in the configuration file that has not been picked up by the Docker container builders yet, so there is no option to set this when starting the container.

To manually change it, edit the main config file for NextCloud, which in our case is `/k8s-data/cloud.dark.star/nextcloud/config/config.php` and add the following parameter somewhere in the file (preferably at the end):

    'default_phone_region' => 'DE',

Obviously, change the country code to match your users' main country. And make sure you do not mess up the list, there should be a comma after every line, even after the last one. This is what the end of the file should look like if you live in Germany:

    ...
      'dbpassword' => 'user1password2cloud',
      'installed' => true,
      'default_phone_region' => 'DE',
    );

[nextcloud_imagemagick_security]: https://docs.nextcloud.com/server/17/admin_manual/configuration_files/previews_configuration.html

This is it, now you can set up NextCloud as you want.

Note that there is still the issue that the Collabora CODE server could not be installed during the initial setup. It is by default only available for x86_64, although there are ARM and ARM64 versions of it around. Keep watching this space as there will probably be a guide on how to install it later, as soon as I get around to fiddling with it.
{: .notice}

[^1]: Well, to be honest, I didn't know about this method when I wrote part 4, so after I found out that MariaDB can automatically create a database for us, I used this simpler method here...

## Outlook

Now that we have deployed three apps, we will wrap it up for now. This guide is designed to grow with time, and some things I would like to add are mentioned here. Feel free to suggest any topics you would like to see covered.

* Persistent Volumes (longhorn or nfs-subdir-external-provisioner)
* Scaling our cluster to multiple nodes (required for longhorn for example)
* Using helm to deploy workloads
* Path-Based Routing and advanced middleware with Traefik2
* Dashboards / Frontends for k8s
* Monitoring

