# Setup guide

One-time setup to commit this project to GitLab, wire it to AAP, and run the first build. Do the sections in order.

## 1. Before you commit

Two files contain placeholders you must replace. Nothing else in the code needs editing before you run it.

1. **Fill in the build values.** `inventories/build/group_vars/all.yml` has six `CHANGE_ME` values:

   | Variable | Set it to |
   |---|---|
   | `build_ip` | The reserved, unused static IP for the build VM |
   | `build_netmask` | Netmask for that IP, for example `255.255.255.0` |
   | `build_gateway` | Gateway for that network |
   | `build_hostname` | Short hostname for the build VM (no dots) |
   | `build_domain` | DNS domain for the build VM |
   | `stig_appliance_host` | Hostname or IP of the STIG appliance |

   Leaving any `CHANGE_ME` in place makes provision or scan fail. Provision will not stop you first, so search the file before committing. These values are committed to git, so confirm your policy allows the build IP and appliance name in the repo. If not, delete them from this file and set them as extra variables on the AAP workflow template instead; the playbooks work either way.

2. **Fill in the profile.** `profiles/rhel9-nonprod-base/profile.yml` has placeholder names (`SOURCE_RHEL9_CONTENT_LIBRARY`, `DT_DATACENTER`, `DT_DATASTORE`, `DT_BUILD_NETWORK`, `DT_RESOURCE_POOL`, `DT_GOLDEN_IMAGE_CONTENT_LIBRARY`). Replace each with the exact vCenter name. Names are case sensitive. Check these too:
   - `placement.folder` is the full path of an existing VM folder.
   - `source.item` is a VM template item in `source.library`, and its name carries the version. A missing item fails at deploy time.
   - `placement.resource_pool` is an existing resource pool in the target cluster.
3. **Check the threshold file.** `profiles/rhel9-nonprod-base/thresholds.yml` is the compliance gate. After the first good scan, regenerate its numbers on the appliance with `saf generate threshold -i <evidence_dir>/cinc-report.json -o thresholds.yml`, then copy the result back into the repo.
4. **Check for secrets.** Nothing in this repo should contain passwords, keys, or tokens. vCenter and SSH credentials live only in AAP.

## 2. Commit to GitLab

1. In GitLab, create an **empty** project (no README). Set its default branch to `main`.
2. In the folder holding this project:

   ```bash
   git init -b main
   git remote add origin <this-project-url>
   git submodule add <stig-role-project-url> stig
   git submodule add <cinc-profile-project-url> cinc
   git add .
   git commit -m "Golden image pipeline"
   git push -u origin main
   ```

   `git submodule add` clones each project into its folder (`stig/` for the STIG role, `cinc/` for the CINC profile), writes `.gitmodules`, and records the commit you cloned. Commit `.gitmodules` and both entries. Do not copy their files in by hand. The root of the CINC profile project must be the profile itself (`inspec.yml` at the top). If the profile depends on other profiles, vendor them in that project with `cinc-auditor vendor`, because the appliance has no git or internet access to fetch them.

3. **Submodule URL.** If the other projects are on the same GitLab instance, you can use relative URLs so one credential covers all of them: `git submodule add ../<stig-project>.git stig`. It resolves against this project's remote. Use a full HTTPS URL if a project is in a different group or instance.
4. **Check it.** Run `git submodule status`. It should list `stig` and `cinc` with commit IDs. `stig/roles/rhel9_stig` and `cinc/inspec.yml` must exist.
5. **Clone elsewhere** with `git clone --recurse-submodules <url>`. Plain `git clone` leaves `stig/` empty.
6. **Take new STIG role or CINC profile changes** with `git submodule update --remote stig cinc`, then commit and push the new pointers. That push is what starts a build; see section 5.

## 3. GitLab settings

**This project**

