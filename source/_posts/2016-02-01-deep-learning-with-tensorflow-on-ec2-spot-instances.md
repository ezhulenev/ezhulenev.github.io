---
layout: post
title: "Large Scale Deep Learning with TensorFlow on EC2 Spot Instances"
date: 2016-02-01 11:14:16 -0500
comments: true
categories: [clojure, machine learning, deep learning, tensorflow, aws, ecs, ec2]
keywords: clojure, machine learning, deep learning, tensorflow, aws, ecs, ec2
---

In this post I'm demonstrating how to combine together **TensorFlow**, **Docker**, **EC2 Container Service** and **EC2 Spot instances** to solve massive
cluster computing problems the most cost-effective way.

> Source code is on on Github: [https://github.com/ezhulenev/distributo](https://github.com/ezhulenev/distributo)

Neural Networks and Deep Learning in particular gained a lot of attention over the last year, and it's only the beginning. Google released
to open source their numerical computing framework [TensorFlow](https://www.tensorflow.org/), which can be used for training and running
deep neural networks for wide variety of machine learning problems, especially image recognition.

> TensorFlow was originally developed by researchers and engineers working on the Google Brain Team within Google's Machine Intelligence research 
> organization for the purposes of conducting machine learning and deep neural networks research, but the system is general enough to be 
> applicable in a wide variety of other domains as well.

Although TensorFlow version used at Google supports distributed training, open sourced version can run only on one node. However some of machine learning
problems are still embarrassingly parallel, and can be easily parallelized regardless of single-node nature of the core library itself
  
1. **Hyperparameter optimization** or **model selection** is the problem of choosing a set of hyperparameters for a learning algorithm, 
usually with the goal of optimizing a measure of the algorithm's performance on an independent data set.
2. **Inference** - splitting input dataset and applying trained model to smaller batches in parallel


## Hyperparameter Optimization

TensorFlow provides primitives for expressing and solving various numerical computations using data flow graphs, and it was primarily designed to solve
neural networks research problems.

Choosing a right design and parameters for your neural network is separate optimization problem:  

* how many layers to use
* how many neurons in each layer
* what learning rate to use

Luckily all of these choices are independent and can be easily run in parallel.

{%img center /images/deep-learning/hyperparameter-optimization.png Hyperparameter Optimization %}

## Inference

When you already have trained model, and you want to score/classify huge dataset, you can use similar approach: split all your input
data into smaller batches, and run them in parallel.

# TensorFlow for Image Recognition

## Packaging into Docker Image

TensorFlow has awesome [Image Recognition Tutorial](https://www.tensorflow.org/versions/0.6.0/tutorials/image_recognition/index.html), which uses already
pre-trained ImageNet model for image recognition/classification. You provide an image, and a model gives you what it 
can see on this image: leopard, container ship, place, etc.. Works like magic.

I've prepared [Docker image](https://github.com/ezhulenev/distributo/tree/master/example/docker-tensorflow) based on official TensorFlow image 
and slightly modified image classification example. It takes a range of images that needs to be classified, and S3 
path where to put results:

{% coderay lang:bash %}
# You have to provider your AWS credentials to upload files on S3
# and S3 bucket name
docker run -it -e 'AWS_ACCESS_KEY_ID=...' -e 'AWS_SECRET_ACCESS_KEY=...' \
        ezhulenev/distributo-tensorflow-example \
        0:100 s3://distributo-example/imagenet/inferred-0-100.txt
{% endcoderay %}

This command will classify first 100 images from [http://image-net.org/imagenet_data/urls/imagenet_fall11_urls.tgz](http://image-net.org/imagenet_data/urls/imagenet_fall11_urls.tgz):

{% coderay %}
n00005787_175   http://farm3.static.flickr.com/2179/2090355369_4c8a60f899.jpg
n00005787_185   http://farm4.static.flickr.com/3205/2815917575_c2ea596ed2.jpg
n00005787_186   http://farm3.static.flickr.com/2084/2517885309_6680a79ab1.jpg
n00005787_190   http://farm1.static.flickr.com/81/245539781_42028c8c67.jpg
n00005787_198   http://farm2.static.flickr.com/1437/680424989_da45c42286.jpg
n00005787_219   http://farm1.static.flickr.com/176/441681804_fec8ae4c58.jpg
{% endcoderay %}

And upload inference results to S3:

{% coderay %}
n00005787_175,http://farm3.static.flickr.com/2179/2090355369_4c8a60f899.jpg,[
  ('brain coral', 0.354022), 
  ('hen-of-the-woods, hen of the woods, Polyporus frondosus, Grifola frondosa', 0.18448937), 
  ('coral reef', 0.15611894), 
  ('gyromitra', 0.035839655), 
  ('coral fungus', 0.03291795)
]
n00005787_185,http://farm4.static.flickr.com/3205/2815917575_c2ea596ed2.jpg,[
  ('sea slug, nudibranch', 0.4032793), 
  ('sea cucumber, holothurian', 0.17277676), 
  ('hermit crab', 0.043269496),
  ('conch', 0.036443222), 
  ('jellyfish', 0.023511186)
]
{% endcoderay %}

## Moving into the AWS Cloud

Amazon has [EC2 Container Service (ECS)](https://aws.amazon.com/ecs/), which is container management service that supports Docker containers,
and allows to easily launch any task packaged into container on EC2 instances using simple API. You don't have to worry about managing
your cluster or installing any additional software. It just works out of the box with Amazon provided AMIs.

ECS clusters are running on regular EC2 instances, and it's up to you what instances to use. One option, that especially makes sense
for large offline model training/inference it to use [EC2 Spot Instances](https://aws.amazon.com/ec2/spot/) which allow you to bid on spare EC2 
computing capacity. Spot instances are usually available at a big discount compared to On-Demand pricing, this allows to 
significantly reduce the cost of running computation, and scale only when price allows to do so.

# Distributo

Distributo is a small library that makes it easier to automate EC2 resource allocation on spot market, 
and provides custom ECS scheduler that takes care of efficient execution of your tasks on available computing resources.

Source code is on Github: [https://github.com/ezhulenev/distributo](https://github.com/ezhulenev/distributo). 

It requires [Leiningen](http://leiningen.org/) to compile and to run example application.

### Resource Allocator

Resource allocator is responsible for allocating compute resources in EC2 based in outstanding 
jobs resource requirements. Right now it's only dummy implementation that can support fixed
size ECS cluster built from spot instances. You need to define upfront how many instances do you need.

### Scheduler

Scheduler decides on what available container instance to start pending jobs. It's using bin-packing 
with fitness calculators (concept borrowed from [Netflix/Fenzo](https://github.com/Netflix/Fenzo)) to 
choose best instance to start new task. It's the main difference from default ECS scheduler that
places tasks on random instances.

## Run TensorFlow Image Recognition with Distributo

Distributo uses AWS JAVA SDK to access your AWS credentials. If you don't have them already configured you
can do it with AWS CLI

{% coderay %}
aws configure
{% endcoderay %}
    
After that you can start you cluster and run TensorFlow inference with this command:    

{% coderay %}
lein run --inference \
  --num-instances 1 \
  --batch-size 100 \
  --num-batches 10 \
  --output s3://distributo-example/imagenet/
{% endcoderay %}

This command will run 10 TensorFlow containers with batches from `[0:100]` up to `[900:1000]` on single instance and 
put inference results into S3 bucket. By default it's buying `m4.large` instances for `$0.03` which can run only 2 containers 
in parallel, in this example 10 jobs will be competing for 1 instance.

Distributo doesn't free resources after it's done with inference. If you are done, don't forget to clean resources:

{% coderay %}
lein run --free-resources
{% endcoderay %}
        
## Future Work

Resource allocator and scheduler could be much more clever about their choices of regions, availability zones 
and instance types to be able to build most price-effective cluster out of resources currently 
available on spot market.

[TwoSigma/Cook](https://github.com/twosigma/Cook) - has lot's of great ideas about fair resource allocation and cluster 
sharing for large scale batch computations, which might be very interesting to implement

## Alternative Approaches

### Spark as Distributed Compute Engine

Apache Spark has Python integration and it's possible to achieve very similar parallelization 
with it: [https://databricks.com/blog/2016/01/25/deep-learning-with-spark-and-tensorflow.html](https://databricks.com/blog/2016/01/25/deep-learning-with-spark-and-tensorflow.html).

However it's completely different from approach that I described, because it doesn't allow to use Docker (easily) and requires non trivial cluster setup. Although it might be more powerful
because it's much easier to build more complicated pipelines.

### AWS Auto Scaling

ECS provides auto scaling out of the box, also it has it's own task scheduler. However task scheduler use random 
containers to place new tasks, which can lead to unefficient resource utilization. And with custom resource allocator it's 
possible to build more sophisticated strategies for buying cheapest computing resources on fluctuating market.

### Mesos

Mesos also provides great API for running tasks packaged as Docker images in the cluster.

However managing Mesos deployment is not trivial task, and you might not want to do have this headache. Also it's much more difficult to provide
truly scalable platform, you'll have to provision your cluster for peak load, which can be expensive and not cost-effective.
