CloudFormation template for an [Exhibitor](https://github.com/Netflix/exhibitor)-managed [ZooKeeper](http://zookeeper.apache.org/) cluster.

## Overview

This template bootstraps a ZooKeeper cluster. The ZK nodes are managed by Exhibitor with S3 for backups and automatic node discovery.

ZooKeeper and Exhibitor are run via a Docker container. You may use the default ([thefactory/zookeeper-exhibitor](https://github.com/thefactory/docker-zk-exhibitor)) or provide your own image.

The servers are part of an auto-scaling group. Incrementing, decrementing, or otherwise modifying the server list should be handled gracefully by ZooKeeper (thanks to Exhibitor).

The template creates a security group for ZK clients, the id for which is exposed as an output (`ClientSecurityGroup`).

The template also creates an internal-facing ELB for clients to interact with Exhibitor via a static endpoint. This is especially useful for node discovery, so Exhibitor's `/cluster/list` API is exposed as an output as well (`ExhibitorDiscoveryUrl`).

Note that this template must be used with Amazon VPC. New AWS accounts automatically use VPC, but if you have an old account and are still using EC2-Classic, you'll need to modify this template or make the switch.

## Usage

### 1. Clone the repository
```bash
git clone git@github.com:mbabineau/cloudformation-zookeeper.git
```

### 2. Create an Admin security group
This is a VPC security group containing access rules for cluster administration, and should be locked down to your IP range, a bastion host, or similar. This security group will be associated with the ZooKeeper servers.

Inbound rules are at your discretion, but you may want to include access to:
* `22 [tcp]` - SSH port
* `2181 [tcp]` - ZooKeeper client port
* `8181 [tcp]` - Exhibitor HTTP port (for both web UI and REST API)

### 3. Launch the stack
Launch the stack via the AWS console, a script, or [aws-cli](https://github.com/aws/aws-cli).

See `zookeeper.json` for the full list of parameters, descriptions, and default values.

Example using `aws-cli`:
```bash
aws cloudformation create-stack \
    --template-body file://zookeeper.json \
    --stack-name <stack> \
    --capabilities CAPABILITY_IAM \
    --parameters \
        ParameterKey=KeyName,ParameterValue=<key> \
        ParameterKey=ExhibitorS3Bucket,ParameterValue=<bucket> \
        ParameterKey=ExhibitorS3Region,ParameterValue=<region> \
        ParameterKey=ExhibitorS3Prefix,ParameterValue=<cluster_name> \
        ParameterKey=VpcId,ParameterValue=<vpc_id> \
        ParameterKey=Subnets,ParameterValue='<subnet_id_1>\,<subnet_id_2>' \
        ParameterKey=AdminSecurityGroup,ParameterValue=<sg_id>
```

### 4. Watch the cluster converge
Once the stack has been provisioned, visit `http://<host>:8181/exhibitor/v1/ui/index.html` on one of the nodes. You will need to do this from a location granted access by the specified `AdminSecurityGroup`.

_Note the Docker image may take several minutes to retrieve. This can be improved with the use of a private Docker registry._

You should see Exhibitor's management UI with a list of ZK nodes in this cluster. Exhibitor adds each node to the cluster via a rolling restart, so you may see nodes getting added and restarting during the first few minutes they're up.
