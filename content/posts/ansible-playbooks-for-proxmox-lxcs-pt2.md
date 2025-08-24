---
title: "Ansible Playbooks for Proxmox and LXCs Part 2"
date: 2025-02-12
draft: false
tags:
  - proxmox
  - ansible
showFullContent: false
description: "Part 2 digs into the real-world challenges of managing Proxmox container states—waiting for containers to finally be ready, handling states like 'started' and 'stopped', and upgrading our modules so SSH key injection works. A practical look at the trials (and occasional headaches) of automating Proxmox with Ansible."
---

# Ansible Playbooks for Proxmox and LXCs - Part 2

I want to continue building on the state-driven modularity of tasks in this Proxmox role. In addition to `present` (the default state) and `absent`, Proxmox supports the following states:

* `started`
* `stopped`
* `restarted`
* `template`

By having the state variable drive the behavior, we implicitly declare the desired outcome. This aligns with Ansible's idempotent design—where the playbook converges the system to the desired state instead of running a series of imperative commands.

Plus, it adds scalability: down the road, if we decide to manage even more nuanced aspects of the container lifecycle, we can handle each state independently and avoid the rigamarole of overlapping logic.

## Adding Stopped/Started States

To recap my goals:

1. Spin up a container
2. Automatically configure the container settings
3. Install some software
4. Handle all of the above in a single playbook

I need the container to automatically start itself once it is created. So, I added the `started` and stopped `states` to my tasks.

```yaml
---
# roles/proxmox_lxc/tasks/start.yml

- name: Ensure LXC container is started on Proxmox
  community.general.proxmox:
    api_host: "{{ proxmox_api_host }}"
    api_user: "{{ proxmox_api_user }}"
    api_token_id: "{{ proxmox_api_id }}"
    api_token_secret: "{{ proxmox_api_secret }}"
    node: "{{ proxmox_node }}"
    vmid: "{{ container.vmid }}"
    state: started
```

_Note:_ `stop.yml` looks exactly the same, just with the word "stop" instead of "start."

In my `main.yml` file, I added the following.

```yaml
- name: Run start tasks if state is started
  include_tasks: start.yml
  when: container.state == 'started'

- name: Run stop tasks if state is stopped
  include_tasks: stop.yml
  when: container.state == 'stopped'
```

A couple of things to note: Proxmox won't let you delete (`absent`) a container unless you stop it first. Likewise, the Proxmox module doesn't let you skip the `present` state and go directly to `started`. In other words, you can't define all your container settings, set the state to `started`, and expect it to just work. You have to create the container first using `present`, wait for it to be fully registered, and then start it.

The need to _wait_ arises due to race conditions—the container doesn't create itself fast enough. If you try to start it too early, you'll get an error message like:

```bash
TASK [proxmox_lxc : Ensure LXC container is started on Proxmox] **************************************************************************************************************************
fatal: [localhost]: FAILED! => {"changed": false, "msg": "An error occurred: 'name'"}
```

You could simply insert a pause, but that feels hacky. I would much rather _fluently_ wait until the container's state is suitable for starting. Since Ansible is built on top of the `proxmoxer` Python library—and we already have all our authentication parameters set up—why not leverage that in a script?

I saved the following Python script to `roles/proxmox_lxc/files/wait_for_container.py`.

