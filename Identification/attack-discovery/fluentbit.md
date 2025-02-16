# Logging
Logging is an important part of the observability story. Oftentimes, logs can help you discover runtime issues occuring within a cluster. They are also used by developers and clusters administrators alike to debug issues that aren't readily apparent. Although Kubernetes has a built-in mechanism for retrieving logs from containers, `kubectl logs`, those logs are not preserved indefinitely. Additionally, `kubectl logs` only captures `stdout` and `stderr` from containers. Using a node agent like Fluent Bit to stream logs off cluster is considered a best practice because it allows you to maintain an archive of your logs which may be useful during an forensics investigation where you're trying to identify the root cause of a breach. 

In this workshop, the logs may help you glean additional information about origin and nature of the attack. 

## Installing Fluent Bit as a cluster DaemonSet
[Fluent Bit](https://fluentbit.io/ "Fluent Bit Project") is an open source log processor and forwarder. Fluent Bit is often the preferred choice for containerized environments like Kubernetes because it is efficient, feature rich, and supports a variety of different backends. 

In this workshop you will install Fluent Bit as a DaemonSet and configure it to send a variety of different logs to CloudWatch. When you complete these steps, Fluent Bit will create the following log groups if they don't already exist.

| Log group name                                  | Log source                                                                              |
| ----------------------------------------------- | --------------------------------------------------------------------------------------- |
| /aws/containerinsights/Cluster-Name/application | All log files in /var/log/containers                                                    |
| /aws/containerinsights/Cluster-Name/host        | Logs from /var/log/dmesg, /var/log/secure, and /var/log/messages                        |
| /aws/containerinsights/Cluster-Name/dataplan    | The logs in /var/log/journal for kubelet.service, kubeproxy.service, and docker.service |

## Prerequisites
[eksctl](https://docs.aws.amazon.com/eks/latest/userguide/eksctl.html) is used to easily configure IAM Roles for Service Accounts (IRSA) in the cluster. The following commands will install eksctl locally in your Cloud9 Environment.

```bash
curl --silent --location "https://github.com/weaveworks/eksctl/releases/latest/download/eksctl_$(uname -s)_amd64.tar.gz" | tar xz -C /tmp
sudo mv -v /tmp/eksctl /usr/local/bin
eksctl version
eksctl completion bash >> ~/.bash_completion
. /etc/profile.d/bash_completion.sh
. ~/.bash_completion
```

Setting default region for rest of commands to work without needing --region flag 
```bash
aws configure
AWS Access Key ID [None]: 
AWS Secret Access Key [None]: 
Default region name [None]: us-west-2
Default output format [None]: json
```




## Installation process
1. If you don't already have a kubernetes namespace named amazon-cloudwatch, create one by using the following command:

    ```bash
    kubectl apply -f https://raw.githubusercontent.com/aws-samples/amazon-cloudwatch-container-insights/latest/k8s-deployment-manifest-templates/deployment-mode/daemonset/container-insights-monitoring/cloudwatch-namespace.yaml
    ```

2. The Fluent Bit daemon will need permission to access CloudWatch. We will create an IAM Role for the fluent-bit service account and permit that service account full access to CloudWatch.

    a. You will need two components in the cluster in order to map IAM role policies to a service account running the Fluent Bit DaemonSet. First, we will ensure that there is an OpenID Connect (OIDC) provider for our cluster. Let's list the cluster's IAM OIDC provider URL:
    
    `aws eks describe-cluster --name security-workshop --query "cluster.identity.oidc.issuer" --output text`

    You the command will return a URL like this exameple:

    `https://oidc.eks.us-east-1.amazonaws.com/id/EXAMPLE61013F13DDF959BD02B192F31`

    Now list the IAM OIDC providers in your account. Replace `EXAMPLE61013F13DDF959BD02B192F31` with the value returned from the previous command.

    `aws iam list-open-id-connect-providers | grep <EXAMPLED539D4633E53DE1B716D3041>`

    If the command returns a string like the following, your cluster already has a configured IAM OIDC provider.
    
    `"Arn": "arn:aws:iam::611769228671:oidc-provider/oidc.eks.us-east-1.amazonaws.com/id/3CB3DF161013F13DDF959BD02B192F31"`

    If the command does not return an ARN, you will need to create an IAM OIDC provider. Execute the following commands to create an IAM OIDC provider.

    `eksctl utils associate-iam-oidc-provider --cluster security-workshop --approve`

    The following command will now create am IAM service account named `fluent-bit` in the `amazon-cloudwatch` NameSpace of the `security-workshop` cluster.
    ```bash
    eksctl create iamserviceaccount \
    --cluster security-workshop \
    --namespace amazon-cloudwatch \
    --name fluent-bit \
    --attach-policy-arn arn:aws:iam::aws:policy/CloudWatchFullAccess \
    --override-existing-serviceaccounts \
    --approve
    ```

3. Run the following command to create a ConfigMap named cluster-info with the cluster name and the region to send logs. Replace cluster-name and cluster-region with your cluster's name and Region.

    ```bash
    #set the following environment variables using the appropriate ClusterName and RegionName
    ClusterName=security-workshop
    RegionName=`curl http://169.254.169.254/latest/dynamic/instance-identity/document|grep region|awk -F\" '{print $4}'`  
    FluentBitHttpPort='2020'
    FluentBitReadFromHead='Off'
    [[${FluentBitReadFromHead} = 'On']] && FluentBitReadFromTail='Off'|| FluentBitReadFromTail='On'
    [[-z ${FluentBitHttpPort}]] && FluentBitHttpServer='Off' || FluentBitHttpServer='On'
    kubectl create configmap fluent-bit-cluster-info \
            --from-literal=cluster.name=${ClusterName} \
            --from-literal=http.server=${FluentBitHttpServer} \
            --from-literal=http.port=${FluentBitHttpPort} \
            --from-literal=read.head=${FluentBitReadFromHead} \
            --from-literal=read.tail=${FluentBitReadFromTail} \
            --from-literal=logs.region=${RegionName} \
            -n amazon-cloudwatch
    ```

    In the previous command, FluentBitHttpServer for monitoring plugin metrics is on by default. To turn it off, change the third line in the command to FluentBitHttpPort=""" (an empty string) in the command. Also by default, Fluent Bit reads log files from the tail, and will only capture logs created after the DeamonSet was deployed. If you want the opposite, set FluentBitReadFromHead="On" and it will collect all logs in the file system.

4. Download and deploy the Fluent Bit daemonset manifest to the cluster by running the following command:

    ```bash
    kubectl apply -f https://raw.githubusercontent.com/aws-samples/amazon-cloudwatch-container-insights/latest/k8s-deployment-manifest-templates/deployment-mode/daemonset/container-insights-monitoring/fluent-bit/fluent-bit.yaml
    ```

5. Validate the Fluent Bit deployment by entering the following command. Each node should have one pod named fluent-bit-\*.

    `kubectl get pods -n amazon-cloudwatch`

The above steps create the following cluster resources:

| Resource Kind      | Resource Name           | Resource Use                                                                           |
| ------------------ | ----------------------- | -------------------------------------------------------------------------------------- |
| NameSpace          | amazon-cloudwatch       | contains fluent-bit resources                                                          |
| DaemonSet          | fluent-bit              | Contains the Pods that run Fluent Bit                                                  |
| ConfigMap          | fluent-bit-config       | Contains the configuration to be used by Fluent Bit.                                   |
| ServiceAccount     | fluent-bit              | Used to run Fluent Bit                                                                 |
| ClusterRole        | fluent-bit-role         | Grants get, list, and watch permissions on pod logs to the Fluent-Bit service account. |
| ClusterRoleBinding | fluent-bit-role-binding | Binds the fluent-bit ServiceAccount to the fluent-bit-role                             |

### Verifing Installation

- List the newly created CloudWatch LogGroups.

    ```bash
    aws logs describe-log-groups --query 'logGroups[*].logGroupName' | grep "container"
    ```

- Check the returned list of log groups should include the following:

    ```
    /aws/containerinsights/{ClusterName}/application
    /aws/containerinsights/{ClusterName}/host
    /aws/containerinsights/{ClusterName}/dataplane
    ```

    > There may be a delay in the creation of log groups and log streams.

## Troubleshooting

- If you don't see these log groups and are looking in the correct Region, check the logs of the Fluent Bit DaemonSet pods to look for the error.

    `kubectl logs fluent-bit-{unique-id}`

- If you see errors related to IAM permissions, check the IAM role attached to the cluster nodes or fluent-bit IAM service account.

- If the pod status is CreateContainerConfigError, get the exact error by running the following command.

    `kubectl describe pod {pod_name} -n amazon-cloudwatch`

# Additional Information:

- Setting up Fluent Bit as a DaemonSet to send logs to CloudWatch <https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/Container-Insights-setup-logs-FluentBit.html>
