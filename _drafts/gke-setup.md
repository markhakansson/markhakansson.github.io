---
layout: post
title: "Gitlab Runner + Google Kubernetes Engine: Provisioning with Terraform"
tags: gitlab gitlab-runner kubernetes gke terraform
categories: devops
---

This is the first part of a two-part series where we look at configuring our own Gitlab runner on GKE for use with any Gitlab project(s) you maintain. 


In this guide we will provision a GKE cluster for use with our Gitlab Runner using the infrastructure provisioning tool [Terraform](https://www.terraform.io/). 

# Prerequisites
It is assumed that you have a Google Cloud Platform project and an account with the proper permissions
to provision GKE cluster and creating service accounts. This guide will not be a tutorial on
Kubernetes or GCP. If you're not familiar with either, I recommend reading or use as reference the following
links:

https://kubernetes.io/docs/home/

https://cloud.google.com/kubernetes-engine/

# Introduction
Whether you're hosting your git repositories on Gitlab or on your own self-hosted Gitlab instance, you might have heard about Gitlab pipelines. With a single configuration file you can specify each the steps of the CI/CD process for your project as jobs. Where a job could be building the project or perhaps executing the unit tests. When a pipeline is run, it is sent to a Gitlab Runner where it follows the instructions given in the configuration file. Then for each job, assigns a "machine" to handle that job. 

If you've used the public Gitlab runners on Gitlab.com, you might've encountered a few performance bottlenecks on larger projects. To speed up our pipelines we could always host a Gitlab runner on our own or rented server. But what if we sometimes need to run many pipelines simultaneously. More than what our server can handle? 

Well, we could get better hardware for our server to be able handle that load at all times. But then what if we only run pipelines a few times a day? In that case the server will stay under-utilized for most of the day! And we will be paying for resources we are not using! Fortunately, the Gitlab runner has support for running in Kubernetes (K8s) clusters. If we pair this with the Google Kubernetes Engine (GKE) we can utilize cloud features such as autoscaling to get the resources required, only when we need it!

But first we need to instantiate a K8s cluster in GKE, where we will have our Gitlab runner running! In this guide we will provision a cluster on GKE using the handy provisioning tool Terraform.

# Defining the cluster requirements
Before creating a cluster we need to think about what our requirements are. Our main goal is to have a cost-efficient solution. Which lends us the following requirements

- __Autoscaling nodes:__ we want nodes to scale up and down depending on the workload. 
- __Cache bucket:__ Gitlab Runner supports caching of build dependencies so we want to initialize a bucket on Google Cloud to store them at.

# General configuration
With Terraform you configure/provision the infrastructure using [Terraform providers](https://www.terraform.io/docs/providers/index.html). They allow extensive customization and many options, but which might require more than a few hours to configure correctly. Fortunately there exists a Terraform module for GKE that will take care of the unnecessary details for us. Module's are abstractions created from the Terraform providers as base. 

First create a directory for your Terraform configuration. Inside create the cluster configuration fie `cluster.tf`. This is where define what we want our cluster to look like. We create a new module using the GKE module as source

{% highlight terraform%}
// cluster.tf
module "gke" {
  source                     = "terraform-google-modules/kubernetes-engine/google"
  project_id                 = "<PROJECT_ID>"
  name                       = "ci-cluster"
  region                     = "us-central1"
  zones                      = ["us-central1-a"]
 
  ... 

}
{% endhighlight%}
The GKE module can take various variables as input. The full list of acceptable inputs can be viewed [here](https://www.terraform.io/docs/providers/index.html).

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

# Node pool configuration
Next we configure our node pools. 
{% highlight terraform%}
module "gke" {

  ... 

  node_pools = [
    {
      name               = "manager-node-pool"
      machine_type       = "e2-standard-2"
      min_count          = 1
      max_count          = 1
      disk_size_gb       = 50 
      auto_repair        = true
      auto_upgrade       = true
      service_account    = "project-service-account@<PROJECT ID>.iam.gserviceaccount.com"
      preemptible        = false
    },
    {
      name               = "worker-node-pool"
      machine_type       = "e2-standard-16"
      autoscaling        = true
      min_count          = 1
      max_count          = 50
      disk_size_gb       = 50 
      disk_type          = "pd-standard"
      image_type         = "COS"
      auto_repair        = true
      auto_upgrade       = true
      service_account    = "project-service-account@<PROJECT ID>.iam.gserviceaccount.com"
      preemptible        = false
      initial_node_count = 80
    },
  ] 
  ...
  
}
{% endhighlight%}
Here we create two node pools. The first is the manager pool, which only contains a single node where the Gitlab Runner pod will be running in. The second pool is where we will have our worker pods. That is the pods that will be executing and running the pipeline jobs.   

# Bucket configuration
