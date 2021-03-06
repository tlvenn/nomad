---
layout: "docs"
page_title: "Interpreted Variables"
sidebar_current: "docs-jobspec-interpreted"
description: |-
  Learn about the Nomad's interpreted variables.
---
# Interpreted Variables

Nomad support interpreting two classes of variables, node attributes and runtime
environment variables. Node attributes are interpretable in constraints, task
environment variables and certain driver fields. Runtime environment variables
are not interpretable in constraints because they are only defined once the
scheduler has placed them on a particular node.

The syntax for interpreting variables is `${variable}`. An example and a
comprehensive list of interpretable fields can be seen below:

```hcl
task "docs" {
  driver = "docker"

  # Drivers support interpreting node attributes and runtime environment
  # variables
  config {
    image = "my-app"

    # Interpret runtime variables to inject the address to bind to and the
    # location to write logs to.
    args = [
      "--bind", "${NOMAD_ADDR_RPC}",
      "--logs", "${NOMAD_ALLOC_DIR}/logs",
    ]

    port_map {
      RPC = 6379
    }
  }

  # Constraints only support node attributes as runtime environment variables
  # are only defined after the task is placed on a node.
  constraint {
    attribute = "${attr.kernel.name}"
    value     = "linux"
  }

  # Environment variables are interpreted and can contain both runtime and
  # node attributes. There environment variables are passed into the task.
  env {
    "DC"      = "Running on datacenter ${node.datacenter}"
    "VERSION" = "Version ${NOMAD_META_VERSION}"
  }

  # Meta keys are also interpretable.
  meta {
    VERSION = "v0.3"
  }
}
```

## Node Variables <a id="interpreted_node_vars"></a>

Below is a full listing of node attributes that are interpretable. These
attributes are interpreted by __both__ constraints and within the task and
driver.

<table class="table table-bordered table-striped">
  <tr>
    <th>Variable</th>
    <th>Description</th>
    <th>Example Value</th>
  </tr>
  <tr>
    <td><tt>${node.unique.id}</tt></td>
    <td>36 character unique client identifier</td>
    <td><tt>9afa5da1-8f39-25a2-48dc-ba31fd7c0023</tt></td>
  </tr>
  <tr>
    <td><tt>${node.datacenter}</tt></td>
    <td>Client's datacenter</td>
    <td><tt>dc1</tt></td>
  </tr>
  <tr>
    <td><tt>${node.unique.name}</tt></td>
    <td>Client's name</td>
    <td><tt>nomad-client-10-1-2-4</tt></td>
  </tr>
  <tr>
    <td><tt>${node.class}</tt></td>
    <td>Client's class</td>
    <td><tt>linux-64bit</tt></td>
  </tr>
  <tr>
    <td><tt>${attr.&lt;property&gt;}</tt></td>
    <td>Property given by <tt>property</tt> on the client</td>
    <td><tt>${attr.arch} => amd64</tt></td>
  </tr>
  <tr>
    <td><tt>${meta.&lt;key&gt;}</tt></td>
    <td>Metadata value given by <tt>key</tt> on the client</td>
    <td><tt>${meta.foo} => bar</tt></td>
  </tr>
</table>

Below is a table documenting common node properties:

<table class="table table-bordered table-striped">
  <tr>
    <th>Property</th>
    <th>Description</th>
  </tr>
  <tr>
    <td><tt>arch</tt></td>
    <td>CPU architecture of the client (e.g. <tt>amd64</tt>, <tt>386</tt>)</td>
  </tr>
  <tr>
    <td><tt>consul.datacenter</tt></td>
    <td>The Consul datacenter of the client (if Consul is found)</td>
  </tr>
  <tr>
    <td><tt>cpu.numcores</tt></td>
    <td>Number of CPU cores on the client</td>
  </tr>
  <tr>
    <td><tt>driver.&lt;property&gt;</tt></td>
    <td>See the [task drivers](/docs/drivers/index.html) for property documentation</td>
  </tr>
  <tr>
    <td><tt>unique.hostname</tt></td>
    <td>Hostname of the client</td>
  </tr>
  <tr>
    <td><tt>kernel.name</tt></td>
    <td>Kernel of the client (e.g. <tt>linux</tt>, <tt>darwin</tt>)</td>
  </tr>
  <tr>
    <td><tt>kernel.version</tt></td>
    <td>Version of the client kernel (e.g. <tt>3.19.0-25-generic</tt>, <tt>15.0.0</tt>)</td>
  </tr>
  <tr>
    <td><tt>platform.aws.ami-id</tt></td>
    <td>AMI ID of the client (if on AWS EC2)</td>
  </tr>
  <tr>
    <td><tt>platform.aws.instance-type</tt></td>
    <td>Instance type of the client (if on AWS EC2)</td>
  </tr>
  <tr>
    <td><tt>os.name</tt></td>
    <td>Operating system of the client (e.g. <tt>ubuntu</tt>, <tt>windows</tt>, <tt>darwin</tt>)</td>
  </tr>
  <tr>
    <td><tt>os.version</tt></td>
    <td>Version of the client OS</td>
  </tr>
