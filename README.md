# Pacemaker-Ansible: High-Availability Cluster Setup with Ansible

This repository provides an Ansible playbook to automate the configuration of a Pacemaker high-availability (HA) cluster on Red Hat Enterprise Linux (RHEL) or compatible systems (e.g., CentOS, AlmaLinux, Rocky Linux). The playbook sets up a 3-node cluster with shared storage using iSCSI, Logical Volume Manager (LVM), and an Apache web server managed by Pacemaker resources. The cluster ensures high availability for an Apache service with a virtual IP, a shared XFS filesystem, and LVM-managed storage, using Storage-Based Death (SBD) on `/dev/sdb1` for fencing.

![image](https://github.com/user-attachments/assets/c6faf52f-9963-426c-a44a-f230f0b72397)


## Features
- Configures a 3-node Pacemaker cluster (`node1`, `node2`, `node3`) with Corosync for cluster communication.
- Sets up shared storage using an iSCSI device (`/dev/sdc1`).
- Creates and manages a volume group (`my_vg`) and logical volume (`my_lv`) with LVM, formatted as XFS.
- Deploys Pacemaker resources:
  - `apache_LVM`: Manages LVM volume group activation with `system_id` locking.
  - `apache_FS`: Mounts the XFS filesystem on `/var/www`.
  - `apache_VIP`: Assigns a virtual IP (`192.168.11.200`) for HA access.
  - `apache_webSRV`: Runs the Apache HTTP server.
- Configures SBD-based STONITH on `/dev/sdb1` for reliable fencing.
- Ensures resource ordering and cleanup for idempotent execution.

## Prerequisites
- **Operating System**: RHEL 7/8/9, CentOS 7/8, AlmaLinux, or Rocky Linux.
- **Ansible**: Version 2.8 or later.
- **Nodes**: Three nodes (`node1`, `node2`, `node3`) with SSH access and `sudo` privileges.
- **iSCSI Storage**: An iSCSI target providing `/dev/sdc1` accessible to all nodes.
- **SBD Device**: A shared block device (`/dev/sdb1`) configured for SBD fencing, accessible to all nodes.
- **Packages**: Required packages (`pcs`, `pacemaker`, `corosync`, `lvm2`, `httpd`, `fence-agents`, etc.) available via system repositories.
- **Subscriptions**: For RHEL, ensure nodes are registered with access to the "High Availability" or "Resilient Storage" channels.
- **Network**: Stable network connectivity between nodes, the iSCSI target, and the SBD device.

## Repository Structure
```
pacemaker-ansible/
├── inventory               # Ansible inventory file
├── playbook.yml             # Main playbook
├── roles/                   # Ansible roles
│
│   ├── pcs-cluster/         # Role: Setup and configure PCS Cluster
│   │   ├── defaults/
│   │   │   └── main.yml
│   │   ├── files/
│   │   ├── handlers/
│   │   │   └── main.yml
│   │   ├── meta/
│   │   │   ├── main.yml
│   │   │   └── README.md
│   │   ├── tasks/
│   │   │   ├── cluster.yml
│   │   │   ├── main.yml
│   │   │   ├── resources.yml
│   │   │   └── sbd.yml
│   │   ├── templates/
│   │   │   ├── index.html.j2
│   │   │   ├── lvm.conf.j2
│   │   │   ├── sbd.sysconfig.j2
│   │   │   └── status.conf.j2
│   │   ├── tests/
│   │   │   ├── inventory
│   │   │   └── test.yml
│   │   └── vars/
│   │       └── main.yml
│
│   └── shared-storage/      # Role: Setup shared storage between nodes
│       ├── defaults/
│       │   └── main.yml
│       ├── files/
│       ├── handlers/
│       │   └── main.yml
│       ├── meta/
│       │   ├── main.yml
│       │   └── README.md
│       ├── tasks/
│       │   ├── initiator.yml
│       │   ├── main.yml
│       │   └── target.yml
│       ├── templates/
│       ├── tests/
│       │   ├── inventory
│       │   └── test.yml
│       └── vars/
│           └── main.yml

```

## Installation
1. **Clone the Repository**:
   ```bash
   git clone https://github.com/Mahmoudmohamed811/pacemaker-ansible.git
   cd pacemaker-ansible
   ```

2. **Install Ansible** (if not already installed):
   ```bash
   sudo dnf install ansible
   ```

3. **Configure Inventory**:
   Edit `inventory.yml` to define your cluster nodes:
   ```yaml
   all:
     hosts:
       node1:
         ansible_host: <node1_ip>
       node2:
         ansible_host: <node2_ip>
       node3:
         ansible_host: <node3_ip>
   ```

4. **Set Up iSCSI and SBD**:
   - Ensure the iSCSI target is configured and `/dev/sdc1` is accessible:
     ```bash
     ssh node1 'sudo iscsiadm -m session -o show'
     ssh node1 'lsblk | grep sdc1'
     ```
   - Verify the SBD device `/dev/sdb1` is available on all nodes:
     ```bash
     ssh node1 'lsblk | grep sdb1'
     ssh node2 'lsblk | grep sdb1'
     ssh node3 'lsblk | grep sdb1'
     ```

## Configuration
Customize the variables in `defaults/main.yml` as needed:
```yaml
pacemaker_cluster_name: mycluster
pacemaker_nodes:
  - node1
  - node2
  - node3
hacluster_password: "123"  # Use Ansible Vault for production
lvm_vg_name: my_vg
lvm_lv_name: my_lv
lvm_lv_size: 200
lvm_device: /dev/sdc1
apache_vip: 192.168.11.200
apache_cidr_netmask: 24
apache_group: apachegroup
apache_timeout: 60s
sbd_device: /dev/sdb1
sbd_stonith_name: MY_SBD_FENCE
```

- **Security Note**: Encrypt `hacluster_password` using Ansible Vault for production:
  ```bash
  ansible-vault encrypt_string 'your_password' --name 'hacluster_password'
  ```

## Usage
1. **Run the Playbook**:
   ```bash
   ansible-playbook -i inventory.yml playbook.yml
   ```
   Use verbose output for debugging:
   ```bash
   ansible-playbook -i inventory.yml playbook.yml -vvv
   ```

2. **Verify the Cluster**:
   - Check Pacemaker status:
     ```bash
     ssh node1 'sudo pcs status'
     ```
     Expect `apache_LVM`, `apache_FS`, `apache_VIP`, and `apache_webSRV` running.
   - Verify LVM:
     ```bash
     ssh node1 'sudo vgdisplay my_vg'
     ssh node1 'sudo lvdisplay my_vg/my_lv'
     ssh node1 'ls /dev/my_vg/my_lv'
     ```
   - Check filesystem:
     ```bash
     ssh node1 'sudo file -s /dev/my_vg/my_lv'
     ```
     Expect XFS filesystem.
   - Verify SBD:
     ```bash
     ssh node1 'sudo sbd -d /dev/sdb1 list'
     ```
   - Test Apache:
     ```bash
     curl http://192.168.11.200
     ```

3. **Manage the Cluster**:
   - Put a node in standby:
     ```bash
     ssh node1 'sudo pcs node standby node2'
     ```
   - Move resources:
     ```bash
     ssh node1 'sudo pcs resource move apache_LVM node1'
     ```
   - Check SBD status:
     ```bash
     ssh node1 'sudo sbd -d /dev/sdb1 dump'
     ```

## Screenshots
Below are screenshots demonstrating the successful setup of the Pacemaker cluster:

![Image](https://github.com/user-attachments/assets/33b0240b-9382-4496-b246-0d82fb3a0f73)

### Apache Web Service
The Apache web service running on the cluster, accessible at `http://192.168.11.200`:

![Image](https://github.com/user-attachments/assets/71c9d2c2-ca46-476b-a3c5-06692fc4e996)

## Troubleshooting
- **Error: `Device /dev/my_vg/my_lv not found`**:
  - Verify iSCSI device:
    ```bash
    ssh node1 'lsblk | grep sdc1'
    ```
  - Check LVM activation:
    ```bash
    ssh node1 'sudo vgchange -ay my_vg'
    ssh node1 'ls /dev/my_vg/my_lv'
    ```
  - Ensure Pacemaker is stopped during LVM setup:
    ```bash
    ssh node1 'sudo pcs cluster stop --all'
    ```

- **Error: `Logical volume my_vg/my_lv contains a filesystem in use`**:
  - Unmount the filesystem:
    ```bash
    ssh node1 'sudo umount /var/www'
    ```
  - Check for processes:
    ```bash
    ssh node1 'sudo lsof /dev/my_vg/my_lv'
    ```

- **SBD Issues**:
  - Verify SBD device:
    ```bash
    ssh node1 'lsblk | grep sdb1'
    ```
  - Check SBD configuration:
    ```bash
    ssh node1 'cat /etc/sysconfig/sbd'
    ```

- **Logs**:
  - LVM: `ssh node1 'sudo journalctl -u lvm2-lvmetad'`
  - Pacemaker: `ssh node1 'sudo journalctl -u pacemaker'`
  - SBD: `ssh node1 'sudo journalctl -u sbd'`

- **Verbose Output**:
  ```bash
  ansible-playbook -i inventory.yml playbook.yml -vvv > playbook.log
  ```

## Contributing
Contributions are welcome! Please:
1. Fork the repository.
2. Create a feature branch (`git checkout -b feature/your-feature`).
3. Commit changes (`git commit -m 'Add your feature'`).
4. Push to the branch (`git push origin feature/your-feature`).
5. Open a pull request.
