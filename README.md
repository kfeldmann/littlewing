# Terraflop - Infrastructure Automation

Terraflop is a simple infrastructure automation tool.

Terraflop supports only AWS. It leverages aws-cli,
and litterally can do anything that aws-cli can do.

## How does it work

To put it simply, Terraflop runs aws-cli commands that you specify
in your configuration, and stores the output (typically the output
contains all of the details of the resource created or referenced by
the command) so that subsequent commands can make use of that data.
For example, imagine you're using
aws-cli by itself. You create a VPC. You create an Internet Gateway. Next, to
attach the gateway to the VPC you will need the ID of each.
If you want to automate the setup of these items, then your _automation_
needs to be able to remember these IDs. Terraflop is designed to
do exactly that.

Here is what that exercise looks like in Terraflop:

```YAML
- aws.ec2.create-vpc.main:
  - --cidr-block
  - 10.0.0.0/16

- aws.ec2.create-internet-gateway.main: null

- aws.ec2.attach-internet-gateway.main:
  - --internet-gateway-id
  - ${aws.ec2.create-internet-gateway.main.InternetGateway.InternetGatewayId}
  - --vpc-id
  - ${aws.ec2.create-vpc.main.Vpc.VpcId}
```

With this basic capability, quite a lot can be achieved. You can
use variables (even nested variables) to dynamically name resources,
and use aws-cli's full functionality to query, modify, or execute actions
in addition to creating resources.

## Installation

- Install and configure aws-cli per Amazon's instructions
- Install python 2.7
- Install pyyaml
- Clone or download Terraflop
- Put the `flop` command in your path
   - Copy, or symlink it into a directory that is already
     in your path, or...
   - Extend your path to cover the directory where `flop` resides

## Basics

Terraflop has just one command: `flop`. Flop reads all .yml files found
in the current directory and reads them in alphabetical order. Each
file contains variables, steps, or both. After loading all of the
variables in a file, the steps in that file are executed in the
order listed.

Terraflop does not figure out ordering for you. To control the
order of operations, pay attention to the order of your steps and
files. It is common to name files with a numeric prefix so that
they sort unambiguously (see `example-apps`).

```
10_app.yml
20_web-tier.yml
30_database-tier.yml
```

The configuration files use YAML (.yml) format. YAML is a
standard format documented elsewhere.

In Terraflop, the commands to be run are called steps. Steps are
constructed using the following structure:

```YAML
steps:
- aws.CATEGORY.COMMAND.NAME:
  - ARG
  - ...
```

where CATEGORY is the command category in aws-cli, such as:
`ec2`, `iam`, `autoscaling`, etc.

COMMAND is the aws-cli command, such as: `create-vpc`, etc.

NAME is a unique name for this instance of this command. This will
allow you to distinguish between multiple uses of the same command.
For example, if you are creating three subnets, you will call
create-subnet three times. Provide three names to distinguish them.
This is also how you will distinguish the subnets when you refer
to them in subsequent steps.

The arguments (ARG, ...) are exactly what you would pass to aws-cli for
the same command. If a given command requires an argument of `--foo`
and it's value of `bar`, those two separate strings should be listed in that
order under the command.

There should be at most one `steps:` key in each file. This `steps:` object
is a list that contains all of the steps for that file.

Example:

```YAML
steps:
- aws.ec2.describe-network-acls.default-acl:
  - --filters
  - Name=default,Values=true
```

For each step, Terraflop first checks the output directory
(`.terraflop-output` in the current working directory) to see if
the step has been run already. If it has run already, it will be
skipped. Inside each of the output files you will see the attributes
that can be referenced from subsequent commands.

Variables can be used throughout a terraflop configuration. There
are two types of variables: strings and maps.

String variables are defined using this syntax:

```
var.NAME: VALUE
```

Example:

```YAML
var.region: us-west-2
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
${var.NAME}
```

To dereference one item in a map variable, use:

```
${var.NAME.KEY}
```

To dereference an attribute from the output of a previous command,
use the following syntax:

```
${aws.CATEGORY.COMMAND.NAME.STRUCTURE}
```

where STRUCTURE is the structure from the aws-returned json, converted
to a string using dots to delineate levels of heirarchy, and digits
to indicate elements of lists.

Examples:

```
${aws.ec2.create-vpc.main.Vpc.VpcId}
${aws.ec2.describe-images.dev-website-web.Images.0.ImageId}
${aws.ec2.describe-images.dev-website-web.Images.0.BlockDeviceMappings.0.DeviceName}
```

Variable dereferencing can be nested, such as:

```
${aws.ec2.create-vpc.${var.environment}.Vpc.VpcId}
```

The inner-most (and left-most) variable will be dereferenced first,
working outwards (and right-wards) until the string is variable-free.
This nesting enables parameterized configurations that are easily cloned
and/or reused.

You can run `flop` on some initial commands and view the output
to copy the references to use in subsequent commands. When you re-run
`flop`, the previous commands that have already run will be skipped.
In this way, your work can be very iterative.

Variables and output are all global. There is no local scope.

There is no special concept of modules in Terraflop, but you can easily
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

At least once per application configuration, usually in one of the first files,
you should set the profile and region to be used by aws cli. This is done with
the keywords `profile:` and `region:`. The values can be set using either
variables or strings. It is also possible to change profiles throughout your
configuration, for example to accept a peering connection request in another
account.

To make changes in a production setting, take a look at the strategy
used in the blue/green pools of the sample app's web tier. By versioning
the names of steps that _update_ infrastructure, you can change the config,
bump the version, and then `flop` will see those as new steps which have
not yet run. As a side effect, you will have the output of each of your
previous versions acumulating in the outputs directory.
