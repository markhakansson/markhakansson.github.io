---
layout: post
title: "GKE and IP masquerade"
tags: gitlab gitlab-runner kubernetes gke terraform
categories: devops
---

# Network configuration

{% highlight terraform%}
// cluster.tf
module "gke" {

  ...

  network                    = "ci-network"
  subnetwork                 = "ci-subnetwork"
  ip_range_pods              = "pods-ip-range"
  ip_range_services          = "services-ip-range"
  http_load_balancing        = false
  network_policy             = true
  
  ...

}
{% endhighlight%}