</table>

Here are some examples of using node attributes and properties in a job file:

```hcl
job "docs" {
  # This will constrain this job to only run on 64-bit clients.
  constraint {
    attribute = "${attr.arch}"
    value     = "amd64"
  }

  # This will restrict the job to only run on clients with 4 or more cores.
  # Note: you may also declare a resource requirement for CPU for a task.
  constraint {
    attribute = "${cpu.numcores}"
    operator  = ">="
    value     = "4"
  }

  # Only run this job on a memory-optimized AWS EC2 instance.
  constraint {
    attribute = "${attr.platform.aws.instance-type}"
    value     = "m4.xlarge"
  }
}
```

## Environment Variables <a id="interpreted_env_vars"></a>

The following are runtime environment variables that describe the environment
the task is running in. These are only defined once the task has been placed on
a particular node and as such can not be used in constraints.

<table class="table table-bordered table-striped">
  <tr>
    <th>Variable</th>
    <th>Description</th>
  </tr>
  <tr>
    <td><tt>${NOMAD_ALLOC_DIR}</tt></td>
    <td>The path to the shared <tt>alloc/</tt> directory. See [here](/docs/jobspec/environment.html#task_dir) for more information.</td>
  </tr>
  <tr>
    <td><tt>${NOMAD_TASK_DIR}</tt></td>
    <td>The path to the task <tt>local/</tt> directory. See [here](/docs/jobspec/environment.html#task_dir) for more information.</td>
  </tr>
  <tr>
    <td><tt>${NOMAD_MEMORY_LIMIT}</tt></td>
    <td>The memory limit in MBytes for the task</td>
  </tr>
  <tr>
    <td><tt>${NOMAD_CPU_LIMIT}</tt></td>
    <td>The CPU limit in MHz for the task</td>
  </tr>
  <tr>
    <td><tt>${NOMAD_ALLOC_ID}</tt></td>
    <td>The allocation ID of the task</td>
  </tr>
  <tr>
    <td><tt>${NOMAD_ALLOC_NAME}</tt></td>
    <td>The allocation name of the task</td>
  </tr>
  <tr>
    <td><tt>${NOMAD_ALLOC_INDEX}</tt></td>
    <td>The allocation index; useful to distinguish instances of task groups</td>
  </tr>
  <tr>
    <td><tt>${NOMAD_TASK_NAME}</tt></td>
    <td>The task's name</td>
  </tr>
  <tr>
    <td><tt>${NOMAD_IP_&lt;label&gt;}</tt></td>
    <td>The IP for the given port <tt>label</tt>. See
    [here](/docs/jobspec/networking.html) for more information.</td>
  </tr>
  <tr>
    <td><tt>${NOMAD_PORT_&lt;label&gt;}</tt></td>
    <td>The port for the port <tt>label</tt>. See [here](/docs/jobspec/networking.html) for more information.</td>
  </tr>
  <tr>
    <td><tt>${NOMAD_ADDR_&lt;label&gt;}</tt></td>
    <td>The <tt>ip:port</tt> pair for the given port <tt>label</tt>. See
    [here](/docs/jobspec/networking.html) for more information.</td>
  </tr>
  <tr>
    <td><tt>${NOMAD_HOST_PORT_&lt;label&gt;}</tt></td>
    <td>The port on the host if port forwarding is being used for the port
    <tt>label</tt>. See [here](/docs/jobspec/networking.html#mapped_ports) for more
    information.</td>
  </tr>
  <tr>
    <td><tt>${NOMAD_META_&lt;key&gt;}</tt></td>
    <td>The metadata value given by <tt>key</tt> on the task's metadata</td>
  </tr>
  <tr>
    <td><tt>${"env_key"}</tt></td>
    <td>Interpret an environment variable with key <tt>env_key</tt> set on the task.</td>
  </tr>
</table>
