# Deploying Hot R.O.D. in AWS/EKS

!!! important "Enabling µAPM"

    An Organization needs to be pre-provisioned as a µAPM entitlement is required for the purposes of this module. Please contact someone from SignalFx to get a trial instance with µAPM enabled if you don’t have one already.

    To check if you have an Organization with µAPM enabled, just login to SignalFx and check that you have the µAPM tab on the top navbar next to Dashboards.

## 1. Launch the Multipass instance

If you have not already completed the [Smart Agent Preparation](../../smartagent/prep/), then please do so, otherwise jump to [Step #2](../../module8/hotrod-eks/#2-create-environment-variables)

## 2. Create environment variables

Create the following environment variables for **SignalFx** and **AWS** to use in the proceeding steps:

=== "SignalFx"

    ```
    export ACCESS_TOKEN=[ACCESS_TOKEN]
    export REALM=[REALM e.g. us1]
    ```

=== "AWS"

    ```
    export AWS_ACCESS_KEY_ID=[AWS Access Key]
    export AWS_SECRET_ACCESS_KEY=[AWS Secret Access Key]
    export AWS_DEFAULT_REGION=[e.g. us-east-1]
    export AWS_DEFAULT_OUTPUT=json
    export EKS_CLUSTER_NAME=$(hostname)-APP-DEV
    ```

You can check for the latest SignalFx Smart Agent release on [Github](https://github.com/signalfx/signalfx-agent/releases){: target=_blank}.

## 3. Configure AWS CLI for your account

Use the `AWS CLI` to configure access to your AWS environment. The environment variables configured above mean you can just hit enter on each of the prompts to accept the values:

=== "Shell Command"

    ```
    aws configure
    ```

=== "Output"

    ```
    AWS Access Key ID [****************TVAQ]: 
    AWS Secret Access Key [****************MkB4]: 
    Default region name [us-east-1]: 
    Default output format [json]: 
    ```

---

## 4. Create a cluster running Amazon Elastic Kubernetes Service (EKS)

=== "Shell Command"

    ```
    eksctl create cluster \
    --name $EKS_CLUSTER_NAME \
    --region $AWS_DEFAULT_REGION \
    --node-type t3.medium \
    --nodes-min 3 \
    --nodes-max 7 \
    --version=1.15
    ```

=== "Output"

    ```
    [ℹ]  eksctl version 0.16.0
    [ℹ]  using region us-east-1
    [ℹ]  setting availability zones to [us-east-1a us-east-1f]
    [ℹ]  subnets for us-east-1a - public:192.168.0.0/19 private:192.168.64.0/19
    [ℹ]  subnets for us-east-1f - public:192.168.32.0/19 private:192.168.96.0/19
    [ℹ]  nodegroup "ng-371a784a" will use "ami-0e5bb2367e692b807" [AmazonLinux2/1.15]
    [ℹ]  using Kubernetes version 1.15
    [ℹ]  creating EKS cluster "EKS-APP-DEV" in "us-east-1" region with un-managed nodes
    [ℹ]  will create 2 separate CloudFormation stacks for cluster itself and the initial nodegroup
    [ℹ]  if you encounter any issues, check CloudFormation console or try 'eksctl utils describe-stacks --region=us-east-1 --cluster=EKS-APP-DEV'
    [ℹ]  CloudWatch logging will not be enabled for cluster "EKS-APP-DEV" in "us-east-1"
    [ℹ]  you can enable it with 'eksctl utils update-cluster-logging --region=us-east-1 --cluster=EKS-APP-DEV'
    [ℹ]  Kubernetes API endpoint access will use default of {publicAccess=true, privateAccess=false} for cluster "EKS-APP-DEV" in "us-east-1"
    [ℹ]  2 sequential tasks: { create cluster control plane "EKS-APP-DEV", create nodegroup "ng-371a784a" }
    [ℹ]  building cluster stack "eksctl-EKS-APP-DEV-cluster"
    [ℹ]  deploying stack "eksctl-EKS-APP-DEV-cluster"
    [ℹ]  building nodegroup stack "eksctl-EKS-APP-DEV-nodegroup-ng-371a784a"
    [ℹ]  deploying stack "eksctl-EKS-APP-DEV-nodegroup-ng-371a784a"
    [✔]  all EKS cluster resources for "EKS-APP-DEV" have been created
    [!]  unable to write kubeconfig , please retry with 'eksctl utils write-kubeconfig -n EKS-APP-DEV': unable to modify kubeconfig /home/ubuntu/.kube/config: open /etc/rancher/k3s/k3s.yaml.lock: permission denied
    [ℹ]  adding identity "arn:aws:iam::327192335161:role/eksctl-EKS-APP-DEV-nodegroup-ng-3-NodeInstanceRole-2RMH7RBODD62" to auth ConfigMap
    [ℹ]  nodegroup "ng-371a784a" has 0 node(s)
    [ℹ]  waiting for at least 3 node(s) to become ready in "ng-371a784a"
    [ℹ]  nodegroup "ng-371a784a" has 3 node(s)
    [ℹ]  node "ip-192-168-35-104.ec2.internal" is ready
    [ℹ]  node "ip-192-168-52-88.ec2.internal" is ready
    [ℹ]  node "ip-192-168-8-236.ec2.internal" is ready
    [✔]  EKS cluster "EKS-APP-DEV" in "us-east-1" region is ready
    ```

This may take some time (10-15 minutes). Ensure you see your cluster active in AWS EKS console before proceeding.

!!! note
    You can ignore the error about `unable to write kubeconfig` as we address this below.

Once complete update your `kubeconfig` to allow `sudo kubectl` access to the cluster:

=== "Shell Command"

    ```
    sudo eksctl utils write-kubeconfig -n $EKS_CLUSTER_NAME
    ```

---

## 5. Deploy SignalFx SmartAgent to your EKS Cluster

Add the SignalFx Helm chart repository to Helm:

=== "Shell Command"

    ```text
    helm repo add signalfx https://dl.signalfx.com/helm-repo
    ```

=== "Output"

    ```
    "signalfx" has been added to your repositories
    ```

Ensure the latest state of the SignalFx Helm repository:

=== "Shell Command"

    ```text
    helm repo update
    ```

=== "Output"

    ```
    Hang tight while we grab the latest from your chart repositories...
    ...Successfully got an update from the "signalfx" chart repository
    ```

Install the Smart Agent Helm chart with the following commands:

=== "Shell Command"

    ```
    helm install \
    --set signalFxAccessToken=$ACCESS_TOKEN \
    --set clusterName=$EKS_CLUSTER_NAME \
    --set signalFxRealm=$REALM \
    --set kubeletAPI.url=https://localhost:10250 \
    --set traceEndpointUrl=https://ingest.$REALM.signalfx.com/v2/trace \
    signalfx-agent signalfx/signalfx-agent \
    -f workshop/k3s/values.yaml
    ```

=== "Output"

    ```
    NAME: signalfx-agent
    LAST DEPLOYED: Mon Apr 13 14:23:19 2020
    NAMESPACE: default
    STATUS: deployed
    REVISION: 1
    TEST SUITE: None
    NOTES:
    The SignalFx agent is being deployed in your Kubernetes cluster.  You should
    see metrics flowing once the agent image is downloaded and started (this may
    take a few minutes since it has to download the agent container image).

    Assuming you are logged into SignalFx in your browser, visit

    https://app.us0.signalfx.com/#/navigator/kubernetes%20pods/kubernetes%20pods

    to see all of the pods in your cluster.
    ```

Validate cluster looks healthy in SignalFx Kubernetes Navigator dashboard

---

## 6. Deploy Hot R.O.D. Application to EKS

=== "Shell Command"

    ```
    sudo kubectl apply -f ~/workshop/apm/hotrod/k8s/deployment.yaml
    ```

To ensure the Hot R.O.D. application is running see examples below:

=== "Shell Command"

    ```text
    sudo kubectl get pods
    ```

=== "Output"

    ```text
    NAME                      READY   STATUS    RESTARTS   AGE
    hotrod-7564774bf5-vjpfw   1/1     Running   0          47h
    signalfx-agent-jmq4f      1/1     Running   0          138m
    signalfx-agent-nk8p9      1/1     Running   0          138m
    signalfx-agent-q5tzh      1/1     Running   0          138m
    ```

You then need find the IP address assigned to the Hot R.O.D. service:

=== "Shell Command"

    ```text
    sudo kubectl get svc
    ```

=== "Output"

    ```text
    NAME         TYPE           CLUSTER-IP       EXTERNAL-IP                                                               PORT(S)          AGE
    hotrod       LoadBalancer   10.100.188.249   af26ce80ef2e14c9292ae5b4bc0d2dd0-1826890352.us-east-2.elb.amazonaws.com   8080:32521/TCP   47h
    kubernetes   ClusterIP      10.100.0.1       <none>                                                                    443/TCP          3d1h
    ```

Create an environment variable for the IP address and port that the Hot R.O.D. application is exposed on:

=== "Shell Command"

    ```
    HOTROD_ENDPOINT=$(sudo kubectl get svc hotrod -n default -o jsonpath='{.spec.clusterIP}:{.spec.ports[0].port}')
    ```

You can view / exercise Hot R.O.D. yourself in a browser by opening the `EXTERNAL-IP:PORT` as shown above e.g.

=== "Example URL"

    ```text
    https://af26ce80ef2e14c9292ae5b4bc0d2dd0-1826890352.us-east-2.elb.amazonaws.com:8080
    ```

---

## 7. Generate some traffic to the application using Siege Benchmark

=== "Shell Command"

    ```text
    siege -r10 -c10 "http://$HOTROD_ENDPOINT/dispatch?customer=392&nonse=0.17041229755366172"
    ```

Create some errors with an invalid customer number

=== "Shell Command"

    ```text
    siege -r10 -c10 "http://$HOTROD_ENDPOINT/dispatch?customer=391&nonse=0.17041229755366172"
    ```

You should now be able to exercise SignalFx APM dashboards.

---

## 8. Cleaning up

To delete entire EKS cluster:

=== "Shell Command"

    ```
    eksctl delete cluster $EKS_CLUSTER_NAME
    ```

=== "Output"

    ``` 
    [ℹ]  eksctl version 0.16.0
    [ℹ]  using region us-east-1
    [ℹ]  deleting EKS cluster "RWC-APP-DEV"
    [ℹ]  deleted 0 Fargate profile(s)
    [ℹ]  cleaning up LoadBalancer services
    [ℹ]  2 sequential tasks: { delete nodegroup "ng-371a784a", delete cluster control plane "EKS-APP-DEV" [async] }
    [ℹ]  will delete stack "eksctl-EKS-APP-DEV-nodegroup-ng-371a784a"
    [ℹ]  waiting for stack "eksctl-EKS-APP-DEV-nodegroup-ng-371a784a" to get deleted
    [ℹ]  will delete stack "eksctl-EKS-APP-DEV-cluster"
    [✔]  all cluster resources were deleted
    ```

Or to delete individual components:

=== "Shell Command"

    ```
    sudo kubectl delete deploy/hotrod svc/hotrod
    helm delete signalfx-agent
    ```

To switch back to using the local K3s cluster:

=== "Shell Command"

    ```
    sudo sudo kubectl config use-context default
    ```
