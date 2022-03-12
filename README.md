# Load testing EKS cluster with Locust

This repository contains example code for creating EKS clusters and installing necessary addons such as [Cluster Autoscaler](https://github.com/kubernetes/autoscaler/tree/master/cluster-autoscaler) and [AWS Load Balancer Controller](https://github.com/aws/eks-charts/tree/master/stable/aws-load-balancer-controller) and [Locust](https://github.com/deliveryhero/helm-charts/tree/master/stable/locust).

In addition, it provides sample application to be used for testing. Please feel free to replace it with your own application.

For full details about using Locust, please see the [Locust official documentation](http://docs.locust.io/en/stable/).

## Table of content

- [Load testing EKS cluster with Locust](#load-testing-eks-cluster-with-locust)
  - [Table of content](#table-of-content)
  - [Introduction](#introduction)
  - [Overview of solution](#overview-of-solution)
  - [Groundwork](#groundwork)
    - [Prerequisites](#prerequisites)
    - [Provisioning EKS Clusters](#provisioning-eks-clusters)
    - [Installing Basic Addon Charts](#installing-basic-addon-charts)
    - [Installing a Sample Application Helm Chart](#installing-a-sample-application-helm-chart)
  - [Walkthrough](#walkthrough)
    - [STEP 1. Install Locust](#step-1-install-locust)
      - [_Switch kubernetes context to run commands on the locust cluster:_](#switch-kubernetes-context-to-run-commands-on-the-locust-cluster)
      - [_Add Delivery Hero public chart repo:_](#add-delivery-hero-public-chart-repo)
      - [_Write a `locustfile.py` file:_](#write-a-locustfilepy-file)
      - [Install and Configure Locust with locustfile.py](#install-and-configure-locust-with-locustfilepy)
    - [STEP 2. Expose Locust via ALB](#step-2-expose-locust-via-alb)
    - [STEP 3. Checkout Locust Dashboard](#step-3-checkout-locust-dashboard)
      - [_Open the URL from a browser:_](#open-the-url-from-a-browser)
    - [STEP 4. Run Test](#step-4-run-test)
    - [STEP 5. Run Test (2nd)](#step-5-run-test-2nd)
  - [Clean up](#clean-up)
  - [Summary](#summary)
  - [Security](#security)
  - [License](#license)

## Introduction

More and more customers are using Elastic Kubernetes Service (EKS) to run their workload on AWS. This is why it is essential to have a process to test your EKS cluster so that you can identify weaknesses upfront and optimize your cluster before you open it to public. Load test focuses on the performance and reliability of the cluster and it is especially important for those expect high elasticity from EKS. [Locust](https://locust.io/) is an open source load testing tools that comes with a real-time dashboard and programmable test scenarios.

In this post, I walk you through the steps to build two Amazon EKS clusters, one for generating loads using [Locust](https://locust.io/) and another for running a sample workload. The concepts in this post are applicable to any situation where you want to test the performance and reliability of your Amazon [EKS cluster](https://aws.amazon.com/eks).

You can find all of the code and resources used throughout this post in [the associated GitHub repository](https://github.com/aws-samples/load-testing-eks-cluster-with-locust).

## Overview of solution

The following diagram illustrates the architecture we use across this post. Your application is running as a group of pods in an EKS cluster (target cluster) and exposed via a public load balancer. Locust, in the meantime, is running in a separate EKS cluster (locust cluster). [Cluster Autoscaler](https://docs.aws.amazon.com/eks/latest/userguide/cluster-autoscaler.html) and [Horizontal Pod Autoscaler](https://docs.aws.amazon.com/eks/latest/userguide/horizontal-pod-autoscaler.html) will be configured on both clusters to respond to the need for scaling out in terms of the number of nodes and pods respectively.

In addition, [AWS Load Balancer Controller](https://github.com/aws/eks-charts/tree/master/stable/aws-load-balancer-controller) will be installed on both clusters to create Application Load Balancers and connect them with your Kubernetes services via [Ingress](https://kubernetes.io/docs/concepts/services-networking/ingress/).

![concept](./concept.jpg)

> _**Notice.**_
>
> _All costs of our journey cannot be covered in free tier of AWS account._

## Groundwork

For our journey, we need two EKS Clusters-locust cluster is for load generator, the other is for target of workload. You can setup using below groundwork guide.

### Prerequisites

Install CLI tools and settings.

- [kubectl](https://kubernetes.io/docs/tasks/tools/#kubectl) ([check version release](https://kubernetes.io/releases/))
- [eksctl](https://eksctl.io/introduction/#installation) ([check version release](https://github.com/weaveworks/eksctl/releases))
- [helm](https://helm.sh/docs/intro/install/) ([check version release](https://github.com/helm/helm/releases))
- [jq](https://stedolan.github.io/jq/download/)
- [awscli v2](https://docs.aws.amazon.com/cli/latest/userguide/getting-started-version.html)
- [Setting AWS Profile](https://docs.aws.amazon.com/cli/latest/userguide/cli-configure-profiles.html) ([with minimum IAM policies](https://eksctl.io/usage/minimum-iam-policies/)):
  `aws sts get-caller-identity`
- Pull this repository:
  `git clone https://github.com/aws-samples/load-testing-eks-cluster-with-locust`

### Provisioning EKS Clusters

If you already have clusters, you can skip this steps.

- [Create Locust Cluster](./groundwork/eks-clusters#create-locust-cluster)
- [Create Workload Cluster](./groundwork/eks-clusters#create-workload-cluster) (option)

Please check your AWS Account limits to ensure you can provision 2 VPCs, so if you do not have enough resources in your environment, the above cluster creation will fail.

Then you can check the results in AWS Consoles.

- [AWS Console :: CloudFormation Stacks](https://console.aws.amazon.com/cloudformation/home#/stacks?filteringStatus=active&filteringText=awsblog-loadtest&viewNested=true&hideStacks=false)

  ![cfn-console-result](./groundwork/eks-clusters/result-images/cfn-console.png)

- [AWS Console :: EKS Clusters](https://console.aws.amazon.com/eks/home)

  ![eks-console-result](./groundwork/eks-clusters/result-images/eks-console.png)

### Installing Basic Addon Charts

For load-tesing on EKS, we basically need to install these kubernetes addons charts. These charts are commonly used in EKS clusters, thus we need to do the below installation jobs in both of clusters: `awsblog-loadtest-locust` and `awsblog-loadtest-workload` (or your other target cluster).

- [Deploy Metrics server (YAML)](./groundwork/install-addon-chart#deploy-metrics-server-yaml)
- [Skip installation step for HPA(Horizontal Pod Autoscaler)](./groundwork/install-addon-chart#skip-installation-step-for-hpa)
- [Install Cluster Autoscaler (Chart)](./groundwork/install-addon-chart#install-ca-helm-chart)
- [Install AWS Load Balancer Controller (Chart)](./groundwork/install-addon-chart#install-aws-load-balancer-controller-helm-chart)
- [Install Container Insights (YAML)](./groundwork/install-addon-chart#install-container-insights)

### Installing a Sample Application Helm Chart

As a last step of the groundwork, we need to deploy a sample application to be tested. We are going to use `helm` for this as well. (If you already have an application you want to test, you can skip this step and use your own application).

- [Prepare](./groundwork/install-sample-app#prepare)
- [Install Workload Application (Helm Chart)](./groundwork/install-sample-app#install-workload-application-helm-chart)

Then you can check the result like this.

![ingress-home.png](./groundwork/install-sample-app/ingress-home.png)

![ingress-load-gen.png](./groundwork/install-sample-app/ingress-load-gen.png)

## Walkthrough

You now have the Kubernetes side ready, but there are still a few things left to configure before we start testing your application.

### STEP 1. Install Locust

#### _Switch kubernetes context to run commands on the locust cluster:_

In the above section, we installed a sample application on the workload cluster(`awsblog-loadtest-workload`), in order to test the application, we need to switch kubernetes context from the workload cluster to the locust cluster(`awsblog-loadtest-locust`).

```bash
# Unset context - Optional
kubectl config unset current-context

# Set context of locust cluster
export LOCUST_CLUSTER_NAME="awsblog-loadtest-locust"
export LOCUST_CONTEXT=$(kubectl config get-contexts | sed -e 's/\*/ /' | grep "@${LOCUST_CLUSTER_NAME}." | awk -F" " '{print $1}')
kubectl config use-context ${LOCUST_CONTEXT}

# Check
kubectl config current-context

# Like this..
# <IAM_ROLE>@awsblog-loadtest-locust.<TARGET_REGION>.eksctl.io
```

#### _Add Delivery Hero public chart repo:_

```bash
# Helm chart repo setting
helm repo add deliveryhero "https://charts.deliveryhero.io/"
helm repo update deliveryhero

# Check
helm repo list | egrep "NAME|deliveryhero"
```

#### _Write a `locustfile.py` file:_

Create a file with the code below and save it as `locustfile.py`. If you want to test another endpoint, [edit this line](./walkthrough/locustfile.py#L10) from `/` to another one (like [`/load-gen/loop?count=50&range=1000` what we deployed in groundwork](https://github.com/SPONGE-JL/load-testing-spring-worker#api-list)). For more in depth explanation how to write the `locustfile.py`, you can refer to [the official locust documentation](http://docs.locust.io/en/stable/index.html).

```bash
# Execute in root of repository, and maybe this file has existed.
cat <<EOF > ./walkthrough/locustfile.py
from locust import HttpUser, task, between

default_headers = {'User-Agent': 'Mozilla/5.0 (Macintosh; Intel Mac OS X 10_13_4) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/76.0.3809.100 Safari/537.36'}

class WebsiteUser(HttpUser):
    wait_time = between(1, 5)

    @task(1)
    def get_index(self):
        self.client.get("/", headers=default_headers)
EOF

# Check
cat ./walkthrough/locustfile.py
```

#### Install and Configure Locust with locustfile.py

Executing the following command will install locust using the `locustfile.py` created above. It will also start locust with 5 worker pods with HPA (Horizontal Pod Autoscaler) enabled. This will provide us a good starting point for the load test that automatically scales as the size of the load increases.

```bash
# Create ConfigMap 'eks-loadtest-locustfile' from the locustfile.py created above
kubectl create configmap eks-loadtest-locustfile --from-file ./walkthrough/locustfile.py
kubectl describe configmap eks-loadtest-locustfile

# Install Locust Helm Chart
helm upgrade --install eks-loadtest-locust deliveryhero/locust \
    --set loadtest.name=eks-loadtest \
    --set loadtest.locust_locustfile_configmap=eks-loadtest-locustfile \
    --set loadtest.locust_locustfile=locustfile.py \
    --set worker.hpa.enabled=true \
    --set worker.hpa.minReplicas=5
```

### STEP 2. Expose Locust via ALB

After Locust is successfully installed, create [an Ingress to expose Locust Dashboard](./walkthrough/alb-ingress.yaml) so that you can access it from a browser. Maybe it needs some time (2 minutes or more) to get the enpoint of ALB.

```bash
# Move to the root of directory
# Create Ingress using ALB
kubectl apply -f walkthrough/alb-ingress.yaml
```

### STEP 3. Checkout Locust Dashboard

After creating an Ingress in step 2, you will be able to get an URL of an Application Load Balancer by running the following command. Maybe it needs some time (2 minutes or more) to get the enpoint of ALB.

```bash
# Check
kubectl get ingress ingress-locust-dashboad

# Copy the exposed hostname to clipboard (only mac osx)
kubectl get ingress ingress-locust-dashboad -o jsonpath='{.status.loadBalancer.ingress[0].hostname}' | pbcopy
```

![alb-ingress-kubectl](./walkthrough/result-images/alb-ingress-kubectl.png)

You can also find the same info from AWS Console. It can be found under [EC2 > Load Balancers](https://ap-northeast-2.console.aws.amazon.com/ec2/v2/home?#LoadBalancers).

![alb-ingress-aws-console](./walkthrough/result-images/alb-ingress-aws-console.png)

#### _Open the URL from a browser:_

![locust-dashboard-home](./walkthrough/result-images/locust-dashboard-home.png)

Alternatively, you can just port-forward the connection to the locust dashboard without setting the ingress endpoint. Open your browser and connect to <http://localhost:8089>

```bash
# Port forwarding from local to locust-service resource
kubectl port-forward svc/locust 8089:8089
```

### STEP 4. Run Test

Enter the URL of your application that you deployed. Let’s enter `100` for the number of users and `1` for the spawn rate. Put [the endpoint URL of the workload cluster which was created before](./groundwork/install-sample-app#check-the-api-responses).

<img width="450" alt="locust-dashboard-case1" src="./walkthrough/result-images/locust-dashboard-case1.png">

<br>

Let it run for few minutes and in the meantime, switch between the `Statistics` tab and the `Charts` tab to see how the test is unfolding.

![locust-dashboard-case1-charts](./walkthrough/result-images/locust-dashboard-case1-charts.png)

<br>

In the statistics tab, locust briefly summarize our loads testing result. It is important not to have any noticeable fails counts while loads get spawned, especially when the cluster autoscaler triggers new nodes get provisioned and pods takes time to be initialized and get ready to takes new loads.

![locust-dashboard-case1-statics](./walkthrough/result-images/locust-dashboard-case1-statics.png)

<br>

We can watch Cloudwatch container Insights dashboard to get the glimpse of the basic metrics of our locust namespace.  We can further experiment with the visualization of various metrics including EKS control plane by setting up Prometheus/Grafana dashboard.

![cw-performance-case1-locust](./walkthrough/result-images/cw-performance-case1-locust.png)

![cw-performance-case1-workload](./walkthrough/result-images/cw-performance-case1-workload.png)

Everything looks fine seeing our workload cluster can undertake those loads without any issues. Now we can give it a little more stress. Stop the test for now and put more users in the next step. Let’s put `1,000` users with spawn rate of `10` and compare it with the previous graph.

<img width="450" alt="locust-dashboard-case2" src="./walkthrough/result-images/locust-dashboard-case2.png">

![locust-dashboard-case2-chart](./walkthrough/result-images/locust-dashboard-case2-charts.png)

![locust-dashboard-case2-statics](./walkthrough/result-images/locust-dashboard-case2-statics.png)

It looks similar to previous test. It seems that the service can cover these loads.

### STEP 5. Run Test (2nd)

Now it’s time for a million users and examine for a prolonged time. Let’s put `1,000,000` users with spawn rate of `100`. Both workloads cluster and locust cluster itself has enough resources to cover those traffic, it shows steady upward sloping graph. Suppose we had smaller instances for our clusters or we gave load generator more cpu burden, and when the resource threshold met for cluster autoscalers to kick in to add more nodes, the graph would have reflected several sudden bumps.

<img width="450" alt="locust-dashboard-case3" src="./walkthrough/result-images/locust-dashboard-case3.png">

![locust-dashboard-case3-chart](./walkthrough/result-images/locust-dashboard-case3-charts.png)

![locust-dashboard-case3-statics](./walkthrough/result-images/locust-dashboard-case3-statics.png)

When our cluster need to scale during the high peak of loads, we may see slower response time and even get 50X HTTP responses. Then we need some techniques for our clusters to be more responsive and fault tolerant.  There are some tips that might be useful when load test your own.

- Find the optimal pod’s readiness probes to minimize the time to wait for newly scaled pods.
- Fine tune HPA/CA configuration for the specific workloads in order to the autoscaler as responsive as it can when it need to scale out.
- HPA reaction time + CA reaction time + node provisioning time can take up to 5 minutes, and node provisioning usually takes most of that time, and it is useful to have lighter AMI and have bigger instance type to get utilize of better bin packing efficiency.
- Check with the scaling pod’s entire lifecycle to see if there’re any bottleneck - metric scraping delay + HPA trigger for pods to scale out + container image pulling + application loading time + readiness probe delay
- Overprovisioning employs temporary pods with negative priority and take ample space in the cluster. When the event of scaling action, it can dramatically reduce the node provisioning time and it trades cost for scheduling latency.
- Prevent Scale Down Eviction to the CA if you node scales down during the load testing.
- It’s all about a tradeoff between resource optimization and a cost. You need extra headroom of resources within a budget.

## Clean up

When you are done with your tests, clean up all the testing resources.

```bash
# Setting to clean up target EKS cluster: Workload / Locust

# Set optional environment variables
export AWS_PROFILE="YOUR_PROFILE" # If not, use 'default' profile
export AWS_REGION="YOUR_REGION"   # ex. ap-northeast-2

# Set common environment variables
export TARGET_GROUP_NAME="workload"
export TARGET_CLUSTER_NAME="awsblog-loadtest-${TARGET_GROUP_NAME}"

# Switch Context to executes commands on the target cluster: Workload / Locust
# Unset context
kubectl config unset current-context
# Set context of workload cluster
export TARGET_CONTEXT=$(kubectl config get-contexts | grep "${TARGET_CLUSTER_NAME}" | awk -F" " '{print $1}')
kubectl config use-context ${TARGET_CONTEXT}

# Check if current context set to run on workload cluster
kubectl config current-context
```

```bash
# Clean up target EKS cluster: Workload / Locust

# Uninstall All Helm Charts
helm list -A |grep -v NAME |awk -F" " '{print "helm -n "$2" uninstall "$1}' |xargs -I {} sh -c '{}'

# Check Empty Helm List
helm list -A

# Delete IRSA setting (must run before delete the cluster!)
eksctl delete iamserviceaccount --cluster "${TARGET_CLUSTER_NAME}" \
  --namespace "kube-system" --name "${CA:-cluster-autoscaler}"
eksctl delete iamserviceaccount --cluster "${TARGET_CLUSTER_NAME}" \
  --namespace "kube-system" --name "${AWS_LB_CNTL:-aws-load-balancer-controller}"

# Delete target EKS cluster: Workload / Locust
eksctl delete cluster --name="${TARGET_CLUSTER_NAME}"

# Delete ECR Repository: Workload only
aws ecr delete-repository --repository-name "${ECR_REPO_NAME:-sample-application}"
```

It’s done. One things to note, don’t forget to remove **both clusters: `Locust` and `Workload`**.

## Summary

In this blog, we set up two EKS clusters and tools for load testing our workloads. With the cluster autoscaler in action, we could clearly watch high loads generated by Locust were handled smoothly as expected. Locust dashboard and Cloudwatch insight are useful to monitor maximum loads our cluster can withstand.

We can experiment further by customizing `locustfile.py` script with various testing scenarios reflecting real world use cases. In our next blog posting, we will introduce various techniques to make our EKS cluster better handle spiky high load of traffic during load testing.

TAGS: [EKS](https://aws.amazon.com/blogs/containers/tag/eks/), [Loadtest](https://aws.amazon.com/blogs/devops/tag/load-test/), [Locust](https://aws.amazon.com/blogs/devops/tag/locust/)

## Security

See [CONTRIBUTING](CONTRIBUTING.md#security-issue-notifications) for more information.

## License

This library is licensed under the MIT-0 License. See the LICENSE file.
