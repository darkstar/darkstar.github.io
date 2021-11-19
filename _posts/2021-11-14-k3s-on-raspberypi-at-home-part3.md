---
layout: post
title:  "k3s on a Raspberry Pi 4 at home, Part 3"
date:   2021-11-14 12:00:00 +0100
tags:   raspberrypi k3s
---

This is part three of our small series of posts, where we deploy the first static website to our cluster, complete with TLS certificates.

## Scenario 1: Creating a static website

Now we can finally start creating our first goal, the static website. To deploy this, we need at least some simple HTML, saved in a file called `index.html`:

{% highlight html %}
<HTML>
<BODY>
<H2>Hello World from k3s!</H2>
<P>It works!</P>
</BODY>
</HTML>
{% endhighlight %}

Now is also a good time to set up a directory structure on the USB stick (if you use one), so, assuming you mounted the USB stick as `/k8s-data`, copy this HTML file into a new directory `/k8s-data/web.dark.star/` on your Raspberry Pi.

Create a namespace for the static website. Let's call it `web`:

    pi@raspberrypi:~ $ kubectl create namespace web
    namespace/web created

Now we build a Deployment which describes our webserver. This makes it easy to later scale up the Website, should external traffic require additional CPU performance. Also, Kubernetes will automatically restart the web server when it notices that the number of desired replicas in the deployment (initially 1) is not met.

Save this file as `web-deployment.yaml`:

{% highlight yaml %}
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-nginx
  namespace: web
  labels:
    app: web-nginx
spec:
  replicas: 1
  selector:
    matchLabels:
      app: web-nginx
  template:
    metadata:
      labels:
        app: web-nginx
    spec:
      containers:
      - name: nginx
        image: nginx
        ports:
        - containerPort: 80
        volumeMounts:
        - mountPath: /usr/share/nginx/html
          name: html-volume
          readOnly: true
      volumes:
      - name: html-volume
        hostPath:
          path: /k8s-data/web.dark.star
          type: Directory
{% endhighlight %}

This deployment will be called `web-nginx` and it gets an additional label, the key-value pair `app: web-nginx`. This is to make it easier to later determine which objects belong to the same app in the cluster.
The spec defines the number of pod replicas (1 in our case), and a selector which defines what pods to count into that number (all pods with the `app: web-nginx` label).
It also defines the template for the pods, which will have the same label, `app: web-nginx` as the deployment. The rest of the pod definition is pretty straightforward: use the `nginx` container from DockerHub, expose port 80, and mount a volume called `html-volume` read-only into the nginx root path.

The volume itself is defined in the last section, and here we just use the `hostPath` Syntax to specify an (existing) directory on the Kubernetes worker node. This will, obviously, work only in simple 1-node clusters and we would need something better if we were to scale up our cluster to 2 or more nodes.

Note that we specifically don't give the pod a name in the template, as the name of the pod cannot be a fixed string (what if 2 or more replicas should be launched? There would be multiple objects with the same name in the cluster). Kubernetes automatically assigns the pods a name based on the name of the deployment and some random, unique string.

Apply this manifest into your cluster:

    pi@raspberrypi:~ $ kubectl apply -f web-deployment.yaml
    deployment/web-nginx created

Now wait for the pods to pull the container from DockerHub and start up:

    pi@raspberrypi:~ $ kubectl get deployment -n web
    NAME        READY   UP-TO-DATE   AVAILABLE   AGE
    web-nginx   1/1     1            1           8m
    
    pi@raspberrypi:~ $ kubectl get pods -n web
    NAME                         READY   STATUS    RESTARTS   AGE
    web-nginx-85855d8965-7fqks   1/1     Running   0          8m

