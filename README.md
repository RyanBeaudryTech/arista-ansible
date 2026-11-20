# Introduction
This project differs from my previous automation projects in that ansible is using eAPI to access the devices instead of SSH. API calls are made to the Arista switches using HTTPS over port 443 and the results are sent back in JSON.

The topology is a data center spine-leaf underlay built with Arista cEOS in Containerlab, configured and verified entirely with Ansible over eAPI.

The goal of this project is to treat the network as code: the topology, IP plan, and configuration all live in version-controlled files, and a verification role proves the underlay actually works rather than just confirming that config was pushed.

## Topology

![Topology](docs/topology.png)

- **2 spines, 2 leaves**, running Arista cEOS-lab 4.36.3.2M
- Every leaf connects to every spine. Spines do not connect to each other, and leaves do not connect to each other.
- Routed links between all switches, with an eBGP underlay and ECMP across both spines

## Design

### Addressing

| Switch | Loopback0 | BGP ASN |
|---|---|---|
| spine1 | 10.0.0.1/32 | 65000 |
| spine2 | 10.0.0.2/32 | 65000 |
| leaf1 | 10.0.0.11/32 | 65001 |
| leaf2 | 10.0.0.12/32 | 65002 |

| Link | Spine side | Leaf side |
|---|---|---|
| spine1 Eth1 to leaf1 Eth1 | 10.1.1.0/31 | 10.1.1.1/31 |
| spine1 Eth2 to leaf2 Eth1 | 10.1.1.2/31 | 10.1.1.3/31 |
| spine2 Eth1 to leaf1 Eth2 | 10.1.2.0/31 | 10.1.2.1/31 |
| spine2 Eth2 to leaf2 Eth2 | 10.1.2.2/31 | 10.1.2.3/31 |

The third octet of each link subnet identifies the spine (10.1.**1**.x for spine1, 10.1.**2**.x for spine2), which makes troubleshooting faster.

### Design decisions

- **/31 point-to-point links.** Each routed link only ever has two devices, so a /31 uses exactly two addresses with nothing wasted.
- **Shared spine ASN (65000).** eBGP drops any route that already contains its own ASN in the AS path. Because both spines share an ASN, a route learned through spine1 cannot be passed back up to spine2, which prevents routing loops through the fabric.
- **Unique ASN per leaf.** The same rule means leaves must have different ASNs. If both leaves used the same ASN, each would reject the other's loopback route.
- **ECMP with `maximum-paths 4 ecmp 4`.** BGP installs only one best path by default. Raising the limit lets each leaf load-balance across both spines, with headroom to add spines later without changing the config.

## Repository structure

```
.
├── ansible.cfg                 # Points Ansible at the inventory and roles
├── inventory.yml               # Hosts grouped into spines and leaves
├── lab.clab.yml                # Containerlab topology definition
├── group_vars/
│   └── all.yml                 # eAPI connection settings for all switches
├── host_vars/                  # Per-switch data: loopback, ASN, links
│   ├── clab-arista-lab-spine1.yml
│   ├── clab-arista-lab-spine2.yml
│   ├── clab-arista-lab-leaf1.yml
│   └── clab-arista-lab-leaf2.yml
├── roles/
│   ├── underlay_interfaces/    # Loopback and routed P2P interfaces (Jinja2 template)
│   ├── underlay_bgp/           # eBGP underlay with ECMP (Jinja2 template)
│   └── verify_fabric/          # Post-deployment validation checks
└── playbooks/
    ├── test.yml                # eAPI connectivity test
    ├── underlay.yml            # Builds the underlay
    └── verify.yml              # Validates the fabric
```

The data (IP plan, ASNs, neighbors) lives in `host_vars`, and the config format lives in Jinja2 templates inside each role. Adding a switch means adding a host_vars file and inventory entry, not writing new config.

## How it works

### Configuration

Ansible connects to each switch over **eAPI** (HTTPS/JSON) using the `arista.eos` collection, rather than SSH and screen-scraping. Each role renders a Jinja2 template with the switch's host_vars and pushes it with `eos_config`, which compares the rendered config against the running config and only applies the differences.

Running the underlay playbook a second time reports `changed=0`, confirming the playbook is idempotent and safe to re-run.

### Verification

The `verify_fabric` role pulls structured JSON from each switch and asserts on it:

