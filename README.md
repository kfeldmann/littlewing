# Terraflop - Infrastructure Automation

Terraflop is a simple infrastructure automation tool. It can be used
to create and manage version controlled infrastructure as if it were
software.

Terraflop supports only AWS at this point. It leverages aws-cli,
and it litterally can do anything that aws-cli can do.

## How does it work

To put it simply, Terraflop runs aws-cli commands that you specify
in your configuration, and stores the output so that subsequent
commands can make use of it.

That's it.

But with this capability, quite a lot can be achieved. You can
use variables (even nested variables) to dynamically name resources,
and use aws-cli's full functionality to query, modify, or execute actions
in addition to creating resources.

## Installation

- Install and configure aws-cli per Amazon's instructions
- Install python 2.7
- Install pyyaml
- Clone Terraflop
- Put the 'flop' command in your path
   - Move, copy, or symlink it into a directory that is already
     in your path, or...
   - Extend your path to cover the directory where 'flop' resides

## Basics

Terraflop has just one command: flop. Flop reads all .yml files found
in the current directory and reads them in alphabetical order. Each
file contains variables, steps, or both. After loading all of the
variables in a file, the steps in that file are executed in the
order listed.

Terraflop does not figure out ordering for you. To control the
order of operations, pay attention to the order of your steps and
files. It is common to name files with a numeric prefix so that
they sort unambiguously (see example-apps).

    10_app.yml
    20_web-tier.yml
    30_database-tier.yml

For each step, Terraflop first checks the output directory
(.terraflop-output in the current working directory) to see if
the step has been run already. If it has run already, it will be
skipped. Inside each of the output files you will see the attributes
that can be referenced from subsequent commands.

Variables can be used throughout a terraflop configuration. There
are two types of variables: strings and maps.

String variables are defined using this syntax:

    var.NAME: VALUE

Example:

    var.region: us-west-2

Map variables are defined using this syntax:

    var.NAME:
      KEY: VALUE
      ...

Example:

    var.subnets:
      dev-website-web-a: 10.0.0.0/24
      dev-website-web-b: 10.0.1.0/24

To dereference a string variable, use:

    ${var.NAME}

To dereference one item in a map variable, use:

    ${var.NAME.KEY}

To dereference an attribute from the output of a previous command,
use the following syntax:

    ${aws.CATEGORY.COMMAND.NAME.STRUCTURE}

where STRUCTURE is the structure from the aws-returned json, converted
to a string using dots to delineate levels of heirarchy, and digits
to indicate elements of lists.

Examples:

    ${aws.ec2.create-vpc.main.Vpc.VpcId}
    ${aws.ec2.describe-images.dev-website-web.Images.0.ImageId}
    ${aws.ec2.describe-images.dev-website-web.Images.0.BlockDeviceMappings.0.DeviceName}

Variable dereferencing can be nested, such as:

    ${aws.ec2.create-vpc.${var.environment}.Vpc.VpcId}

The inner-most (and left-most) variable will be dereferenced first,
working outwards (and right-wards) until the string is variable-free.
This nesting enables parameterized configurations that are easily cloned
and/or reused.

You can run 'flop' on some initial commands and view the output
to copy the references to use in subsequent commands. When you re-run
'flop', the previous commands that have already run will be skipped.
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

To make changes in a production setting, take a look at the strategy
used in the blue/green pools of the sample app's web tier. By versioning
the names of steps that _update_ infrastructure, you can change the config,
bump the version, and then 'flop' will see those as new steps which have
not yet run. As a side effect, you will have the output of each of your
previous versions acumulating in the outputs directory.