You can see the internal IP address of your pod by adding `-o wide` to the last command. Then you can try to `curl` it:

    pi@raspberrypi:~ $ kubectl -n web get pod/web-nginx-85855d8965-7fqks -o wide
    NAME                        READY  STATUS   RESTARTS  AGE  IP          NODE         NOMINATED NODE  READINESS GATES
    web-nginx-85855d8965-7fqks  1/1    Running  0         8d   10.42.0.52  raspberrypi  <none>          <none>
    
    pi@raspberrypi:~ $ curl http://10.42.0.52
    <HTML>
    <BODY>
    <H2>Hello World from k3s!</H2>
    <P>It works!</P>
    </BODY>
    </HTML>

Note that this IP address is internal and changes everytime your pod gets rescheduled. So it is neither stable/reliable to access from other pods, nor is it possible to make it available externally.

To try that, and to also see that kubernetes intelligently re-schedules crashed pods, try deleting the nginx pod:

    pi@raspberrypi:~ $ kubectl delete -n web pod/web-nginx-85855d8965-7fqks
    pod "web-nginx-85855d8965-7fqks" deleted
    
    pi@raspberrypi:~ $ kubectl get pods -n web
    NAME                         READY   STATUS              RESTARTS   AGE
    web-nginx-85855d8965-lk9ms   0/1     ContainerCreating   0          12s
    
    pi@raspberrypi:~ $ kubectl get pods -n web -o wide
    NAME                        READY  STATUS   RESTARTS  AGE  IP           NODE         NOMINATED NODE  READINESS GATES
    web-nginx-85855d8965-lk9ms  1/1    Running  0         23s  10.42.0.176  raspberrypi  <none>          <none>

The deployment has done its job and ensured that exactly 1 pod is always running. And we see that the new pod gets a different IP address. It also has a different name, because the name of the pod is randomly generated by the deployment.

To give the webserver a fixed IP within the cluster, we create a service. Save this as `web-service.yaml`:

{% highlight yaml %}
apiVersion: v1
kind: Service
metadata:
  namespace: web
  name: web-nginx-service
spec:
  selector:
    app: web-nginx
  ports:
    - protocol: TCP
      port: 80
{% endhighlight %}

The service will have a name, and it selects the pods that the network connections should be forwarded to, by selecting them via the label `app: web-nginx`, which we defined in our deployment above. The IP address discovery is completely dynamic. If we have multiple pods running with the `app: web-nginx` label, a random one is chosen for every request[^1]. Note that the service only defines port 80, HTTP, even though we will later be able to access it via HTTPS: The full HTTPS session will be handled by the ingress (or rather the Traefik network controller), which will terminate the client's TLS session, forward the traffic via HTTP within the cluster, and re-encrypt the web server's reply on egress. This way, no pod or service needs to be explicitly configured for TLS, as that is all taken care of by the Kubernetes cluster network transparently.

Now apply the manifest to your cluster:

    pi@raspberrypi:~ $ kubectl apply -f web-service.yaml
    service/web-nginx-service created

You can inspect the created service object to get the stable (within this cluster, and as long as you don't delete and re-create the service) IP address:

    pi@raspberrypi:~ $ kubectl -n web get service
    NAME                TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)   AGE
    web-nginx-service   ClusterIP   10.43.138.190   <none>        80/TCP    8d

You will find that you can `curl http://10.43.138.190` and also get back your HTML page. This works even if you delete your pod and let the deployment re-create it: the service's IP address won't change.

But that still doesn't mean we can access our pod from the outside world. For that we will need an ingress. The ingress defines the HTTP host and URL path where our server will be accessible.

Create a file called `web-ingress.yaml` with the following content:

{% highlight yaml %}
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: web-nginx-ingress
  namespace: web
  annotations:
    kubernetes.io/ingress.class: "traefik"
spec:
  rules:
  - host: web.dark.star
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: web-nginx-service
            port:
              number: 80
{% endhighlight %}

The ingress defines rules, which the Traefic network controller uses to determine the correct ingress/service to route incoming traffic to. In this case, when the host is `web.dark.star`, it routes all request URLs 1:1 to a backend which is described by the service `web-nginx-service`, port 80. With the `path:` parameter you could have multiple services reply on the same external hostname, under different "sub-URLs". This is called path-based routing. We might look at that in a later part, until then check the Traefik documentation for more details.