- **Settings > Repository > Protected branches:** protect `main`. Allow merge for maintainers, and no direct push if you want every change reviewed.
- **Settings > Merge requests:** turn on "Pipelines must succeed" so the lint job gates merges.
- **Settings > CI/CD > Runners:** the lint job in `.gitlab-ci.yml` needs a runner with `ansible-lint`, `yamllint`, and `ansible-core` installed and access to Ansible Galaxy (or a mirror) for `vmware.vmware`. If your runner requires tags, add a `tags:` entry to the `lint` job.
- **Settings > Webhooks:** added in section 4 after AAP gives you the URL and key.

**STIG role project and CINC profile project**

- Nothing to change. This project only reads them.

**Access for AAP**

AAP must be able to clone this project and both submodules.

- Create a **deploy token** with `read_repository` scope on this project, and one on each of the STIG and CINC profile projects. If they are in one group, a single group deploy token covers them.
- Or use a service account with a personal access token and `read_repository` scope on both.
- AAP sends one source control credential for the project clone and its submodules, so it needs access to all three.

## 4. AAP setup

All objects are created in the AAP web UI. Nothing is committed for them.

### Prerequisites

- **Execution environment** with:
  - `vmware.vmware` 2.11.0
  - Python packages `pyVmomi` and `vmware-vcenter`
  - `community.general` and `ansible.posix`

  AAP also installs the collections in `collections/requirements.yml` on project sync if the environment can reach Galaxy or Automation Hub.
- **STIG appliance** (`stig_appliance_host`):
  - `cinc-auditor` and `saf` on the `PATH` of `lab-secops`. The CINC profile is copied there on every run, so there is nothing else to install and no git is needed.
  - `lab-secops` can SSH to the build VM, and that VM's `authorized_keys` already allows the appliance's key. The CINC scan runs from the appliance to the build VM.
- **Source template** in the content library has `lab-secops` with the keys AAP uses, VMware Tools or open-vm-tools, and Perl (needed for guest customization).
- **Network:** the AAP execution nodes can reach vCenter on 443, the build VM on port 22, and the appliance on port 22. The static `build_ip` is reserved and unused. GitLab can reach the AAP gateway URL for the webhook.

### Credentials

| Name | Type | Used by | Notes |
|---|---|---|---|
| vcenter | VMware vCenter | 10, 70, 90 | Host, username, password. The playbooks read the `VMWARE_*` variables this injects. Account needs rights to deploy from the library, power, customize, publish templates, and delete VMs in the build folder. |
| lab-secops | Machine | 20, 40, 50 | Username `lab-secops` and its SSH private key. Privilege escalation (sudo) is required for 20 and 50; 40 needs none. |
| gitlab-read | Source Control | Project | The deploy token or service account from section 3. |

### Project

- **Source control URL:** this project's HTTPS URL. **Branch:** `main`. **Credential:** `gitlab-read`.
- Options on: **Update revision on job launch**, **Track submodules**.

### Inventory

- Create an inventory, then add a **Source** of type "Sourced from a Project". Choose the project, with source file `inventories/build/hosts.yml`. Enable "Update on project update".
- Variables in `inventories/build/group_vars/all.yml` load with it, including `ansible_user: lab-secops` and the evidence directory.

### Job templates

Create one per playbook. Inventory and Project are the ones above for all of them, and no job template needs prompts.

| Job template | Playbook | Credentials | Notes |
|---|---|---|---|
| golden-10-provision | `playbooks/10-provision-build-vm.yml` | vcenter | |
| golden-20-harden | `playbooks/20-harden-rhel9.yml` | lab-secops | Privilege escalation on |
| golden-40-scan | `playbooks/40-scan-and-attest.yml` | lab-secops | Runs against `stig_appliance` |
| golden-50-generalize | `playbooks/50-generalize-sanitize.yml` | lab-secops | Privilege escalation on |
| golden-70-promote | `playbooks/70-promote-content-library.yml` | vcenter | |
| golden-90-cleanup | `playbooks/90-destroy-build-vm.yml` | vcenter | |

Set a generous timeout on 20 (the STIG run is long) and on 10 (customization waits on SSH, up to 15 minutes).