| Check | Runs on | Passes when |
|---|---|---|
| BGP sessions | All switches | Every neighbor defined in host_vars is `Established` |
| ECMP | Leaves | The route to each other leaf's loopback has one next hop per spine |
| Reachability | Leaves | Pings between leaf loopbacks, sourced from the local loopback, succeed |

Nothing in the checks is hardcoded. Expected neighbors come from host_vars, the list of remote leaves comes from the inventory, and the expected number of ECMP paths is the number of spines in the inventory. Adding a leaf or spine updates the checks automatically.

**Proving the checks work:** removing `maximum-paths` from leaf1 by hand causes the ECMP check to fail on leaf1. Re-running the underlay playbook restores the config, and the verification passes again.

## Running the lab

### Requirements

- Linux host or VM (tested on Ubuntu 22.04 with 16 GB RAM and 4 vCPUs). This four node lab uses roughly 6.5 GB of memory.
- Docker
- [Containerlab](https://containerlab.dev)
- Ansible with the `arista.eos` collection: `ansible-galaxy collection install arista.eos`

### Getting the cEOS image

The Arista lab image cannot be redistributed, so it is not included in this repo. Download it from [arista.com](https://www.arista.com/en/support/software-download) (sign up with a free account, requires a company email address) then import it:

```bash
docker import cEOS64-lab-4.36.3.2M.tar ceos:4.36.3.2M
```

If you use a different version, update the `image:` line in `lab.clab.yml` to match.

### Deploy and configure

```bash
# Start the lab
sudo containerlab deploy -t lab.clab.yml

# Confirm Ansible can reach all switches
ansible-playbook playbooks/test.yml

# Build the underlay
ansible-playbook playbooks/underlay.yml --diff

# Verify the fabric
ansible-playbook playbooks/verify.yml
```

### Tear down

```bash
sudo containerlab destroy -t lab.clab.yml --cleanup
```

## Notes

- The credentials in `group_vars/all.yml` (`admin`/`admin`) are the Containerlab default for cEOS lab nodes and are not used anywhere else.


## Lessons learned

**Lab environment**
- **cEOS needs real memory.** My first deploy froze the Ubuntu VM completely. Four cEOS nodes use roughly 6.5 GB of RAM, and the VM only had a few GB, so Linux ran out of memory and locked up. Raising the VM to 16 GB fixed it, and I now keep `free -h` open in a second terminal when deploying.
- **Expanding a VM disk takes two steps.** Importing the cEOS image failed with a disk space error. Growing the virtual disk in VMware wasn't enough on its own: I also had to grow the partition and filesystem inside Ubuntu with `growpart` and `resize2fs` before the space was usable.
- **Getting the image has a catch.** Arista's registration doesn't accept personal email domains like Gmail, which is worth knowing before planning a home lab around cEOS.

**EOS**
- **`ip routing` is off by default.** Unlike what I'm used to on routers, EOS doesn't route until you turn it on, and every port is a switchport until you set `no switchport`.
- **BGP only installs one path by default.** Both spines advertised the same routes, but without `maximum-paths` each leaf only used one of them. ECMP has to be enabled explicitly.

**Ansible and eAPI**
- **`eos_command` returns text unless you ask for JSON.** My first test playbook reached every switch but failed when I tried to read `stdout[0].version`, because the output was a block of text. Adding `output: json` returned structured data I could query directly.
- **eAPI sessions start unprivileged.** Read-only commands worked, but pushing config failed with "privileged mode required." Setting `ansible_become: true` and `ansible_become_method: enable` fixed it.
- **A task's keywords aren't steps.** `loop`, `when`, and `register` are settings on the task, not lines that run in order. The module is what gets repeated for each loop item, no matter where `loop:` appears in the YAML. Understanding this logic was an adjustment after working with Python which executes lines in order.
- **Text matching is fragile.** Checking ping output for `0% packet loss` would also match `100% packet loss`. Matching `' 0% packet loss'` (with the leading space) fixes it, but it's a good example of why it is preferable to query on JSON fields wherever possible.
- **A check that has never failed hasn't proven anything.** I removed `maximum-paths` by hand to confirm the ECMP check actually catches the problem before trusting it.

## Next step

Add an EVPN/VXLAN overlay on top of the underlay, so hosts on different leaves can share a Layer 2 network across the routed fabric.
