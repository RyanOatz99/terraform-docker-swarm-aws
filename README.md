# AWS Docker Swarm Terraform Module

This is a Terraform configuration that sets up a Docker Swarm on an existing VPC with a configurable amount of managers and worker nodes. The swarm is configured to have TLS enabled.

## Terraformed layout

In the VPC there will be 2 x _number of availability zones in region_ subnets created. Each EC2 instance will be placed in an subnet in a round-robin fashion.

There are no elastic IPs allocated in the module in order to prevent using up the elastic IP allocation for the VPC. It is up to the caller to set that up.

## Prerequisites

The `aws` provider is configured in your TF file.

AWS permissions to do the following

- Manage EC2 resource
- Security Groups
- IAM permissions
- S3 Create and Access

## Limitations

- Maximum of 240 docker managers.
- Maximum of 240 docker workers.
- Only one VPC and therefore only one AWS region.
- The VPC must have to following properties
  - The VPC should have access to the Internet
  - The DNS hostnames support must be enabled (otherwise the node list won't work too well)
  - VPC must have a CIDR block mask of `/16`.

## Example

The `examples/simple` folder shows an example of how to use this module.

## Usage of S3

S3 was used because EFS and SimpleDB (both better choices in terms of cost and function) are NOT available in `ca-central-1` and likely some other non-US regions.

## Cloud Config merging

The default merge rules of cloud-config is used which may yield unexpected results (see [cloudconfig merge behaviours](https://jen20.com/2015/10/04/cloudconfig-merging.html)) if you are changing existing keys. To bring back the merge behaviour from 1.2 add

    merge_how: "list(append)+dict(recurse_array)+str()"

## Upgrading the swarm

Though `yum update` can simply update the software, it may be required to update things that are outside such as updates to the module itself, `cloud_config_extra` information or AMI updates. To do such an update without having to recreate the swarm it is best to do it one manager node at a time and do `manager0` last. This module ignores changes to cloud config or AMI information in order to prevent updates of those to force a new resource inadvertently.

The first thing to do is update the workers.  It is important not to do all the workers at once unless you are certain your swarm can handle the reduced capacity.  You can do the workers in batches but generally the process is

    for each batch of workers
      for each worker in batch
        on a manager: sudo docker node update --availability drain <worker>
        on worker: sudo docker swarm leave
        terraform taint --module=docker-swarm aws_instance.worker.<worker index>
      terraform apply

Once the workers are done the managers have to be done next.  The process is similar with the addition of a demotion to take the node out of manager status first and when they rejoin the swarm they will be managers again.

    for each manager > 0
      on manager0: sudo docker node demote <manager>
      on manager0: sudo docker node update --availability drain <manager>
      on manager: sudo docker swarm leave
      terraform taint --module=docker-swarm aws_instance.manager.<manager index>
    terraform apply
    on manager1: sudo docker node demote manager0
    on manager1: sudo docker node update --availability drain manager0
    terraform taint --module=docker-swarm aws_instance.manager.0
    terraform apply

By doing the above, you can let the raft consensus recover itself.

Before updating `manager0` make sure `manager1` is up and running as it checks it it can join using `manager1` otherwise it will initialize a new swarm.
