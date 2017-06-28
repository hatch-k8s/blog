---
layout: post
title: How to pull images from a private repository with kubernetes
date: 2017-06-28 10:10:10 -0200
categories: registry
---

#How to pull images from a private repository with kubernetes


In order to be able to pull images from a private regristry in kubernetes, we need to provide
some kind of authentication. Here I am not talking about managed environments (cloud) but bare
metal installations or fully managed clusters. 

As kubernetes docs explain there are three ways to instruct your kubelet how to authenticate
with a private registry to pull the images.

- Add the docker configuration file with the token in all nodes of your cluster
First, to obtain the configuration file we have to login to our private register. To do that
execute
```
docker login my-private-registry.com
```
You have to provide the user and password (if you dont have a different auth mechanism). It will 
generate a file on your home director `$HOME/.docker/config.json`, in case your docker is old version
(below 1.7) then `$HOME/.docker/config.json`.

This config file should be placed on the root home directory `/root/.docker/config.json` of every 
node that would be part of the cluster. Now docker can pull from the registry in every node
when kubelet is creating a pod and the image is not in the cache.


- Specify a secret in your namespace with the docker config token and use it on the pods yamls 
Use imagePullSecrets is the newest way to achieve and personally the one I like most. You can
use the same approach you use for keeping other secrets and specify the secret that contains the
config directly in the resource definition file (also there is an alternative adding a service
account with the docker config token attached so all pods will inherit from it).

First we must to generate a secret that contains our docker configuration token. To do that we can 
use a kubectl to help us
```
kubectl create secret docker-registry registry_key --docker-server=my-private-registry.com 
--docker-username=my-docker-user --docker-password=my-docker-password --docker-email=my-docker-email
```
This command generates a secret with a configuration like
```
apiVersion: v1
kind: Secret
metadata:
  name: registry_key
  namespace: default
data:
  .dockerconfigjson: <<base64 docker config json auth key>>
type: kubernetes.io/dockerconfigjson
```
So if you have multiple registries and your home docker config.json keep information for all,
you can select the auth key for your own and generate a base64 hash with it and
paste it in a resource secret yaml and use kubectl create.


- Not really pull from repository only take the cached images 
Even it sounds a but ilogical, in terms of security could be a nice approach. This way our
nodes have not to access server (registry) to download the images. The disavantage is
we have to ensure all nodes have all images with the correct label. As you can imagine the 
problems come when you update or add a new component in your cluster. The image should be
downloaded first to give order to kubernetes to update the resource. 

In order to tell kubernetes not try to access any image repository we should add the property
imagePullPolicy with the value 'Never'.

{% if page.comments %}
<div id="disqus_thread"></div>
<script>

/**
*  RECOMMENDED CONFIGURATION VARIABLES: EDIT AND UNCOMMENT THE SECTION BELOW TO INSERT DYNAMIC VALUES FROM YOUR PLATFORM OR CMS.
*  LEARN WHY DEFINING THESE VARIABLES IS IMPORTANT: https://disqus.com/admin/universalcode/#configuration-variables*/
/*
var disqus_config = function () {
this.page.url = https://hatch-k8s.github.io/hatch-k8s;  // Replace PAGE_URL with your page's canonical URL variable
this.page.identifier = 'private-registry'; // Replace PAGE_IDENTIFIER with your page's unique identifier variable
};
*/
(function() { // DON'T EDIT BELOW THIS LINE
var d = document, s = d.createElement('script');
s.src = 'https://hatch-k8s.disqus.com/embed.js';
s.setAttribute('data-timestamp', +new Date());
(d.head || d.body).appendChild(s);
})();
</script>
<noscript>Please enable JavaScript to view the <a href="https://disqus.com/?ref_noscript">comments powered by Disqus.</a></noscript>
{% endif %}