```python
#!/usr/bin/env python3
import sys
import time
import argparse
from proxmoxer import ProxmoxAPI

parser = argparse.ArgumentParser(
    description="Wait for a Proxmox LXC container to be registered and have the expected hostname."
)
parser.add_argument("--host", required=True, help="Proxmox API host")
parser.add_argument("--user", required=True, help="Proxmox API user")
parser.add_argument("--token_name", required=True, help="Proxmox API token name")
parser.add_argument("--token_value", required=True, help="Proxmox API token secret")
parser.add_argument("--node", required=True, help="Proxmox node name")
parser.add_argument("--vmid", type=int, required=True, help="VMID of the container")
parser.add_argument("--expected-hostname", required=True, help="Expected hostname for the container")
parser.add_argument("--retries", type=int, default=10, help="Number of retries")
parser.add_argument("--delay", type=int, default=3, help="Delay between retries in seconds")
args = parser.parse_args()

proxmox = ProxmoxAPI(
    args.host,
    user=args.user,
    token_name=args.token_name,
    token_value=args.token_value,
    verify_ssl=False,
)

for attempt in range(args.retries):
    try:
        # Get the container configuration
        config = proxmox.nodes(args.node).lxc(args.vmid).config.get()
        current_hostname = config.get('hostname')
        if current_hostname == args.expected_hostname:
            print(f"Container {args.vmid} exists with expected hostname: {current_hostname}")
            sys.exit(0)
        else:
            sys.stderr.write(
                f"Attempt {attempt+1}: Container exists but hostname '{current_hostname}' does not match expected '{args.expected_hostname}'. Retrying in {args.delay} seconds...\n"
            )
    except Exception as e:
        sys.stderr.write(f"Attempt {attempt+1}: Container not found. Retrying in {args.delay} seconds...\n")
    time.sleep(args.delay)

sys.exit(1)
```

Make it executable:

```bash
chmod +x roles/proxmox_lxc/files/wait_for_container.py
```

Now, I created a new task that invokes this script, under `roles/proxmox_lxc/files/status.yml`:

```yaml
---
- name: Wait for container to be registered with expected hostname
  command: >
    {{ role_path }}/files/wait_for_container.py
    --host "{{ proxmox_api_host }}"
    --user "{{ proxmox_api_user }}"
    --token_name "{{ proxmox_api_id }}"
    --token_value "{{ proxmox_api_secret }}"
    --node "{{ proxmox_node }}"
    --vmid "{{ container.vmid }}"
    --expected-hostname "{{ container.hostname }}"
    --retries 10
    --delay 3
  register: container_status
  until: container_status.rc == 0
  retries: 10
  delay: 3
```

Let me explain:

1. The Python script loads the `proxmoxer` module and authenticates against our Proxmox instance's API.
2. It then retrieves the configuration for the VMID of the LXC we just created.
3. It checks to ensure that the container's hostname matches the hostname defined in our LXC manifest.
4. The script exits with a return code of `0` (success) if the hostname matches, or `1` (failure) if not.
5. Our Ansible task runs this script, waiting (with a 3-second delay) until it sees a return code of `0`, retrying up to 10 times.

Once these conditions are met, the container is fully spun up and ready to be started.

## Upgrade the `community.general` Module and Uploading a Public Key

It is essential that we be able to `ssh` into our instance once it is configured by Ansible. By default, Debian does not allow root login using a password—you'd have to log in to your LXC via Proxmox (`pct enter`) and explicitly enable it. And that is:

1. Not in the spirit of full automation
2. Shit security

Instead, we should include a public key using the `pubkey` parameter. In `create.yml`, I added the following:

```yaml
pubkey: "{{ lookup('file', container.pubkey_file) | default(omit) }}"
```

In your `lxcs.yml` file, add the path to your private key:

```yaml
pubkey_file: "~/.ssh/nuc_rsa.pub"
```

It was at this moment that I ran my playbook and encountered a very confusing error that took me a couple of hours to figure out:

```json
{
   "changed": false,
   "msg": "An error occurred: 400 Bad Request: Parameter verification failed. - {'ssh-public-key': 'property is not defined in schema and the schema does not allow additional properties'}"
}
```