Apply the file to the cluster and check its status:

    pi@raspberrypi:~ $ kubectl apply -f web-ingress.yaml
    ingress/wen-nginx-ingress created
    
    pi@raspberrypi:~ $ kubectl get ingress -n web
    NAME                CLASS    HOSTS           ADDRESS       PORTS   AGE
    web-nginx-ingress   <none>   web.dark.star   192.168.1.3   80      8d

That's it. This should make our website accessible from the outside, provided that port forwarding on your router works correctly, and that the DNS updates have taken place and propagated through the internet:

    pi@raspberrypi:~ $ curl http://web.dark.star
    <HTML>
    <BODY>
    <H2>Hello World from k3s!</H2>
    <P>It works!</P>
    </BODY>
    </HTML>

This works only for HTTP so far, not HTTPS, but we will fix that in a minute.

Note however, that trying to access the IP of your router directly will not yield that website:

    pi@raspberrypi:~ $ curl -s icanhazip.com
    198.51.100.17
    pi@raspberrypi:~ $ curl http://198.51.100.17
    404 page not found

This is because the Traefik needs a hostname so that it can route the request to the correct ingress->service->pod, but without a hostname it doesn't know what to do. So it instead routes the traffic to a _default backend_, which, by default, doesn't exist, and so Traefik returns a 404 error to the client.

To enable TLS for HTTPS, we need to change the YAML file slightly. Edit the `web-ingress.yaml` file and make the following changes:

{% highlight yaml %}
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: web-nginx-ingress
  namespace: web
  annotations:
    kubernetes.io/ingress.class: "traefik"
    # insert the following line here:
    cert-manager.io/cluster-issuer: letsencrypt-staging
spec:
  rules:
  - host: web.dark.star
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: web-nginx-service
            port:
              number: 80
  # insert these 4 lines here. Watch the indentation, the "tls:" should be on the
  # same level as the "rules:" above
  tls:
  - hosts:
    - web.dark.star
    secretName: web-dark-star-tls
{% endhighlight %}

To update the ingress in your cluster, simply apply it again:

    pi@raspberrypi:~ $ kubectl apply -f web-ingress.yaml
    ingress/web-nginx-ingress changed

You should now see a certificate being created, and after a few seconds it should be ready:

    pi@raspberrypi:~ $ kubectl get certificates -n web
    NAME                  READY   SECRET                AGE
    web-darkstar-me-tls   True    web-darkstar-me-tls   54s

Try accessing your site through https, this should now work.

    pi@raspberrypi:~ $ curl -k https://web.dark.star
    <HTML>
    <BODY>
    <H2>Hello World from k3s!</H2>
    <P>It works!</P>
    </BODY>
    </HTML>

Note that you still only get an invalid staging certificate, that's why we had to use the `-k` switch in the example above. You should _always_ test everything in TLS with a staging certificate first. If it works, you can easily switch to the production certificate.

To switch to a real certificate, edit the YAML manifest and change the cluster-issuer in the annotations from `letsencrypt-staging` to `letsencrypt-prod`. Then apply the file again, or use `kubectl edit -n web ingress/web-nginx-ingress` to make the same change in the live cluster as well.

As soon as you either apply or edit the ingress, cert-manager will request a new certificate and install it. After a few seconds you should have a valid certificate. Note that in browsers this might require you to hard-reload by using Ctrl+F5, to flush the old certificate from the browser cache.

[^1]: Not completely random, the network operator tries to route requests from the same IP to the same pod for a little while, so that session data etc. can be preserved. If you want to test this, scale your replica to 2 or more, change your index.html page to an index.php page which includes some PHP code to print the server hostname (`<?php echo gethostname(); ?>`, and press F5 a couple of times.

This is it for now, after deploying a static website, we will take a look at deploying a more complex, 2-tier application in the next part. Stay tuned!

