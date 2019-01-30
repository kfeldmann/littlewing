# Littlewing - Infrastructure Automation

Littlewing is a simple infrastructure automation tool.

Littlewing supports only AWS. It leverages
[aws-cli](https://aws.amazon.com/cli/), and litterally can do
anything that aws-cli can do.

## How it works

To put it simply, Littlewing runs aws-cli commands that you specify
in your configuration. It stores the output of those commands so
that subsequent commands
can make use of that data. The output of each command typically contains
all of the attributes of the resource created or referenced by that command.

Imagine you're using
aws-cli by itself. You create a VPC. You create an Internet Gateway. Next,
to attach the gateway to the VPC you will need the ID of each. You scroll
up in your terminal looking for those values in the output of your
previous commands...

If you want to _automate_ the setup of these items, then it is your _software_
that needs to be able to look up these IDs. Littlewing is designed to
do exactly that.

Here is what the above exercise looks like in Littlewing:

```YAML
- aws.ec2.create-vpc.main:
  - --cidr-block
  - 10.0.0.0/16

- aws.ec2.create-internet-gateway.main: null

- aws.ec2.attach-internet-gateway.main:
  - --internet-gateway-id
  - ~{aws.ec2.create-internet-gateway.main.InternetGateway.InternetGatewayId}
  - --vpc-id
  - ~{aws.ec2.create-vpc.main.Vpc.VpcId}
```

With this basic capability, quite a lot can be achieved. You can
use variables (even nested variables) to dynamically name resources,
and use aws-cli's full functionality to query, modify, or execute actions
in addition to creating resources.

## Installation

- [Install and configure aws-cli per Amazon's instructions](https://docs.aws.amazon.com/cli/latest/userguide/cli-chap-install.html)
- [Install pyyaml](https://pyyaml.org/wiki/PyYAMLDocumentation)
- Clone or download Littlewing
- Put the `lw` command in your path
   - Copy or symlink it into a directory that is already
     in your path, or...
   - Extend your path to cover the directory where `lw` resides

## Basics

Littlewing has just one command: `lw`. Lw reads all .yml files found
in the current directory and reads them in alphabetical order. Each
file contains variables, steps, or both. After loading all of the
variables in a file, the steps in that file are executed in the
order listed.

Littlewing does not figure out ordering for you. To control the
order of operations, pay attention to the order of your steps and
files. It is common to name files with a numeric prefix so that
they sort unambiguously (see `example-apps`).

```
10_meta.yml
20_web-tier.yml
30_database-tier.yml
```

The configuration files use YAML 1.1 (.yml) format. [YAML is a
standard format documented here](http://yaml.org/spec/1.1/).

In Littlewing, the commands to be run are called steps. Steps are
constructed using the following structure:

```YAML
steps[.TARGET]:
- PROVIDER.[AWS_CATEGORY.]COMMAND.NAME:
  - ARG
  - ...
```

TARGET is an optional label for the step list. If `lw` is then run
with a target string as its argument, only steps matching that target
will be examined.

PROVIDER should be either `aws` or `exec`. Steps using the `aws`
provider will be run using aws-cli, and those using the `exec`
provider will be run by spawning a subprocess in your local
environment (not on any remote instance).

AWS_CATEGORY is the command category in aws-cli, such as:
`ec2`, `iam`, `autoscaling`, etc. This is relevant only for the
`aws` provider, and should be left out for steps using the
`exec` provider.

COMMAND is the command to run, such as `create-vpc` for
aws, or any local command, shell script, etc. for the `exec`
provider.

NAME is a unique name for this instance of this command. This will
allow you to distinguish between multiple uses of the same command.
For example, if you are creating three subnets, you will call
create-subnet three times. Providing three names will allow you to
distinguish the subnets when you refer to them in subsequent steps.
All step keys (PROVIDER.[AWS_CATEGORY.]COMMAND.NAME) must be unique
throughout your project (not simply unique within the file).

The arguments (ARG, ...) are exactly what you would pass to aws-cli for
the same command. If a given command requires an argument of `--filters`
and it's value of `Name=default,Values=true`, then those two separate
strings should be listed in that order under the command.

There should be at most one `steps[.TARGET]:` key for each unique
TARGET in each file. This `steps[.TARGET]`
object is a list that contains all of the steps for that TARGET in
that file.

Example:

```YAML
steps.build:
- aws.ec2.describe-network-acls.default-acl:
  - --filters
  - Name=default,Values=true
```

A step can also consist of the key `print`, with any string
(with or without variables and/or output references) as its
value.

Example:

```YAML
- print: "URL: http://~{aws.elb.create-load-balancer.dev-web-0.DNSName}/"
```

For each step, Littlewing first checks the output directory
(`.lw-output` in the current working directory) to see if
the step has been run already. If it has run already, it will be
skipped. Inside each of the output files you will see the attributes
that can be referenced from subsequent commands.

To force a step to run every time, regardless whether it already
has run in the past, prepend the key with a `+` character.

Print statements, variable assignments, and everything outside of
any step list will always run every time.

You can run `lw` on some initial commands and view the output
to copy the references to use in subsequent commands. When you re-run
`lw`, the previous commands that have already run will be skipped.
In this way, your work can be very iterative.

## Variables

Variables can be used throughout a Littlewing configuration. There
are two types of variables: strings and maps.

String variables are defined using this syntax:

```
var.NAME: VALUE
```

Example:

```YAML
var.appname: my-app-name
```

Map variables are defined using this syntax:

```
var.NAME:
  KEY: VALUE
  ...
```

Example:

```YAML
var.subnets:
  dev-website-web-a: 10.0.0.0/24
  dev-website-web-b: 10.0.1.0/24
```

To dereference a string variable, use:

```
~{var.NAME}
```

To dereference one item in a map variable, use:

```
~{var.NAME.KEY}
```

## Resource Attributes

To dereference an attribute from the output of a previous command,
use the following syntax:

```
~{aws.CATEGORY.COMMAND.NAME.STRUCTURE}
```

where STRUCTURE is the structure from the aws-returned json, converted
to a string using dots to delineate levels of heirarchy, and digits
to indicate elements of lists.

Examples:

```
~{aws.ec2.create-vpc.main.Vpc.VpcId}
~{aws.ec2.describe-images.dev-website-web.Images.0.ImageId}
~{aws.ec2.describe-images.dev-website-web.Images.0.BlockDeviceMappings.0.DeviceName}
```

Variable dereferencing can be nested, such as:

```
~{aws.ec2.create-vpc.~{var.environment}.Vpc.VpcId}
```

The inner-most (and left-most) variable will be dereferenced first,
working outwards (and right-wards) until the string is variable-free.
This nesting enables parameterized configurations that are easily cloned
and/or reused.

Variables and output attributes are all global. There is no local scope.

## Loops

Imagine that you want to update all route tables in a certain project,
adding a new route to each one. You might not know up front how many
route tables there are (or maybe you want to make a re-usable module
that can be applied to multiple projects). You can use a `describe`
command to produce a listing of the appropriate route tables, and
then loop over that list to apply the new route.

Loops in Littlewing are created using the following syntax within a
step list:

```YAML
- loop: COUNT
- <other steps...>
- end-loop:
```

Inside the loop, the variable `LOOP_INDEX` will be automatically incremented
for each cycle, starting at zero. In nested loops, the index of parent loops
is stored as `LOOP_INDEX_1` for the parent, `LOOP_INDEX_2` for the grandparent,
etc. (up to `LOOP_INDEX_9` for a nesting depth of 10).

In our example above, you could refer to each route table by using the
`LOOP_INDEX` variable in the position of the list index of the route table
listing. For example:

```
aws.ec2.describe-route-tables.a.RouteTables.~{var.LOOP_INDEX}.RouteTableId
```

Littlewing stores the number of items in each output list using a custom `Count`
attribute. For example:

```
aws.ec2.describe-route-tables.a.RouteTables.Count
```

Loops can be nested.

## Modules

There is no special concept of modules in Littlewing, but you can easily
reuse parameterized configurations (see example-apps). The procedure is
to create a reusable configuration snippet, place it in a commmon directory,
and then symlink it into your application's configuration directory with
the name of the symlink controlling the sorting of that file within your app.

Typically you will define variables in an application-specific file, then link
a "module" file after it to execute steps, parameterized by the variables set
ahead of time.

For example, in the modules directory you can find some sample modules that
create app-level networking pieces, and tier-level networking pieces. Notice
how the sample app defines variables for each in fairly small files, then
symlinks in the larger module files to do the heavy lifting.

## Miscellaneous

At least once per application configuration, usually in one of the first files,
you should set the profile and region to be used by aws cli. This is done with
the keywords `profile:` and `region:`. The values can be set using either
variables or strings. It is also possible to change profiles throughout your
configuration, for example to accept a peering connection request in another
account.

To make changes in a production setting, take a look at the strategy
used in the blue/green pools of the 'test-site' example app's web tier.
By versioning the names of steps that _update_ infrastructure, you can
change the config,
bump the version, and then `lw` will see those as new steps which have
not yet run. As a side effect, you will have the output of each of your
previous versions acumulating in the outputs directory.
