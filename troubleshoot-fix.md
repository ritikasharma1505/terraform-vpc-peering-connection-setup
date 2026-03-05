### VPC Peering Routing Issue and Fix


### Problem Observed

During the initial Terraform deployment of the cross-region VPC peering setup, the VPC peering connection was successfully created and accepted, but the route entries required for inter-VPC communication were not consistently appearing in the route tables.

Example expected route:

```
Destination: 10.0.0.0/16
Target: pcx-xxxxxxxx
```

However, the route table only showed:

```
10.1.0.0/16 → local
0.0.0.0/0 → igw
```

Even though Terraform state indicated the route existed.

**Root Cause**

This issue was caused by AWS eventual consistency during cross-region VPC peering creation.

The sequence of events inside AWS looks like this:

1. Terraform creates VPC Peering Request
2. Terraform accepts peering from secondary region
3. AWS marks peering as "active"
4. AWS internally propagates routing information
5. Peering becomes fully usable
6. Routes can be safely attached

Terraform originally attempted to create routes immediately after the peering connection was accepted.

However, AWS sometimes requires a few seconds for internal propagation, meaning the connection might appear active but is not fully ready for route association.

As a result:

• Terraform created the route resource
• AWS had not finished propagating the peering
• Route creation silently failed or became inconsistent
• Terraform state showed the route as active, but the AWS console did not

This is a known eventual consistency behavior in AWS networking APIs.

Initial Attempt

Initially the route depended only on the peering accepter:

```
depends_on = [
  aws_vpc_peering_connection_accepter.secondary_accepter
]
```
This ensured that:

Peering request -> Peering acceptance -> Route creation

However, this was not sufficient because the peering connection might still be propagating internally after acceptance.

Final Solution Implemented

The routes were updated to depend on DNS configuration resources instead:

```
depends_on = [
  aws_vpc_peering_connection_options.primary_dns,
  aws_vpc_peering_connection_options.secondary_dns
]
```

This changes the dependency chain to:

1. Create VPC peering request
2. Accept peering connection
3. Configure peering DNS options
4. Create route entries

Why This Works ?

The DNS option configuration requires the peering connection to be fully active before it can be applied.

AWS enforces this restriction:

Peering options can be added only to active peerings

Because of this requirement:

Terraform is forced to wait until the peering connection is fully active and propagated before configuring DNS options.

Once DNS configuration succeeds, the connection is guaranteed to be stable, and route creation becomes reliable.

Final Execution Order

Terraform now builds the infrastructure in the following sequence:

```
VPC Creation
      ↓
Subnet Creation
      ↓
Internet Gateway
      ↓
Route Tables
      ↓
VPC Peering Request
      ↓
VPC Peering Acceptance
      ↓
DNS Resolution Configuration
      ↓
Route Creation
      ↓
EC2 Instance Deployment
```

This ordering ensures that route creation occurs only after the peering connection is completely ready.

Key Improvement

Instead of using artificial delays like:

time_sleep

this solution relies on Terraform dependency graph ordering, which is the recommended and deterministic approach.

Advantages:

• No arbitrary wait times
• Faster Terraform runs
• Deterministic resource ordering
• More production-ready infrastructure code

Result

After implementing this dependency structure and recreating the infrastructure:

• VPC peering connections were established successfully
• Route table entries were created reliably
• Cross-region private connectivity between VPCs worked as expected

Instances in one VPC were able to communicate with instances in the peered VPC using private IP addresses.