What do you _mean_ the property is not in the schema? It is clearly defined in the [Ansible document](https://docs.ansible.com/ansible/latest/collections/community/general/proxmox_module.html#parameter-pubkey)!

Well, it turns out that the version of the `community.general` module that came _pre-installed_ with the most recent version of Ansible was not up-to-date. I fixed this by re-running the collection installation with the `--upgrade` flag:

```bash
ansible-galaxy collection install community.general --upgrade
```

In retrospect, this probably should have been done at the very start. I _did_ run `ansible-galaxy collection install community.general`, but it _didn't tell me there was a new version available_. I accept responsibility for that oversight, though a clearer message would have been nice.

## Storing the Container IP

In practice, we could just give our container a static IP, but, in short, I don't want to. I create my containers using DHCP and let the router give them an address automatically. Afterwards, I'll put a reservation in my router and modify the container to be static (we can automate this too, but not for transient test VMs—later).

That said, we need to make another `proxmoxer` script to get the interface info from the newly created LXC. Create an executable file:

```bash
touch roles/proxmox_lxc/files/get_container_ip.py
chmod +x roles/proxmox_lxc/files/get_container_ip.py
```

Edit that file and add the following:

```python
#!/usr/bin/env python3
import sys
import time
import argparse
from proxmoxer import ProxmoxAPI

parser = argparse.ArgumentParser(
    description="Retrieve container IP from Proxmox with retries"
)
parser.add_argument("--host", required=True, help="Proxmox API host")
parser.add_argument("--user", required=True, help="Proxmox API user")
parser.add_argument("--token_name", required=True, help="Proxmox API token name")
parser.add_argument("--token_value", required=True, help="Proxmox API token secret")
parser.add_argument("--node", required=True, help="Proxmox node name")
parser.add_argument("--vmid", type=int, required=True, help="VMID of the container")
parser.add_argument("--retries", type=int, default=10, help="Number of retries")
parser.add_argument("--delay", type=int, default=3, help="Delay between retries in seconds")
args = parser.parse_args()

proxmox = ProxmoxAPI(
    args.host,
    user=args.user,
    token_name=args.token_name,
    token_value=args.token_value,
    verify_ssl=False
)

ip_address = None
for attempt in range(args.retries):
    try:
        interfaces = proxmox.nodes(args.node).lxc(args.vmid).interfaces.get()
    except Exception as e:
        sys.stderr.write(f"Attempt {attempt+1}: Error retrieving interfaces: {e}\n")
        time.sleep(args.delay)
        continue

    for interface in interfaces:
        if interface.get("name") == "eth0":
            inet = interface.get("inet")
            if inet:
                ip_address = inet.split("/")[0]
                break

    if ip_address:
        print(ip_address)
        sys.exit(0)
    else:
        sys.stderr.write(f"Attempt {attempt+1}: eth0 not found or no IP assigned. Retrying in {args.delay} seconds...\n")
        time.sleep(args.delay)

sys.stderr.write("Failed to retrieve container IP address after multiple attempts.\n")
sys.exit(1)
```

Now create the new task under `roles/proxmox_lxc/tasks/get_ip.yml`:

```yaml
---
- name: Retrieve container IP via DHCP using proxmoxer
  command: >
    {{ role_path }}/files/get_container_ip.py
    --host "{{ proxmox_api_host }}"
    --user "{{ proxmox_api_user }}"
    --token_name "{{ proxmox_api_id }}"
    --token_value "{{ proxmox_api_secret }}"
    --node "{{ proxmox_node }}"
    --vmid "{{ container.vmid }}"
    --retries 10
    --delay 3
  register: ip_result
  changed_when: false

- name: Set container IP fact
  set_fact:
    container_ip: "{{ ip_result.stdout }}"

# Debug task to show container IP - comment out if not needed
- name: Debug - Show container IP
  debug:
    msg: "Container IP is: {{ container_ip }}"
```

This will execute out Python script, store the IP as a fact and (optionally) output it to the terminal. Add this task to `main.yml` like so:

```yaml
- name: Retrieve container IP if flag is set
  include_tasks: get_ip.yml
  when: container.get_ip | default(false)
```

This tells your your playbook to retrieve the container up when the `get_ip` flag is set to `true`. Try it out by stopping and starting a container with the flag set:

```yaml
lxcs:
  - vmid: 114
    state: stopped
  - vmid: 114
    state: started
    get_ip: true
```

You should see the following in your output.

```bash
TASK [proxmox_lxc : Debug - Show container IP] ********************************************
ok: [localhost] => {
    "msg": "Container IP is: 192.168.1.154"
}
```

## Closing

It's late, and this is a lot longer than I originally anticipated. I'm going to call it a wrap for part 2. In part 3, we'll continue configuring the fully provisioned LXCs over SSH with Ansible. I'll commit these changes [to GitHub](https://github.com/sbarbett/proxmox-ansible) if you want to clone or fork the repository.

TTFN