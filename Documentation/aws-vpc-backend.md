# Amazon VPC Backend for Flannel

When running within an Amazon VPC, we recommend using the aws-vpc backend which, instead of using encapsulation, manipulates IP routes to achieve maximum performance. Because of this, a separate flannel interface is not created.

The biggest advantage of using flannel AWS-VPC backend is that the AWS knows about that IP. That makes it possible to set up ELB to route directly to that container.

To run flannel on AWS, first create an [Amazon VPC](http://aws.amazon.com/vpc/).
Amazon VPC enables us to launch EC2 instances into a virtual network, which may be configured via its route table.

From the VPC dashboard, run the "VPC Wizard":

- Select "VPC with a Single Public Subnet"
- Configure the network and the subnet address ranges

<br/>
<div class="row">
  <div class="col-lg-10 col-lg-offset-1 col-md-10 col-md-offset-1 col-sm-12 col-xs-12 co-m-screenshot">
    <a href="img/aws-vpc.png">
      <img src="img/aws-vpc.png" alt="New Amazon VPC" />
    </a>
  </div>
</div>
<div class="caption">Creating a new Amazon VPC</div>

With a VPC and subnet set up, create an Identity and Access Management ([IAM](http://aws.amazon.com/iam/)) role to grant the required permissions to the EC2 instances.

From the console, select Services > Security & Identity > IAM.

Create a [policy](http://docs.aws.amazon.com/IAM/latest/UserGuide/policies_overview.html) that will later be assigned to an IAM role.
Under "Policies" on the left, select "Create Policy", then "Create Your Own Policy".
The following permissions are required in the sample policy document.

- ec2:CreateRoute
- ec2:DeleteRoute
- ec2:ReplaceRoute
- ec2:DescribeRouteTables
- ec2:DescribeInstances

```
{
  "Version": "2012-10-17",
  "Statement": [
    {
          "Effect": "Allow",
          "Action": [
              "ec2:CreateRoute",
              "ec2:DeleteRoute",
              "ec2:ReplaceRoute"
          ],
          "Resource": [
              "*"
          ]
    },
    {
          "Effect": "Allow",
          "Action": [
              "ec2:DescribeRouteTables",
              "ec2:DescribeInstances"
          ],
          "Resource": "*"
    }
  ]
}
```

Note that although the first three permissions can be tied to the route table resource of our subnet, the ec2:Describe\* permissions can not be limited to a particular resource.
For simplicity, leave the "Resource" as wildcard in both.

With the policy added, attach it to a new IAM role by clicking "Create New Role" and setting the following options:
Under "Roles" on the left, select "Create New Role".

- Role Name: `demo-role`
- Role Type: "Amazon EC2"
- Attach the policy  created earlier

Now, launch an EC2 instance.
In the launch wizard, choose the `CoreOS-stable-681.2.0` image and under "Configure Instance Details" perform the following steps:

- Change the "Network" to the VPC just created
- Enable "Auto-assign Public IP"
- Select IAM `demo-role`

<br/>
<div class="row">
  <div class="col-lg-10 col-lg-offset-1 col-md-10 col-md-offset-1 col-sm-12 col-xs-12 co-m-screenshot">
    <a href="img/aws-instance-configuration.png" class="co-m-screenshot">
      <img src="img/aws-instance-configuration.png" alt="AWS EC2 Instance Details" />
    </a>
  </div>
</div>
<div class="caption">Configuring AWS EC2 instance details</div>

Under the "Configure Security Group" tab add the rules to allow etcd traffic (tcp/2379), SSH and ICMP.

Go ahead and launch the instance!

Because the instance will be sending and receiving traffic for IPs other than the one assigned by our subnet, we need to disable source/destination checks.

<br/>
<div class="row">
  <div class="col-lg-10 col-lg-offset-1 col-md-10 col-md-offset-1 col-sm-12 col-xs-12 co-m-screenshot">
    <a href="img/aws-src-dst-check.png" class="co-m-screenshot">
      <img src="img/aws-src-dst-check.png" alt="Disable AWS Source/Dest Check" />
    </a>
  </div>
</div>
<div class="caption">Disable AWS Source/Dest Check</div>

All that’s left now is to start etcd, publish the network configuration and run the flannel daemon.
First, SSH into `demo-instance-1`:

- Start etcd:

```
$ etcd --advertise-client-urls http://$INTERNAL_IP:2379 --listen-client-urls http://0.0.0.0:2379
```
- Publish configuration in etcd (ensure that the network range does not overlap with the one configured for the VPC).  This will
  attempt to automatically determine your route table.

```
$ etcdctl set /coreos.com/network/config '{"Network":"10.20.0.0/16", "Backend": {"Type": "aws-vpc"}}'
```

- If you want to manually specify your route table ID or if you want to update multiple route tables, e.g. for a deployment across multiple availability zones, use either a string for one or an array for one or more route tables like this.

```
$ etcdctl set /coreos.com/network/config '{"Network":"10.20.0.0/16", "Backend": {"Type": "aws-vpc", "RouteTableID": ["rtb-abc00001","rtb-abc00002","rtb-abc00003"]} }'}}'
```


- Fetch the latest release using wget from [here](https://github.com/flannel-io/flannel/releases/download/v0.8.0/flannel-v0.8.0-linux-amd64.tar.gz)
- Run flannel daemon:

```
sudo ./flanneld --etcd-endpoints=http://127.0.0.1:2379
```

Next, create and connect to a clone of `demo-instance-1`.
Run flannel with the `--etcd-endpoints` flag set to the *internal* IP of the instance running etcd.

Confirm that the subnet route table has entries for the lease acquired by each of the subnets.

<br/>
<div class="row">
  <div class="col-lg-10 col-lg-offset-1 col-md-10 col-md-offset-1 col-sm-12 col-xs-12 co-m-screenshot">
    <a href="img/aws-routes.png" class="co-m-screenshot">
      <img src="img/aws-routes.png" alt="AWS Routes" />
    </a>
  </div>
</div>
<div class="caption">AWS Routes</div>

### Limitations

Keep in mind that the Amazon VPC [limits](http://docs.aws.amazon.com/AmazonVPC/latest/UserGuide/VPC_Appendix_Limits.html) the number of entries per route table to 50. If you require more routes, request a quota increase or simply switch to the VXLAN backend.
