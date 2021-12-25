---
title: "Eclipse/steady — Support components"
date: 2019-10-30T11:40:00+00:00
draft: true
images:
tags:
  - untagged
---

The previous components need to be served on the Internet at given set of endpoints. Initially, this task was performed by HAProxy, a high availability load balancer for high traffic websites. However, due to its lack of caching functionality, the current implementation tunnels certain connection through an NGINX deployment serving the content and caching based on a parameters passed through the query (a bool indicating whether to cache and another one indicating its validity). This redundance of web servers lead to higher overhead cost and the cache logic being relegated to the application. 

It its also important to note that in a k8s context, the internal IPs of the machines hosting the web servers are dynamic. As such, the traditional static routing configuration often seen in web servers are inapplicable. **Ingresses** are k8s native objects meant to address these issues. Backed by Ingress Controllers, HTTP and HTTPS routes from outside the cluster can be redirected to internal services according to a set of defined rules. 

The [NGINX Ingress Controller](https://github.com/kubernetes/ingress-nginx) has been chosen as the entrypoint to the system to tackle the aforementioned problems with caching and dynamic configuration (see Figure (\ref{fig:vulas_admin})). By utilizing NGINX's feature for caching content, the `X-Accel-Expires` Header can be changed in the application's AJAX query to control both whether to cache the content and its expiry date. For persisting the cache between different replicas of the ingress controller, each pod of the devised statefulset mounts the same NFS on the same cache path. Other solutions such as using a Redis or Memcached cluster as a cache storage are eliminated for its high overhead cost. 

{{< image src="/img/vulas_admin.png"  position="center" >}}

In order to serve the contents through a secure connection, an SSL certificate should be added to the nginx-ingress-controller through a config map. This operation is manual and could be automated by using [cert-manager](https://github.com/jetstack/cert-manager) to manage and certificate cycling but can't be used due to high overhead costs.