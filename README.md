# ecs-service

Provides an ECS service - effectively a task that is expected to
run until an error occurs or a user terminates it (typically a
webserver or a database). This module's primary intent is to make
it easier to set up a load balanced service using an existing
[Application Load
Balancer](https://docs.aws.amazon.com/elasticloadbalancing/latest/application/introduction.html).
If using a load balancer the module will automatically create a
[listener rule](https://docs.aws.amazon.com/elasticloadbalancing/latest/application/listener-update-rules.html),
a [target group](https://docs.aws.amazon.com/elasticloadbalancing/latest/application/load-balancer-target-groups.html),
and [security groups](https://docs.aws.amazon.com/AmazonVPC/latest/UserGuide/VPC_SecurityGroups.html)
when in `awsvpc` mode.
In addition, the module will automatically create a task definition
if one is not supplied. This module does **not** support Amazon's
[Classic Load Balancer](https://docs.aws.amazon.com/elasticloadbalancing/latest/classic/introduction.html)
or
[Network Load Balancer](https://docs.aws.amazon.com/elasticloadbalancing/latest/network/introduction.html).

See the [ECS Services section in Amazon's ECS developer guide](https://docs.aws.amazon.com/AmazonECS/latest/developerguide/ecs_services.html).

Example Usage
-----------------

### The containers.json file

For an ECS service that does *not* use an existing task definition, the
set of containers that will comprise the service must be specified
in the `container_definition_file` argument. This file is named
`containers.json` by default, and must be a valid JSON document.

Please note that this example contains only a small subset of the
available parameters. For further information, see
`container_definition_file` argument in the [Task
definition](#task_definition) block.

```json
[
  {
    "name": "apache",
    "image": "httpd",
    "portMappings": [
      {
        "containerPort": 80
      }
    ]
  }
]

```


### Simple Fargate service on a public subnet
```hcl
module "service_name" {
  source = "git@github.com:cites-illinois/as-aws-modules//ecs-service"

  name = "service_name"

  load_balancer {
    name           = "load_balancer_name"
    port           = 443
    container_name = "main_container_name"
    container_port = 8080
    host_header    = "myservice.example.com"
  }

  alias {
    domain   = "example.com"
    hostname = "myservice"
  }

  network_configuration {
    assign_public_ip = "true"
    tier             = "public"
    vpc              = "my-vpc"
  }
}
```

### Simple Fargate service on a private subnet with a NAT gateway with service discovery
```hcl
module "service_name" {
  source = "git@github.com:cites-illinois/as-aws-modules//ecs-service"

  name  = "service_name"

  service_discovery {
    namespace_id = "ns-cxn6fqejoygbxan5"
  }

  network_configuration {
    tier = "nat"
    vpc  = "my-vpc"
  }
}
```


### Simple ECS service in bridge mode
```hcl
module "service_name" {
  source = "git@github.com:cites-illinois/as-aws-modules//ecs-service"

  name        = "service_name"
  launch_type = "EC2"

  load_balancer {
    name           = "load_balancer_name"
    port           = 443
    container_name = "main_container_name"
    container_port = 8080
    host_header    = "myservice.example.com"
  }

  alias {
    domain   = "example.com"
    hostname = "myservice"
  }

  task_definition {
    network_mode = "bridge"
  }
}
```

### ECS service using an externally defined task definition
```hcl
module "service_name" {
  source = "git@github.com:cites-illinois/as-aws-modules//ecs-service"

  name                = "service_name"
  cluster             = "cluster_name"
  launch_type         = "EC2"

  task_definition_arn = "task_definition_name:revision"
  desired_count       = 3

  load_balancer {
    name           = "load_balancer_name"
    port           = 443
    container_name = "main_container_name"
    container_port = 8080
    host_header    = "myservice.example.com"
  }

  alias {
    domain   = "example.com"
    hostname = "myservice"
  }

  ordered_placement_strategy = [
    {
      type  = "binpack"
      field = "cpu"
    }
  ]

  placement_constraints = [
    {
      type       = "memberOf"
      expression = "attribute:ecs.availability-zone in [us-west-2a, us-west-2b]"
    }
  ]
}
```

Argument Reference
-----------------

The following arguments are supported:

* `name` - (Required) The name of the service (up to 255 letters,
numbers, hyphens, and underscores).

* `task_definition` - (Optional) A [Task definition](#task_definition)
block. Task definition blocks are documented below.

* `task_definition_arn` - (Optional) The family and revision
(`family:revision`) or full ARN of the task definition that you
want to run in your service. If given, the task definition block is
ignored.

* `desired_count` - (Optional) The number of instances of the task
definition to place and keep running. Defaults to 1.

* `launch_type` - (Optional) The launch type on which to run your
service. The valid values are EC2 and FARGATE. Defaults to FARGATE.

* `cluster` - (Optional) ARN of an ECS cluster. Defaults to default.

* `deployment_maximum_percent` - (Optional) The upper limit (as a
percentage of the service's `desired_count`) of the number of running
tasks that can be running in a service during a deployment. Defaults to 200%.

* `deployment_minimum_healthy_percent` - (Optional) The lower limit
(as a percentage of the service's `desired_count`) of the number of
running tasks that must remain running and healthy in a service
during a deployment. Defaults to 50%.

* `ordered_placement_strategy` - (Optional) Service level strategy rules that
are taken into consideration during task placement. The maximum
number of `ordered_placement_strategy` blocks is 5. Defined [below](#ordered_placement_strategy).

* `health_check_grace_period_seconds` - (Optional) Seconds to ignore
failing load balancer health checks on newly instantiated tasks to
prevent premature shutdown, up to 1800. Only valid for services
configured to use a load balancer.

* `load_balancer` - (Optional) A [Load Balancer block](#load_balancer).
Load balancer blocks documented below.

* `placement_constraints` - (Optional) rules that are taken into
consideration during task placement. Maximum number of
`placement_constraints` is 10. Defined [below](#placement_constraints).

* `network_configuration` - (Optional) A
[Network Configuration](#network_configuration) block. This parameter
is required for task definitions that use the `awsvpc` network mode
to receive their own Elastic Network Interface, and it is not
supported for other network modes.

* `stickiness` - (Optional) A [Stickiness block](#stickiness).
Stickiness blocks are documented below.

* `health_check` -  (Optional) A [Health Check block](#health_check).
Health Check blocks are documented below.

* `volume` - (Optional) A set of [volume blocks](#volume) that
containers in your task may use. Volume blocks are documented below.

* `service_discovery` - (Optional) A [Service Discovery](#service_discovery) block.
This parameter is used to configure
[Amazon Service Discovery](https://docs.aws.amazon.com/AmazonECS/latest/developerguide/service-discovery.html)
for the service.

* `service_discovery_health_check_config` – (Optional) A [Service Discovery health check config](#service_discovery_health_check_config) block that contains settings for an
optional health check. Only for public DNS namespaces.

* `service_discovery_health_check_custom_config` – (Optional) A [Service Discovery
custom health check config](#service_discovery_health_check_custom_config) block that
contains settings for ECS managed health checks.

* `alias` – (Optional) An [alias](#alias) block used to define a Route 53 alias record
that points to the load balancer. Requires that a `load_balancer` block is definied.

`alias`
-------

An `alias` block supports the following:

* `domain` - (Required) The name of the Route 53 zone in which the alias record
is to be created.

* `hostname` – (Optional) The name of the host to be created in the specified Route
53 zone. Defaults to the value of the `name` attribute (i.e., the service name).


`load_balancer`
-----------------

A `load_balancer` block supports the following:

* `name` - (Required) The name of the LB.

* `port` - (Optional) The port of the listener. Defaults to 443.

* `container_name` - (Required) The name of the container to associate
with the load balancer as it appears in the container definition (by
default, `containers.json`).

* `container_port` - (Required) The port on the container to associate
with the load balancer as it appears in the container definition (by
default, `containers.json`).

* `certificate_domain` - (Optional) The domain name associated with an Amazon Certificate Manager (ACM) certificate. If specified, the certificate is looked up by the domain name, and the resulting certificate ARN is associated with the listener for the ECS service.

* `host_header` - (Required) A [hostname condition](https://docs.aws.amazon.com/elasticloadbalancing/latest/application/load-balancer-listeners.html#host-conditions)
that defines a rule to forward requests to the service's target group.

* `path_pattern` - (Optional) A [path condition](https://docs.aws.amazon.com/elasticloadbalancing/latest/application/load-balancer-listeners.html#path-conditions)
that defines a rule to forward requests based on the URL path.
Defaults to '*'.

* `deregistration_delay` - (Optional) The amount time for Elastic Load Balancing to wait before changing the state of a deregistering target from draining to unused. The range is 0-3600 seconds. The default value is 300 seconds.

* `security_group_id` - (Optional) The Security Group ARN associated with the Load Balancer. If not specified it is looked up by the `name` of the Load Balancer.

* `priority` - (Optional) The priority for the rule between 1 and 50000.
Leaving it unset will automatically set the rule with next
available priority after currently existing highest rule. A listener
can't have multiple rules with the same priority.

> Note: As a result of an AWS limitation, a single load_balancer
> can be attached to the ECS service at most. See [related docs](https://docs.aws.amazon.com/AmazonECS/latest/developerguide/service-load-balancing.html#load-balancing-concepts).

`stickiness`
-----------------

A `stickiness` block supports the following:

* type - (Required) The type of sticky sessions. The only current
possible value is `lb_cookie`.

* cookie_duration - (Optional) The time period, in seconds, during
which requests from a client should be routed to the same target.
After this time period expires, the load balancer-generated cookie
is considered stale. The range is 1 second to 1 week (604800 seconds).
The default value is 1 day (86400 seconds).

* enabled - (Optional) Boolean to enable / disable `stickiness`.
Default is `true`


`health_check`
-----------------

A `health_check` block supports the following:

* `interval` - (Optional) The approximate amount of time, in seconds,
between health checks of an individual target. Minimum value 5
seconds, Maximum value 300 seconds. Default 30 seconds.

* `path` - (Optional) The destination for the health check request. Default /.

* `port` - (Optional) The port to use to connect with the target.
Valid values are either ports 1-65536, or `container_port`. Defaults
to `container_port`.

* `protocol` - (Optional) The protocol to use to connect with the
target. Defaults to HTTP.

* `timeout` - (Optional) The amount of time, in seconds, during which
no response means a failed health check. The range is 2 to 60 seconds
and the default is 5 seconds.

* `healthy_threshold` - (Optional) The number of consecutive health
checks successes required before considering an unhealthy target
healthy. Defaults to 3.

* `unhealthy_threshold` - (Optional) The number of consecutive health
check failures required before considering the target unhealthy.
Defaults to 3.

* `matcher` - (Optional) The HTTP codes to use when checking for a
successful response from a target. You can specify multiple values
(for example, "200,202") or a range of values (for example, "200-299").

`ordered_placement_strategy`
---------------------------

`ordered_placement_strategy` blocks support the following:

* `type` - (Required) The type of placement strategy. Must be one
of: `binpack`, `random`, or `spread`

* `field` - (Optional) For the `spread` placement strategy, valid values
are `instanceId` (or `host`, which has the same effect), or any platform
or custom attribute that is applied to a container instance. For
the `binpack` type, valid values are `memory` and `cpu`. For the `random`
type, this attribute is not needed. For more information, see
[Placement Strategy](https://docs.aws.amazon.com/AmazonECS/latest/APIReference/API_PlacementStrategy.html).

> Note: `ordered_placement_strategy` is not supported when `launch_type` is FARGATE.

> Note: for `spread`, `host` and `instanceId` will be normalized, by AWS, to be `instanceId`. This means the statefile will show `instanceId` but your config will differ if you use `host`.

`placement_constraints`
-----------------------

`placement_constraints` blocks support the following:

* `type` - (Required) The type of constraint. The only valid values at
this time are `memberOf` and `distinctInstance`.

* `expression` - (Optional) Cluster Query Language expression to apply
to the constraint. Does not need to be specified for the `distinctInstance`
type. For more information, see [Cluster Query Language in the Amazon
EC2 Container Service Developer Guide.](https://docs.aws.amazon.com/AmazonECS/latest/developerguide/cluster-query-language.html)

> Note: `placement_constraints` is not supported when `launch_type` is FARGATE.

`network_configuration`
-----------------------

A `network_configuration` block supports the following:

* `tier` - (Optional) A subnet tier tag (e.g., public, private, nat) to determine subnets to be associated with the task orservice.

* `vpc` - (Optional) The name of the virtual private cloud to be associated with the task or
service. **NOTE:** Required when using `tier`.

* `subnets` - (Required) The subnet IDs to associated with the task or service. **NOTE:** Optional when using `tier`.

* `security_groups` - (Optional) The security groups associated with
the task or service. If you do not specify a security group, the
default security group for the VPC is used.

* `security_group_names` - (Optional) Additonal security groups to
associated with the task or service. This is a space delimited
string list of security group names. This may be specified with or
without `security_groups` in which case both lists are merged. If
you do not specify a security group, the default security group for
the VPC is used.

* `assign_public_ip` - (Optional) Assign a public IP address to the
ENI (Fargate launch type only). Valid values are `true` or `false`.
Default `false`.

* `ports` - (Optional) Ports to open on the container to outside
traffic. (For FARGATE testing use only.)

For more information, see [Task Networking](https://docs.aws.amazon.com/AmazonECS/latest/developerguide/task-networking.html).

> Note: `network_configuration` is only supported when `network_mode`
> is `aws_vpc`.
>
> Note: The `tier` and `subnet` attributes can be used together. In this case the subnets
> to be associated with the service consist of the union of the subnets defined explicitly
> in `subnet` and derived from `tier` and `vpc`.

`service_discovery`
-----------------

A `service_discovery` block configures [Amazon Service Discovery](https://docs.aws.amazon.com/AmazonECS/latest/developerguide/service-discovery.html) and supports the following:

* `name` – (Optional) The hostname of the service. Defaults to the service name specified
 by the `name` argument.

* `namespace_id` – (Required) The ID of the namespace to use for DNS configuration.

* `dns_routing_policy` – (Optional) The routing policy that you want to apply to all records that Route 53 creates when you register an instance and specify the service. Valid Values: MULTIVALUE, WEIGHTED.

* `dns_ttl` – (Optional) The amount of time, in seconds, that you want DNS resolvers to cache the settings for this resource record set. Default is 60 seconds.

* `dns_type` – (Optional) The type of the resource, which indicates the value that Amazon Route 53 returns in response to DNS queries. Valid Values: A, AAAA, SRV, CNAME. Default is "A".

`service_discovery_health_check_config`
---------------------------------------

The following arguments are supported:

* `failure_threshold` – (Optional) The number of consecutive health
checks. Maximum value of 10.

* `resource_path` – (Optional) The path that you want Route 53 to
request when performing health checks. Route 53 automatically adds
the DNS name for the service. If you don't specify a value, the
default value is /.

* `type` – (Optional) The type of health check that you want to
create, which indicates how Route 53 determines whether an endpoint
is healthy. Valid Values: HTTP, HTTPS, TCP

`service_discovery_health_check_custom_config`
----------------------------------------------

The following arguments are supported:

* `failure_threshold` – (Optional) The number of 30-second intervals that you want
service discovery to wait before it changes the health status of a service instance.
Maximum value of 10.

`task_definition`
-----------------

If a `task_definition_arn` is not given, a container definition will be created for the service. The name of the automatically created container definition is the same as the ECS service name.
The created container definition may optionally be further modified by specifying a `task_definition` block with one of more of the following options:

* `container_definition_file` - (Optional) A file containing a list of valid [container
definitions](https://docs.aws.amazon.com/AmazonECS/latest/developerguide/task_definition_parameters.html#container_definitions)
provided as a single JSON document. See
[Example Task Definitions](https://docs.aws.amazon.com/AmazonECS/latest/developerguide/example_task_definitions.html)
for example container definitions, note that _only_ the content of the `containerDefinitions` key
in these example task definitions belongs in the specified `container_definition_file`.
The default filename is `containers.json`.

* `task_role_arn` - (Optional) The ARN of an IAM role that allows
your Amazon ECS container task to make calls to other AWS services.

* `network_mode` - (Optional) The Docker networking mode to use for
the containers in the task. The valid values are `none`, `bridge`,
`awsvpc`, and `host`.

* `cpu` - (Optional) The number of
[cpu units](https://docs.aws.amazon.com/AmazonECS/latest/developerguide/task_definition_parameters.html#task_size)
used by the task.  Supported for FARGATE only, defaults to 256 (0.25 vCPU).

* `memory` - (Optional) The amount (in MiB) of
[memory](https://docs.aws.amazon.com/AmazonECS/latest/developerguide/task_definition_parameters.html#task_size)
used by the task. Supported for FARGATE only, defaults to 512.

`volume`
--------

A `volume` block supports the following:

* `name` - (Required) The name of the volume. This name is referenced
in the `sourceVolume` parameter of container definition in the
`mountPoints` section.

* `host_path` - (Optional) The path on the host container instance
that is presented to the container. If not set, ECS will create a
non persistent data volume that starts empty and is deleted after
the task has finished.

Attributes Reference
--------------------

The following attributes are exported:

* `id` - The Amazon Resource Name (ARN) that identifies the service

* `fqdn` – The fully qualified domain name of the Route 53 record for
the service. Only created when an `alias` block is provided.

* `target_group_arn` - The ARN of the Target Group created when a
`load_balancer` block is given

* `task_definition_arn` - Full ARN of the Task Definition created
for the service when a `task_definition_arn` is not given

* `security_group_id` - The ID of the security group created for
the service (`awsvpc` mode only)

* `subnet_ids` – A list of subnet IDs associated with the service.


Credits
--------------------

**Nota bene:** The vast majority of the verbiage on this page was
taken directly from the Terraform manual, and in a few cases from
Amazon's documentation.