### Workflow template: `golden-image-build`

Allow concurrent jobs: **off**. Only one build VM and static IP can be in use at a time.

No extra variables are needed; the build values come from `inventories/build/group_vars/all.yml`.

**Graph**

Add nodes in this order, each linked on **success**:

1. golden-10-provision
2. golden-20-harden
3. golden-40-scan
4. golden-50-generalize
5. Approval node (set an approver team and a timeout)
6. golden-70-promote

Then add **golden-90-cleanup** and link it from:

- node 6 on **success**,
- nodes 1 to 4 on **failure**,
- the approval node on **failure** (denied or timed out).

Check that every path ends at cleanup. Cleanup does nothing harmful if the VM is already gone.

**Notifications:** add a notification template (email or similar) on the workflow for start, success, and failure. AAP only posts status back to GitLab for merge-request events, which this setup doesn't use.

**Schedule (optional):** add a schedule on the workflow if you also want nightly builds. It builds the commit your project currently points at, including the pinned `stig` role.

**Webhook**

On the workflow template, turn on **Enable webhook** and choose service **GitLab**. AAP shows a **Webhook URL** and **Webhook key**. Copy both for the next step. Leave the webhook credential empty unless you want status posted to GitLab.

### GitLab webhook

In this GitLab project, go to **Settings > Webhooks > Add new webhook**:

- **URL:** the AAP webhook URL.
- **Secret token:** the AAP webhook key.
- **Trigger:** Push events only, with branch filter `main` (a wildcard or regular expression pattern of `main`).
- **SSL verification:** on.

Use **Test > Push events** to send a sample. A successful test starts a workflow run, so only do it once the rest is ready, or cancel the run.

## 5. First run

1. In AAP, sync the project and the inventory source. Fix any submodule or credential errors now.
2. Launch **golden-image-build** manually from AAP. Watch each node.
3. Confirm after the run:
   - A new item exists in the target content library, named `<profile name>-<sha>-<workflow job id>`.
   - The build VM `gibuild-rhel9-<workflow job id>` is gone.
   - `/home/lab-secops/evidence/<workflow job id>/` on the appliance holds the report, summary, checklist, threshold file, and `attestation.json`.
4. Only after a clean manual run, rely on the webhook. Merge a small change to `main` and watch the workflow start on its own.

## How builds start

| Event | Result |
|---|---|
| Push to `main` here, including a merge request merge | AAP workflow starts |
| Merge request opened or updated | Lint job runs only |
| Change pushed to the STIG role or CINC profile project | Nothing; bump the `stig` or `cinc` pointer here and push to `main` |
| AAP schedule (if you added one) | Workflow starts |

A webhook launch always runs the project's default branch.

## Troubleshooting

| Symptom | Likely cause |
|---|---|
| `role 'rhel9_stig' was not found`, or the scan can't find the profile | `stig/` or `cinc/` is empty. Run `git submodule update --init`, or turn on "Track submodules" and check the AAP credential can read those projects. |
| Project sync fails on the submodule | The Source Control credential has no access to the STIG or CINC profile project, or a relative submodule URL doesn't resolve for it. Use a full HTTPS URL in `.gitmodules`. |
| Errors mentioning `CHANGE_ME` | A placeholder in `inventories/build/group_vars/all.yml` or `profile.yml` was not replaced. |
| Provision times out waiting for SSH | Customization failed. Check open-vm-tools and Perl in the template, `build_domain`, and the build network. Customization logs are on the VM. |
| Scan fails with a connection error | The appliance can't SSH to the build VM as `lab-secops`, or the `cinc/` profile is invalid (check `inspec.yml` is at its root). |
| Scan fails at `saf validate threshold` | Findings exceed `thresholds.yml`. Review `summary.md` in the evidence directory. |
| Webhook test returns an error | Wrong URL or key, or GitLab can't reach the AAP gateway. |
| Lint job fails on `vmware.vmware` | The runner can't reach Galaxy. Mirror the collection or pre-install it on the runner. |
