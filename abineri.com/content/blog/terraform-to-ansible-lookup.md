---
title: "Using data from Terraform in Ansible"
date: 2018-06-05T13:21:04+01:00
draft: false
---

My current role involves a lot of environments that are defined as some Terraform resources along with Ansible playbooks to configure the application stack. For a recent project, I was destroying and recreating one environment multiple times, and found that some values I needed to use "later" in Ansible were different after destruction and recreation, and it would be much easier to look them up _during_ the Ansible run, rather than having to manually update Ansible variables.

In the example I'll work through below, I needed to fetch the internal address of a secondary IP configuration and pass it into Ansible. (In this case, we're using Azure, but the principle would be the same across Terraform providers.)

The first step in this scenario is to use a Terraform "data source" to get the data, because the value (the IP address) isn't set until the resource exists.

```
data "azurerm_network_interface" "dmz1" {
  name                = "${var.name}-dmz-1"
  resource_group_name = "${module.group.group}"
}
```

Then we tell Terraform to output the second (as usual, the list is zero-indexed) private IP address from the network interface - the name you give the output is the name you will pass to `terraform output` later.

```
output dmz1_secondary_ip {
  value = "${data.azurerm_network_interface.dmz1.private_ip_addresses[1]}"
}
```

Now you can run `terraform refresh` to populate the output - Terraform will show you the outputs at the end of the refresh task so you can see if it is what you expect.

The last piece of the puzzle is to use Ansible's `pipe` lookup, which runs a command and returns the output (I agree, pipe is a strange name for it. I guess you're "piping" to and from the command?)

```
secondary_ip: "{{ lookup('pipe','cd .. && terraform output dmz1_secondary_ip') }}"
```

The `cd ..` is because in our workflow, we keep Ansible playbooks in a separate directory (called `ansible`, we're very creative) "under" the Terraform resources. The working directory of the "piped" command is the working directory of the playbook.

You can also do any handling you need as part of the lookup, for example:

```
share_password: "{{ lookup('pipe','cd .. && terraform output primary_connection_string | cut -d \\; -f3 | sed s/AccountKey=//') }}"
```

Now, as part of your pipeline/script/manual invocation of Ansible, it will get the information from Terraform's state automatically. Note that `terraform output` doesn't refresh the state, it just outputs what is in Terraform's state, so for _really_ dynamic data you may want to run `terraform refresh` as part of the process.