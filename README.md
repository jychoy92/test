# Proxmox GPU Passthrough to Docker VM

**Host:** `chzpmx01` | Intel i5-12600H | NVIDIA RTX 2000 Ada (`01:00.0`)

---

## Phase 1: Proxmox Host Configuration

> All commands run as **root** on the Proxmox host.

### 1.1 Enable IOMMU via proxmox-boot-tool (ZFS + UEFI boot)

```bash
echo "root=ZFS=rpool/ROOT/pve-1 boot=zfs intel_iommu=on iommu=pt" > /etc/kernel/cmdline
proxmox-boot-tool refresh
reboot
Verify after reboot:

cat /proc/cmdline
# Should contain: intel_iommu=on iommu=pt

find /sys/kernel/iommu_groups/ -maxdepth 1 | wc -l
# Should return > 0 (expected: 25)
1.2 Load VFIO Modules
echo -e "vfio\nvfio_iommu_type1\nvfio_pci\nvfio_virqfd" >> /etc/modules
1.3 Blacklist NVIDIA Drivers on Proxmox Host
echo -e "blacklist nouveau\nblacklist nvidia\nblacklist nvidiafb" >> /etc/modprobe.d/blacklist.conf
1.4 Bind GPU to VFIO
GPU PCI IDs for RTX 2000 Ada on this host:

10de:28b8 — GPU
10de:22be — HDMI Audio
echo "options vfio-pci ids=10de:28b8,10de:22be" >> /etc/modprobe.d/vfio.conf
update-initramfs -u -k all
reboot
Verify after reboot:

lspci -nnk -s 01:00
# Both 01:00.0 and 01:00.1 should show: Kernel driver in use: vfio-pci
Phase 2: Proxmox VM Configuration
Must be run as root on the Proxmox host.

2.1 Change VM Machine Type to q35
PCIe passthrough requires the q35 machine type. Stop the VM first:

qm stop 111
qm set 111 -machine q35
Note: Changing to q35 will rename network interfaces inside the VM (e.g. ens18 → enp6s18). Fix this after the VM boots — see Phase 2.3.

2.2 Add GPU to the Docker VM
Via Proxmox web UI (must be logged in as root or Administrator):

Go to Datacenter → select your Docker VM → Hardware
Click Add → PCI Device
Configure:
Setting	Value
Device	0000:01:00.0
All Functions	✅ Yes
Primary GPU	❌ No
PCI-Express	✅ Yes
ROM-Bar	✅ Yes
Click Add
Or via CLI:

qm set 111 -hostpci0 0000:01:00.0,pcie=1,rombar=1,x-vga=0
Start the VM:

qm start 111
2.3 Fix Network Interface Name Inside the VM
The q35 machine type renames the network interface from ens18 to enp6s18. Access the VM via Proxmox console and update the network config:

If using /etc/network/interfaces:

nano /etc/network/interfaces
# Replace ens18 with enp6s18
systemctl restart networking
If using Netplan:

nano /etc/netplan/00-installer-config.yaml
# Replace ens18 with enp6s18
netplan apply
2.4 Verify GPU is Visible Inside the VM
SSH into the Docker VM and run:

lspci | grep -i nvidia
# Should show: NVIDIA Corporation AD107GLM [RTX 2000 Ada Generation Laptop GPU]
Phase 3: Inside the Docker VM
All commands run inside the Docker VM.

3.1 Install NVIDIA Drivers
sudo apt update && sudo apt install -y build-essential dkms
sudo apt install -y nvidia-driver-550
sudo reboot
Verify:

nvidia-smi
# Should show RTX 2000 Ada with driver version and CUDA version
3.2 Install nvidia-container-toolkit
curl -fsSL https://nvidia.github.io/libnvidia-container/gpgkey \
  | sudo gpg --dearmor -o /usr/share/keyrings/nvidia-container-toolkit-keyring.gpg

curl -s -L https://nvidia.github.io/libnvidia-container/stable/deb/nvidia-container-toolkit.list \
  | sed 's#deb https://#deb [signed-by=/usr/share/keyrings/nvidia-container-toolkit-keyring.gpg] https://#g' \
  | sudo tee /etc/apt/sources.list.d/nvidia-container-toolkit.list

sudo apt update && sudo apt install -y nvidia-container-toolkit
sudo nvidia-ctk runtime configure --runtime=docker
sudo systemctl restart docker
Verify:

docker run --rm --gpus all nvidia/cuda:12.3.0-base-ubuntu22.04 nvidia-smi
# Should show GPU inside the container
Phase 4: Deploy Ollama with GPU
docker-compose.yml:

services:
  ollama:
    image: ollama/ollama:latest
    container_name: ollama
    restart: unless-stopped
    ports:
      - "11434:11434"
    volumes:
      - ollama_data:/root/.ollama
    deploy:
      resources:
        reservations:
          devices:
            - driver: nvidia
              count: all
              capabilities: [gpu]

  open-webui:
    image: ghcr.io/open-webui/open-webui:main
    container_name: open-webui
    restart: unless-stopped
    ports:
      - "3000:8080"
    environment:
      - OLLAMA_BASE_URL=http://ollama:11434
    volumes:
      - open-webui_data:/app/backend/data
    depends_on:
      - ollama

volumes:
  ollama_data:
  open-webui_data:
docker compose up -d

# Pull a model
docker exec -it ollama ollama pull llama3.2

# Verify GPU is being used
docker exec -it ollama ollama ps
Revert / Cleanup
On the Docker VM — Remove NVIDIA Drivers & Toolkit
# Remove nvidia-container-toolkit
sudo apt remove --purge -y nvidia-container-toolkit
sudo rm /etc/apt/sources.list.d/nvidia-container-toolkit.list
sudo rm /usr/share/keyrings/nvidia-container-toolkit-keyring.gpg

# Remove NVIDIA drivers
sudo apt remove --purge -y nvidia-driver-550
sudo apt autoremove -y

# Restart Docker
sudo systemctl restart docker

# Verify GPU is gone
nvidia-smi
# Should return: command not found
On the Proxmox Host — Remove GPU from VM
qm stop 111
qm set 111 --delete hostpci0
qm start 111
On the Proxmox Host — Full VFIO Cleanup (optional)
Only needed if you want to restore the NVIDIA driver on the Proxmox host itself:

# Remove vfio binding
rm /etc/modprobe.d/vfio.conf

# Remove nvidia blacklist
sed -i '/blacklist nouveau/d;/blacklist nvidia/d;/blacklist nvidiafb/d' /etc/modprobe.d/blacklist.conf

# Remove vfio modules
sed -i '/^vfio/d' /etc/modules

# Remove IOMMU from kernel cmdline
echo "root=ZFS=rpool/ROOT/pve-1 boot=zfs" > /etc/kernel/cmdline

proxmox-boot-tool refresh
update-initramfs -u -k all
reboot
Known Limitations
Snapshots Not Supported with GPU Passthrough
VMs with PCI passthrough cannot use Proxmox snapshots due to VFIO migration limitations. Use one of these alternatives:

Option 1: Proxmox Backup (recommended)

vzdump 111 --storage local-zfs --mode stop
Or via UI: VM → Backup → Backup Now → Mode: Stop

Option 2: ZFS snapshot of VM disk

zfs snapshot rpool/data/vm-111-disk-0@before-changes
Option 3: Snapshot without GPU (temporary removal)

qm stop 111
qm set 111 --delete hostpci0
qm snapshot 111 <snapname>
qm set 111 -hostpci0 0000:01:00.0,pcie=1,rombar=1,x-vga=0
qm start 111
Reference
Item	Value
Docker VM ID	111
GPU PCI slot	01:00.0
GPU PCI ID	10de:28b8
Audio PCI ID	10de:22be
IOMMU Group	16
VM Machine Type	q35
VM Network Interface	enp6s18 (renamed from ens18 after q35 change)
Ollama API	http://<vm-ip>:11434
Open WebUI	http://<vm-ip>:3000
Notes
All Proxmox host commands must be run as root — non-root users cannot assign unmapped PCI devices to VMs
The iGPU (Intel Iris Xe, 00:02.0) remains on the Proxmox host for display output — only the NVIDIA dGPU is passed through
IOMMU group 16 contains only the NVIDIA GPU and its audio function — no other devices are affected by the passthrough
Changing VM machine type to q35 will always rename network interfaces — update network config before losing SSH access
Proxmox snapshots are not supported with GPU passthrough — use vzdump backups instead
