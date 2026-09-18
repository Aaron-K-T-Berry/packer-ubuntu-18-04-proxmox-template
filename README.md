# Packer ubuntu proxmox template

This repo has a packer template file to create a cloud init enabled ubuntu server 18.04 proxmox template.

This repo has [Taskfile](https://taskfile.dev/#/) support to make running some of the common commands easier.

**See also:** [Ubuntu 20.04](https://github.com/Aaron-K-T-Berry/packer-ubuntu-20-04-proxmox-template) · [Ubuntu 24.04](https://github.com/Aaron-K-T-Berry/packer-ubuntu-24-04-proxmox-template) — modern live-server autoinstall siblings (see [Sibling templates](#sibling-templates)).

## How installation works

- Uses the classic **Ubuntu Server ISO** (not live-server) with **debian-installer preseed** via [`http/preseed.cfg`](./http/preseed.cfg). This is **not** Subiquity autoinstall.
- Packer serves the preseed over HTTP. The guest must reach Packer's advertised HTTP IP and port during boot (same reachability class of failure as autoinstall, different boot command).
- Cloud-init is **not** enabled by the Packer builder. A **shell-local post-processor** SSHes to the PVE host as `root` and runs `qm set` (cloudinit drive, boot disk, ciuser/cipassword, SCSI hardware). That SSH access is separate from the Proxmox API credentials in your var file.
- Example defaults differ from the 20.04/24.04 siblings: about **512 MiB** RAM, **50G** disk, and `datastore` + `datastore_type` (for example `local` / `directory`).

## Sibling templates

Other Ubuntu Proxmox Packer templates in this family:

| Version | Install path | Repository |
| --- | --- | --- |
| **18.04 (this repo)** | Classic server ISO + debian-installer preseed; cloud-init via `qm set` | [packer-ubuntu-proxmox-template](https://github.com/Aaron-K-T-Berry/packer-ubuntu-proxmox-template) |
| [20.04](https://github.com/Aaron-K-T-Berry/packer-ubuntu-20-04-proxmox-template) | live-server + Subiquity autoinstall; builder-native cloud-init | [packer-ubuntu-20-04-proxmox-template](https://github.com/Aaron-K-T-Berry/packer-ubuntu-20-04-proxmox-template) |
| [24.04](https://github.com/Aaron-K-T-Berry/packer-ubuntu-24-04-proxmox-template) | Same autoinstall pattern as 20.04, with higher RAM and a firstboot helper | [packer-ubuntu-24-04-proxmox-template](https://github.com/Aaron-K-T-Berry/packer-ubuntu-24-04-proxmox-template) |

## Creating a ubuntu proxmox template

You can run the template with the following methods:

- [The simple way](#The-simple-way)
- [The manual way](#The-manual-way)

This template has some required configuration variables, all of these variables can be seen in the `example-vars.json` copy this file and fill in all variables for your environment and pass the var file to packer with the `-var-file` option.

For this template you will need to have the [`ubuntu 18.04 server iso`](http://cdimage.ubuntu.com/releases/18.04/release/) locally available on the PVE host. If you have [`task`](https://taskfile.dev/#/) installed you can do this with the `task ubuntu-18-iso` command to download it to your local machine and then upload it to the PVE host.

The ISO on the node must match the `iso` variable, for example `local:iso/ubuntu-18.04.4-server-amd64.iso`.

### The simple way

This repo uses a [Taskfile](https://taskfile.dev/#/) for common commands. You can find installation instructions for `task` [here](https://taskfile.dev/#/installation).

1. Configure template varibles for your setup

   ```shell
   $ task init
   cp ./example-vars.json ./config.json
   ```

   After the task has completed fill out the `config.json` file for your environment. See the [variables](#variables) section for more details on what these variables are.

2. Check your template file is valid

   ```shell
   $ task validate
   packer validate ./ubuntu-18.04.json
   Template validated successfully.
   ```

3. Build your proxmox template

   ```shell
   $ task build
   packer build -var-file="./config.json" ./ubuntu-18.04.json
   proxmox: output will be in this color.

   ==> proxmox: Creating VM
   ==> proxmox: Starting VM
   ==> proxmox: Starting HTTP server on port 8771

   ...

   Build 'proxmox' finished.

   ==> Builds finished. The artifacts of successful builds are:
   --> proxmox: A template was created: 4444
   --> proxmox:
   ```

4. Your template should now be available on your PVE host

### The manual way

1. Configure template varibles for your setup

   ```shell
   cp ./example-vars.json ./config.json
   ```

   After copying, fill out the `config.json` file for your environment. See the [variables](#variables) section for more details on what these variables are.

2. Check your template file is valid

   ```shell
   $ packer validate ./ubuntu-18.04.json
   Template validated successfully.
   ```

3. Build your proxmox template

   ```shell
   $ packer build -var-file="./config.json" ./ubuntu-18.04.json
   proxmox: output will be in this color.

   ==> proxmox: Creating VM
   ==> proxmox: Starting VM
   ==> proxmox: Starting HTTP server on port 8771

   ...

   Build 'proxmox' finished.

   ==> Builds finished. The artifacts of successful builds are:
   --> proxmox: A template was created: 4444
   --> proxmox:
   ```

4. Your template should now be ready on your PVE host

If you had no errors building your image you should now have a proxmox template ready on your PVE host that is cloud init enabled and should be named the same as the `template_name` variable you set in your `config.json` file.

## Variables

| Variable               | Description                                                                                                                                                                                 | Example                                                                       |
| ---------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ----------------------------------------------------------------------------- |
| `proxmox_host`         | The hostname of your pve node that will be accessed via the web api                                                                                                                         | `100.0.0.100:8006`, `pve.example.com`                                         |
| `proxmox_node_name`    | The name of the node the template will be created on                                                                                                                                        | `pve-01`                                                                      |
| `proxmox_api_user`     | User plus login method for the user that will perform the commands on the PVE host                                                                                                          | `root@pam`                                                                    |
| `proxmox_api_password` | Password of the user that will perform the commands on the PVE host                                                                                                                         | `password`                                                                    |
| `template_name`        | Name of the proxmox template to be created, Good to make this unique and easily identifiable                                                                                                | `ubuntu-18-04-{{ isotime \"2006-01-02-T15-04-05\" }}`                         |
| `template_description` | Description applied to the proxmox template to be created                                                                                                                                   | `Ubuntu 18.04, generated by packer on {{ isotime \"2006-01-02T15:04:05Z\" }}` |
| `ssh_username`         | Default username, if modified from `packer` update the default [`preseed.cfg`](./http/preseed.cfg) file `d-i preseed/late_command string` command and update it to your new sudoer username | `packer`                                                                      |
| `ssh_fullname`         | Default user full name                                                                                                                                                                      | `packer`                                                                      |
| `ssh_password`         | Default user password                                                                                                                                                                       | `packer`                                                                      |
| `hostname`             | Internal OS hostname                                                                                                                                                                        | `ubuntu-18-04-cloudinit`                                                      |
| `vmid`                 | ID of the vm that will be created to configure the template                                                                                                                                 | `4000`                                                                        |
| `locale`               | Locale of the created VM                                                                                                                                                                    | `en_US`                                                                       |
| `cores`                | Available cores on the template                                                                                                                                                             | `1`                                                                           |
| `sockets`              | Available sockets on the template                                                                                                                                                           | `1`                                                                           |
| `memory`               | Available memory on the template                                                                                                                                                            | `512`                                                                         |
| `disk_size`            | OS disk size of the template                                                                                                                                                                | `50G`                                                                         |
| `datastore`            | The data store the vm os disk will be created on                                                                                                                                            | `local`                                                                       |
| `datastore_type`       | The type of the os disk data store                                                                                                                                                          | `directory`                                                                   |
| `iso`                  | PVE host local path to the OS ISO image for the template                                                                                                                                    | `local:iso/ubuntu-18.04.4-server-amd64.iso`                                   |
| `boot_command_prefix`  | Command to enter the command line during the installation process                                                                                                                           | `<esc><esc><enter><wait>`                                                     |
| `preseed_file`         | Pre seed file you wish to be used relative to the `./http` directory                                                                                                                        | `preseed.cfg`                                                                 |

## Troubleshooting

### The installer never starts, or Packer times out waiting for SSH

The guest must download the preseed from Packer's HTTP server. If the advertised HTTP IP/port is not reachable from the PVE node or guest (wrong interface, firewall, VPN-only address), the install stalls.

- Confirm Packer prints `Starting HTTP server on port ...` and that the PVE node can reach that host and port
- Run Packer on a host the PVE bridge can route to

### The build finishes the guest install but cloud-init is missing on the template

The cloud-init drive is applied by the post-processor over SSH (`ssh root@… qm set …`). API credentials alone are not enough.

- Ensure you can SSH as `root` to the PVE host used in `proxmox_host`
- If `proxmox_host` includes the API port (`:8006`), SSH may fail; use a hostname or IP that works for SSH, or adjust the post-processor target accordingly

## Notes

- This template has been tested against the `ubuntu-18.04.4-server-amd64.iso` file (classic server ISO, not live-server)
- You can configure the OS setup by modifying the [preseed file](https://help.ubuntu.com/lts/installation-guide/s390x/apbs02.html) located in the `./http` folder
- Cloud-init is enabled via the shell-local `qm set` post-processor, which requires SSH as `root` to the PVE host
- `config.json` is gitignored. Keep real API passwords out of git
