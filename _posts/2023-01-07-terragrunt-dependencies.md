---
layout: post
show_meta: true
title: Using Terragrunt Dependencies
header: How to use Terragrunt dependencies
date: 2023-01-17 00:00:00
summary: Using Terragrunt Dependencies and its caveats
categories: terraform terragrunt devops
author: Chee Yeo
---

[Terragrunt]: https://terragrunt.gruntwork.io/
[Terragrunt Mock Outputs]: https://terragrunt.gruntwork.io/docs/features/execute-terraform-commands-on-multiple-modules-at-once/#unapplied-dependency-and-mock-outputs
[Terragrunt dependency config]: https://terragrunt.gruntwork.io/docs/reference/config-blocks-and-attributes/#dependency
[Terragrunt dependencies config]: https://terragrunt.gruntwork.io/docs/reference/config-blocks-and-attributes/#dependencies

Terragrunt is a powerful tool to organize and deploy your terraform modules. Rather than writing custom scripts or manually deploying your entire stack of modules by hand, terragrunt allows you to build virtual `stacks` of your infra via the use of `terragrunt.hcl` files which uses the same HCL language as used by terraform

During runtime, terragrunt translates these `terragrunt.hcl` configs into actual terraform files in temp dir `.terragrunt-cache` and delegates to terraform.

One of the more powerful features I find while using `terragrunt` is the ability to define virtual stacks of your infra using [Terragrunt dependency config] and [Terragrunt dependencies config]. 

This allows you to define explicit ordering on the order you want the modules to be applied. A side effect of using this functionality is for modules to pass data downwards as outputs from one module into the next module as its inputs. Normally, this works as expected if the entire `stack` has been applied at the same time. If one of the modules failed during initial apply this may lead to hard to debug errors and unexpected results.

Assuming we have the following structure of modules which must be run in the following sequence:

{% highlight shell linenos %}

stack
├── terragrunt.hcl
│
├── module_a
│   └── terragrunt.hcl
│
├── module_b
│   └── terragrunt.hcl
│
└── module_c
    └── terragrunt.hcl

{% endhighlight %}

There is a dependency of the following order: `A -> B -> C`. Both Module B and Module C relies on certain outputs from Module A. A common pattern is for Module B to use the outputs from Module A as inputs. These inputs are then defined as outputs in Module B, which gets passed into Module C via terragrunt.hcl


The `terragrunt.hcl` file in Module B would have a format such as:
{% highlight terraform linenos %}
# Module B terragrunt.hcl

dependency "module_a" {
  config_path = "../module_a"

  mock_outputs = {
    output_id = "fake-id"
  }
}

inputs {
  input_id = dependency.module_a.outputs.output_id
}

# Module B variables.tf
variable "input_id" {}

# Module B outputs.tf
# Passing the input values as outputs

output "output_id" {
  value = var.input_id
}
{% endhighlight %}

The above declares a dependency on Module A via the `config_path` keyword. The `mock_outputs` declare fake/mock values for module_a outputs if it has not been applied yet which gets passed to `Module B` as an input to its `input_id` variable. This same value then gets passed as an output from Module B as `output_id`.

Module C has a similar format:
{% highlight terraform linenos %}
# Module C terragrunt.hcl

dependency "module_b" {
  config_path = "../module_b"

  mock_outputs = {
    output_id = "fake-id"
  }
}

inputs {
  input_id = dependency.module_b.outputs.output_id
}

# Module C variables.tf

variable "input_id" {}
{% endhighlight %}

Assuming we run `terragrunt` and only Module A gets deployed and persisted to state.

If we re-run it again, one would expect the value of `dependency.module_b.outputs.output_id` to be the actual output from `dependency.module_a.outputs.output_id`.

Instead we get the mock value `fake-id` as Module B has not been applied and hence has no state so its mock value is returned instead. This results in the mock value being passed downstream to Module C as an input value based on its terragrunt.hcl config, leading to difficult to diagnose errors.

In other words, when we use `dependency` config, **if no state exists, the mock values are returned else it fetches and returns the real values from its state.**

From the [Terragrunt Mock Outputs]:
>
Terragrunt will return an error indicating the dependency hasn’t been applied yet if the terraform module managed by the terragrunt config referenced in a dependency block has not been applied yet. This is because you cannot actually fetch outputs out of an unapplied Terraform module, even if there are no resources being created in the module.

One way to break this dependency chain is to refactor both modules B and C so they can both run in parallel and inherit from Module A:
{% highlight terraform linenos %}
# Module B terragrunt.hcl

dependency "module_a" {
  config_path = "../module_a"

  mock_outputs = {
    output_id = "fake-id"
  }

  mock_outputs_merge_strategy_with_state = "shallow"
}

inputs {
  input1 = dependency.module_a.outputs.output_id
}


# Module C terragrunt.hcl

dependency "module_a" {
  config_path = "../module_a"

  mock_outputs = {
    output_id = "fake-id"
  }

  mock_outputs_merge_strategy_with_state = "shallow"
}

dependencies {
  paths = ["../module_b"]
}

inputs {
  input1 = dependency.module_a.outputs.output_id
}
{% endhighlight %}

Here we use `dependencies` block in Module C so it has to wait until after Module B is applied, maintaining the sequence. 

According to the [Terragrunt dependencies config]:
>
The dependencies block is used to enumerate all the Terragrunt modules that need to be applied in order for this module to be able to apply. Note that this is purely for ordering the operations when using run-all commands of Terraform. This does not expose or pull in the outputs like dependency blocks.

Both modules now inherit from Module A which results in either the actual / mock values being passed to it rather than ambiguous intermediate output values. We are also able to maintain the sequence between Module B and Module C.

The following are what I learnt the following whilst working with terragrunt dependencies:

* Keep dependencies to 1 level deep and pass outputs directly between modules without going through any intermediate modules.

* Use the `dependencies` block instead if you don't require the outputs from a module but need to maintain sequence.

* If using `dependency` block with mock outputs, use `mock_outputs_merge_strategy_with_state` to merge the actual outputs after an apply to the module's outputs map.


Hope it helps someone.

H4ppy H4ck1n6 !!! 