---
layout: post
title: "Terraform + Google Kubernetes Engine"
tags: gitlab gitlab-runner kubernetes
categories: devops
---

This is the first part of a two-part series where we look at configuring our own Gitlab Runner on GKE for use within any Gitlab project(s) you maintain. 

# Introduction
In this guide we will provision a GKE cluster for use with Gitlab Runners using Terraform. 

# Prerequistes
It is assumed that you have a Google Cloud Platform project and an account with the proper permissions
to provision GKE cluster and creating service accounts. This guide will not be a tutorial on
Kubernetes or GCP. If you're not familiar with either, I recommend reading or use as reference the following
links 

https://kubernetes.io/docs/home/

https://cloud.google.com/kubernetes-engine/


# Defining the cluster requirements
Before creating any cluster we need to think about what our requirements for the cluster is. 

- __Autoscaling nodes:__ we want nodes to scale up and down depending on the workload amount
- __Cache bucket:__ Gitlab Runner supports caching of build dependencies so we want to initialize a bucket on Google Cloud

# Configure GCP network 
Before we start with the Terraform configuration we need to configure the network in our project on GCP. 

# Cluster configuration
With Terraform we can configure the cluster in every last detail, which might require more than a few hours to configure correctly. Fortunately there exists a Terraform module for GKE that will take care of the details for us.

First create a directory for your Terraform configuration. Inside create the cluster configuration fie `cluster.tf`. This is where define what we want our cluster to look like. Using 

{% highlight terraform%}
module "gke" {
  source                     = "terraform-google-modules/kubernetes-engine/google"
  project_id                 = "<PROJECT ID>"
  name                       = "ci-cluster"
  region                     = "us-central1"
  zones                      = ["us-central1-a"]
  network                    = "ci-network"
  subnetwork                 = "ci-subnetwork"

  ip_range_pods              = "pods-ip-range"
  ip_range_services          = "services-ip-range"
  http_load_balancing        = false
  horizontal_pod_autoscaling = true
  network_policy             = true
 
  ...

}

{% endhighight %}


