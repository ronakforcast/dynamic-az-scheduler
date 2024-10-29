# dynamic-az-scheduler - Dynamic AWS Availability Zone Checker for Kubernetes

This repository contains a Kubernetes solution to dynamically check for available AWS Availability Zones (AZs) and ensure that selected pods are scheduled within available zones. This solution leverages Kubernetes ConfigMaps, ServiceAccounts, Role-Based Access Control (RBAC), and Kyverno policies to achieve high availability and optimal pod placement.

## Table of Contents
- [Overview](#overview)
- [Architecture](#architecture)
- [Prerequisites](#prerequisites)
- [Deployment](#deployment)
- [Usage](#usage)
- [How It Works](#how-it-works)
- [Customization](#customization)
- [Troubleshooting](#troubleshooting)
- [Contributing](#contributing)
- [License](#license)

## Overview

This solution periodically checks for available AWS Availability Zones in a specified region and updates Kubernetes to reflect the current available zone. A Kyverno policy then ensures that specific pods are scheduled on nodes within the available zone, adapting automatically to changes in AWS zone availability.

## Architecture

1. **ConfigMap (`az-check-script`)**: Contains a shell script that checks for available AZs in AWS.
2. **ServiceAccount** and **RBAC**: Allows the deployment to update Kubernetes ConfigMaps dynamically.
3. **Deployment**: Runs a pod that executes the `check-az.sh` script every 60 seconds to check and update the available zone.
4. **Kyverno Policy**: Ensures pods are scheduled on nodes within the available AWS AZ by adding node affinity based on the latest available zone.
5. **IAM Role**: An IAM role attached to the worker nodes provides the necessary AWS permissions, eliminating the need to store AWS credentials.

## Prerequisites

- **Kubernetes Cluster**: A running Kubernetes cluster (EKS recommended) with nodes that have IAM roles attached.
- **AWS CLI**: Ensure that AWS CLI is available in the container.
- **IAM Role for Worker Nodes**:
  - The IAM role attached to worker nodes must have `ec2:DescribeAvailabilityZones` permissions.
  
  Example IAM Policy:
  ```json
  {
    "Version": "2012-10-17",
    "Statement": [
      {
        "Effect": "Allow",
        "Action": "ec2:DescribeAvailabilityZones",
        "Resource": "*"
      }
    ]
  }
  ```

## Deployment

To deploy this solution, follow these steps:

1. **Clone the Repository**:
   ```bash
   git clone https://github.com/ronakforcast/dynamic-az-scheduler.git
   cd yourrepo
   ```

2. **Apply Kubernetes Manifests**:
   Deploy the solution by applying all YAML files in this repository:
   ```bash
   kubectl apply -f .
   ```

3. **Verify the Deployment**:
   Confirm the deployment by checking if the `az-checker` pod is running:
   ```bash
   kubectl get pods -l app=az-checker
   ```

   Check if the `az-availability` ConfigMap is being updated:
   ```bash
   kubectl get configmap az-availability -o yaml
   ```

4. **Verify the Kyverno Policy**:
   Ensure that Kyverno is managing pod scheduling based on the `az-availability` ConfigMap. You can observe the node affinity applied to pods by inspecting their configurations.

## Usage

This solution automatically updates the available AWS AZ and applies it to Kubernetes pods that match specified criteria.

### To Check Current Available Zone

1. View the `az-availability` ConfigMap for the latest available zone:
   ```bash
   kubectl get configmap az-availability -o yaml
   ```

### To Test Scheduling

1. Deploy a sample pod that matches the Kyverno policy criteria and observe that it is scheduled in the available zone:
   ```yaml
   apiVersion: v1
   kind: Pod
   metadata:
     name: sample-pod
     labels:
       app: app1  # This label should match the Kyverno policy
   spec:
     containers:
     - name: nginx
       image: nginx
   ```

2. Check the node on which the pod is scheduled to confirm it’s in the correct zone:
   ```bash
   kubectl describe pod sample-pod | grep -i node
   ```

## How It Works

1. **Script Execution**: The `az-checker` pod runs the `check-az.sh` script, which:
   - Iterates over a predefined list of Availability Zones.
   - Uses AWS CLI to determine if each zone is available.
   - Updates the `az-availability` ConfigMap with the name of an available zone.

2. **Kyverno Policy**: A Kyverno policy (`schedule-pod-in-zone`) reads the available zone from `az-availability` ConfigMap and applies `nodeAffinity` to pods that meet the specified criteria, ensuring they are scheduled on nodes within the available zone.

3. **Dynamic Adaptation**: Every 60 seconds, the script checks AWS again. If the available zone changes, the `az-availability` ConfigMap is updated, and future pods matching the policy will adapt to the new available zone.

## Customization

- **Update Availability Zones**:
  Modify the `AZ_LIST` variable in the `check-az.sh` script within the `az-check-script` ConfigMap to adjust the list of AWS zones to monitor.
  
- **Kyverno Policy Criteria**:
  Adjust the `matchExpressions` or `namespaces` in the Kyverno policy to change which pods should adhere to the node affinity for the available zone.

## Troubleshooting

- **No Available Zone Detected**:
  If `az-availability` ConfigMap remains empty or reports "No available zones found," verify your AWS permissions and the correctness of the `AZ_LIST`.

- **Pods Not Scheduled in Expected Zone**:
  Confirm that the Kyverno policy matches your pods and that the worker nodes are within the specified zones.

- **Permissions Issues**:
  Ensure that the IAM role attached to worker nodes has the `ec2:DescribeAvailabilityZones` permission.

