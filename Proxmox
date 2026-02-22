#!/bin/bash
set -euo pipefail

# =============================
# Enhanced Multi-VM Manager (Pure QEMU + Proxmox Version)
# =============================

# Function to display header
display_header() {
    clear
    cat << "EOF"
 ███████████                                                                        
░░███░░░░░███                                                                       
 ░███    ░███ ████████   ██████  █████ █████ █████████████    ██████  █████ █████   
 ░██████████ ░░███░░███ ███░░███░░███ ░░███ ░░███░░███░░███  ███░░███░░███ ░░███    
 ░███░░░░░░   ░███ ░░░ ░███ ░███ ░░░█████░   ░███ ░███ ░███ ░███ ░███ ░░░█████░     
 ░███         ░███     ░███ ░███  ███░░░███  ░███ ░███ ░███ ░███ ░███  ███░░░███    
 █████        █████    ░░██████  █████ █████ █████░███ █████░░██████  █████ █████   
░░░░░        ░░░░░      ░░░░░░  ░░░░░ ░░░░░ ░░░░░ ░░░ ░░░░░  ░░░░░░  ░░░░░ ░░░░░    
                                                                
        🔧 Pure QEMU + Proxmox Integration 🔧                                      
                                                                                                       
EOF
    echo
}

# Initialize paths and logging
VM_DIR="${VM_DIR:-$HOME/vms}"
PROXMOX_DIR="${PROXMOX_DIR:-$HOME/proxmox}"
LOG_FILE="$VM_DIR/vm_manager.log"
mkdir -p "$VM_DIR" "$PROXMOX_DIR"

# Configuration file
CONFIG_FILE="$HOME/.vm_manager.conf"

# Function to save configuration
save_config() {
    cat > "$CONFIG_FILE" <<EOF
# VM Manager Configuration
VM_DIR="$VM_DIR"
PROXMOX_DIR="$PROXMOX_DIR"
PROXMOX_HOST="${PROXMOX_HOST:-}"
PROXMOX_USER="${PROXMOX_USER:-}"
PROXMOX_REALM="${PROXMOX_REALM:-pam}"
PROXMOX_NODE="${PROXMOX_NODE:-}"
STORAGE="${STORAGE:-local}"
EOF
}

# Function to load configuration
load_config() {
    if [[ -f "$CONFIG_FILE" ]]; then
        source "$CONFIG_FILE"
    fi
}

# Function to configure Proxmox
configure_proxmox() {
    print_status "INFO" "🔧 Configuring Proxmox connection"
    
    read -p "$(print_status "INPUT" "🌐 Proxmox Host/IP (e.g., 192.168.1.100): ")" PROXMOX_HOST
    read -p "$(print_status "INPUT" "👤 Proxmox Username (e.g., root@pam): ")" PROXMOX_USER
    read -p "$(print_status "INPUT" "🔑 Proxmox Password: ")" -s PROXMOX_PASS
    echo
    read -p "$(print_status "INPUT" "🏷️  Proxmox Node name (e.g., pve): ")" PROXMOX_NODE
    read -p "$(print_status "INPUT" "💾 Storage name (e.g., local-lvm): ")" STORAGE
    
    # Test connection
    if command -v pvesh &>/dev/null; then
        if PASSWORD="$PROXMOX_PASS" pvesh get /nodes/$PROXMOX_NODE/status --no-header 2>/dev/null; then
            print_status "SUCCESS" "✅ Proxmox connection successful"
        else
            print_status "ERROR" "❌ Failed to connect to Proxmox"
            return 1
        fi
    fi
    
    save_config
}

# Function to log messages
log_message() {
    local level=$1
    local message=$2
    echo "$(date '+%Y-%m-%d %H:%M:%S') [$level] $message" >> "$LOG_FILE"
}

# Function to display colored output with emojis
print_status() {
    local type=$1
    local message=$2
    
    case $type in
        "INFO") echo -e "\033[1;34m📋 [INFO]\033[0m $message"; log_message "INFO" "$message" ;;
        "WARN") echo -e "\033[1;33m⚠️  [WARN]\033[0m $message"; log_message "WARN" "$message" ;;
        "ERROR") echo -e "\033[1;31m❌ [ERROR]\033[0m $message"; log_message "ERROR" "$message" ;;
        "SUCCESS") echo -e "\033[1;32m✅ [SUCCESS]\033[0m $message"; log_message "SUCCESS" "$message" ;;
        "INPUT") echo -e "\033[1;36m🎯 [INPUT]\033[0m $message" ;;
        *) echo "[$type] $message"; log_message "$type" "$message" ;;
    esac
}

# Function to show progress indicator
show_progress() {
    local pid=$1
    local msg=$2
    local spin='-\|/'
    local i=0
    
    echo -n "$msg "
    while kill -0 "$pid" 2>/dev/null; do
        i=$(( (i+1) %4 ))
        printf "\r${msg} ${spin:$i:1}"
        sleep 0.1
    done
    printf "\r${msg} ✅\n"
}

# Function to check if image file is locked
check_image_lock() {
    local img_file=$1
    local vm_name=$2
    
    # Check if QEMU is already using this image
    if lsof "$img_file" 2>/dev/null | grep -q qemu-system; then
        print_status "WARN" "🔒 Image file $img_file is already in use by another QEMU process"
        
        # Find the process ID
        local pid=$(lsof "$img_file" 2>/dev/null | grep qemu-system | awk '{print $2}' | head -1)
        if [[ -n "$pid" ]]; then
            print_status "INFO" "🔍 Process ID using the image: $pid"
            
            # Check if it's our own VM
            if ps -p "$pid" -o cmd= | grep -q "$vm_name"; then
                print_status "INFO" "🤔 This appears to be the same VM already running"
                read -p "$(print_status "INPUT" "🔄 Kill existing process and restart? (y/N): ")" kill_choice
                if [[ "$kill_choice" =~ ^[Yy]$ ]]; then
                    kill "$pid"
                    sleep 2
                    if kill -0 "$pid" 2>/dev/null; then
                        kill -9 "$pid"
                        print_status "WARN" "⚠️  Forcefully terminated process $pid"
                    fi
                    return 0
                else
                    return 1
                fi
            else
                print_status "ERROR" "🚫 Another QEMU instance is using this image"
                return 1
            fi
        fi
        return 1
    fi
    
    # Check for lock files
    local lock_file="${img_file}.lock"
    if [[ -f "$lock_file" ]]; then
        print_status "WARN" "🔒 Lock file found: $lock_file"
        
        # Check if lock file is stale (older than 5 minutes)
        if [[ $(find "$lock_file" -mmin +5 2>/dev/null) ]]; then
            print_status "WARN" "⏰ Lock file appears stale (older than 5 minutes)"
            read -p "$(print_status "INPUT" "🗑️  Remove stale lock file? (y/N): ")" remove_lock
            if [[ "$remove_lock" =~ ^[Yy]$ ]]; then
                rm -f "$lock_file"
                print_status "SUCCESS" "✅ Removed stale lock file"
                return 0
            else
                return 1
            fi
        fi
        return 1
    fi
    return 0
}

# Function to validate input
validate_input() {
    local type=$1
    local value=$2
    
    case $type in
        "number")
            if ! [[ "$value" =~ ^[0-9]+$ ]]; then
                print_status "ERROR" "❌ Must be a number"
                return 1
            fi
            ;;
        "size")
            if ! [[ "$value" =~ ^[0-9]+[GgMm]$ ]]; then
                print_status "ERROR" "❌ Must be a size with unit (e.g., 100G, 512M)"
                return 1
            fi
            ;;
        "port")
            if ! [[ "$value" =~ ^[0-9]+$ ]] || [ "$value" -lt 23 ] || [ "$value" -gt 65535 ]; then
                print_status "ERROR" "❌ Must be a valid port number (23-65535)"
                return 1
            fi
            ;;
        "name")
            if ! [[ "$value" =~ ^[a-zA-Z0-9_-]+$ ]]; then
                print_status "ERROR" "❌ VM name can only contain letters, numbers, hyphens, and underscores"
                return 1
            fi
            ;;
        "username")
            if ! [[ "$value" =~ ^[a-z_][a-z0-9_-]*$ ]]; then
                print_status "ERROR" "❌ Username must start with a letter or underscore, and contain only letters, numbers, hyphens, and underscores"
                return 1
            fi
            ;;
        "vmid")
            if ! [[ "$value" =~ ^[0-9]+$ ]] || [ "$value" -lt 100 ] || [ "$value" -gt 999999999 ]; then
                print_status "ERROR" "❌ VMID must be a number between 100 and 999999999"
                return 1
            fi
            ;;
    esac
    return 0
}

# Function to check dependencies
check_dependencies() {
    local deps=("qemu-system-x86_64" "wget" "qemu-img" "lsof")
    local missing_deps=()
    
    for dep in "${deps[@]}"; do
        if ! command -v "$dep" &> /dev/null; then
            missing_deps+=("$dep")
        fi
    done
    
    # Check for cloud-init tools
    if ! command -v "cloud-localds" &> /dev/null; then
        if command -v "genisoimage" &> /dev/null; then
            print_status "INFO" "⚠️  Using genisoimage instead of cloud-localds"
        else
            missing_deps+=("cloud-image-utils")
        fi
    fi
    
    if [ ${#missing_deps[@]} -ne 0 ]; then
        print_status "ERROR" "🔧 Missing dependencies: ${missing_deps[*]}"
        print_status "INFO" "💡 On Ubuntu/Debian, try: sudo apt install qemu-system cloud-image-utils wget lsof genisoimage"
        exit 1
    fi
}

# Function to cleanup temporary files
cleanup() {
    if [ -f "user-data" ]; then rm -f "user-data"; fi
    if [ -f "meta-data" ]; then rm -f "meta-data"; fi
}

# Function to get all VM configurations
get_vm_list() {
    find "$VM_DIR" -name "*.conf" -exec basename {} .conf \; 2>/dev/null | sort
}

# Function to get Proxmox VMs
get_proxmox_vms() {
    if [[ -n "$PROXMOX_HOST" ]] && command -v pvesh &>/dev/null; then
        pvesh get /nodes/$PROXMOX_NODE/qemu --no-header 2>/dev/null | jq -r '.[].vmid' 2>/dev/null || echo ""
    fi
}

# Function to load VM configuration
load_vm_config() {
    local vm_name=$1
    local config_file="$VM_DIR/$vm_name.conf"
    
    if [[ -f "$config_file" ]]; then
        # Clear previous variables
        unset VM_NAME OS_TYPE CODENAME IMG_URL HOSTNAME USERNAME PASSWORD
        unset DISK_SIZE MEMORY CPUS SSH_PORT GUI_MODE PORT_FORWARDS IMG_FILE SEED_FILE CREATED USE_BRIDGE BRIDGE_IF
        unset PROXMOX_VMID PROXMOX_ENABLED
        
        source "$config_file"
        return 0
    else
        print_status "ERROR" "📂 Configuration for VM '$vm_name' not found"
        return 1
    fi
}

# Function to save VM configuration
save_vm_config() {
    local config_file="$VM_DIR/$VM_NAME.conf"
    
    cat > "$config_file" <<EOF
VM_NAME="$VM_NAME"
OS_TYPE="$OS_TYPE"
CODENAME="$CODENAME"
IMG_URL="$IMG_URL"
HOSTNAME="$HOSTNAME"
USERNAME="$USERNAME"
PASSWORD="$PASSWORD"
DISK_SIZE="$DISK_SIZE"
MEMORY="$MEMORY"
CPUS="$CPUS"
SSH_PORT="$SSH_PORT"
GUI_MODE="$GUI_MODE"
PORT_FORWARDS="$PORT_FORWARDS"
IMG_FILE="$IMG_FILE"
SEED_FILE="$SEED_FILE"
CREATED="$CREATED"
USE_BRIDGE="${USE_BRIDGE:-false}"
BRIDGE_IF="${BRIDGE_IF:-br0}"
PROXMOX_VMID="${PROXMOX_VMID:-}"
PROXMOX_ENABLED="${PROXMOX_ENABLED:-false}"
EOF
    
    print_status "SUCCESS" "💾 Configuration saved to $config_file"
}

# Function to download image with retry
download_image() {
    local url="$1"
    local output_file="$2"
    
    print_status "INFO" "🌐 Downloading image from $url..."
    
    # Try wget first
    if wget --progress=bar:force --no-check-certificate -q --show-progress "$url" -O "$output_file.tmp"; then
        mv "$output_file.tmp" "$output_file"
        return 0
    fi
    
    # Try curl if wget fails
    print_status "WARN" "⚠️  wget failed, trying curl..."
    if curl -L --progress-bar "$url" -o "$output_file.tmp"; then
        mv "$output_file.tmp" "$output_file"
        return 0
    fi
    
    # Try without progress if both fail
    print_status "WARN" "⚠️  curl failed, trying simple download..."
    if wget --no-check-certificate -q "$url" -O "$output_file.tmp"; then
        mv "$output_file.tmp" "$output_file"
        return 0
    fi
    
    return 1
}

# Function to create new VM
create_new_vm() {
    print_status "INFO" "🆕 Creating a new VM"
    
    # OS Selection
    print_status "INFO" "🌍 Select an OS to set up:"
    local os_options=()
    local i=1
    for os in "${!OS_OPTIONS[@]}"; do
        echo "  $i) $os"
        os_options[$i]="$os"
        ((i++))
    done
    
    while true; do
        read -p "$(print_status "INPUT" "🎯 Enter your choice (1-${#OS_OPTIONS[@]}): ")" choice
        if [[ "$choice" =~ ^[0-9]+$ ]] && [ "$choice" -ge 1 ] && [ "$choice" -le ${#OS_OPTIONS[@]} ]; then
            local os="${os_options[$choice]}"
            IFS='|' read -r OS_TYPE CODENAME IMG_URL DEFAULT_HOSTNAME DEFAULT_USERNAME DEFAULT_PASSWORD <<< "${OS_OPTIONS[$os]}"
            break
        else
            print_status "ERROR" "❌ Invalid selection. Try again."
        fi
    done

    # Custom Inputs with validation
    while true; do
        read -p "$(print_status "INPUT" "🏷️  Enter VM name (default: $DEFAULT_HOSTNAME): ")" VM_NAME
        VM_NAME="${VM_NAME:-$DEFAULT_HOSTNAME}"
        if validate_input "name" "$VM_NAME"; then
            # Check if VM name already exists
            if [[ -f "$VM_DIR/$VM_NAME.conf" ]]; then
                print_status "ERROR" "⚠️  VM with name '$VM_NAME' already exists"
            else
                break
            fi
        fi
    done

    while true; do
        read -p "$(print_status "INPUT" "🏠 Enter hostname (default: $VM_NAME): ")" HOSTNAME
        HOSTNAME="${HOSTNAME:-$VM_NAME}"
        if validate_input "name" "$HOSTNAME"; then
            break
        fi
    done

    while true; do
        read -p "$(print_status "INPUT" "👤 Enter username (default: $DEFAULT_USERNAME): ")" USERNAME
        USERNAME="${USERNAME:-$DEFAULT_USERNAME}"
        if validate_input "username" "$USERNAME"; then
            break
        fi
    done

    while true; do
        read -s -p "$(print_status "INPUT" "🔑 Enter password (default: $DEFAULT_PASSWORD): ")" PASSWORD
        PASSWORD="${PASSWORD:-$DEFAULT_PASSWORD}"
        echo
        if [ -n "$PASSWORD" ]; then
            break
        else
            print_status "ERROR" "❌ Password cannot be empty"
        fi
    done

    while true; do
        read -p "$(print_status "INPUT" "💾 Disk size (default: 20G): ")" DISK_SIZE
        DISK_SIZE="${DISK_SIZE:-20G}"
        if validate_input "size" "$DISK_SIZE"; then
            break
        fi
    done

    while true; do
        read -p "$(print_status "INPUT" "🧠 Memory in MB (default: 2048): ")" MEMORY
        MEMORY="${MEMORY:-2048}"
        if validate_input "number" "$MEMORY"; then
            break
        fi
    done

    while true; do
        read -p "$(print_status "INPUT" "⚡ Number of CPUs (default: 2): ")" CPUS
        CPUS="${CPUS:-2}"
        if validate_input "number" "$CPUS"; then
            break
        fi
    done

    while true; do
        read -p "$(print_status "INPUT" "🔌 SSH Port (default: 2222): ")" SSH_PORT
        SSH_PORT="${SSH_PORT:-2222}"
        if validate_input "port" "$SSH_PORT"; then
            # Check if port is already in use
            if ss -tln 2>/dev/null | grep -q ":$SSH_PORT "; then
                print_status "ERROR" "🚫 Port $SSH_PORT is already in use"
            else
                break
            fi
        fi
    done

    while true; do
        read -p "$(print_status "INPUT" "🖥️  Enable GUI mode? (y/n, default: n): ")" gui_input
        GUI_MODE=false
        gui_input="${gui_input:-n}"
        if [[ "$gui_input" =~ ^[Yy]$ ]]; then 
            GUI_MODE=true
            break
        elif [[ "$gui_input" =~ ^[Nn]$ ]]; then
            break
        else
            print_status "ERROR" "❌ Please answer y or n"
        fi
    done

    # Network options
    read -p "$(print_status "INPUT" "🌐 Use bridge networking? (requires sudo, y/N): ")" use_bridge
    USE_BRIDGE=false
    if [[ "$use_bridge" =~ ^[Yy]$ ]]; then
        USE_BRIDGE=true
        read -p "$(print_status "INPUT" "🔌 Bridge interface (default: br0): ")" BRIDGE_IF
        BRIDGE_IF="${BRIDGE_IF:-br0}"
    fi

    # Proxmox integration
    PROXMOX_ENABLED=false
    if [[ -n "$PROXMOX_HOST" ]]; then
        read -p "$(print_status "INPUT" "☁️  Create on Proxmox too? (y/N): ")" proxmox_create
        if [[ "$proxmox_create" =~ ^[Yy]$ ]]; then
            PROXMOX_ENABLED=true
            while true; do
                read -p "$(print_status "INPUT" "🔢 Proxmox VMID (100-999999999): ")" PROXMOX_VMID
                if validate_input "vmid" "$PROXMOX_VMID"; then
                    # Check if VMID already exists
                    if command -v pvesh &>/dev/null; then
                        if pvesh get /nodes/$PROXMOX_NODE/qemu/$PROXMOX_VMID/status --no-header 2>/dev/null; then
                            print_status "ERROR" "⚠️  VMID $PROXMOX_VMID already exists on Proxmox"
                        else
                            break
                        fi
                    else
                        break
                    fi
                fi
            done
        fi
    fi

    # Additional network options
    read -p "$(print_status "INPUT" "🌐 Additional port forwards (e.g., 8080:80, press Enter for none): ")" PORT_FORWARDS

    IMG_FILE="$VM_DIR/$VM_NAME.img"
    SEED_FILE="$VM_DIR/$VM_NAME-seed.iso"
    CREATED="$(date)"

    # Download and setup VM image
    setup_vm_image
    
    # Create on Proxmox if enabled
    if [[ "$PROXMOX_ENABLED" == true ]]; then
        create_proxmox_vm
    fi
    
    # Save configuration
    save_vm_config
}

# Function to setup VM image
setup_vm_image() {
    print_status "INFO" "📥 Downloading and preparing image..."
    
    # Create VM directory if it doesn't exist
    mkdir -p "$VM_DIR"
    
    # Check if image already exists
    if [[ -f "$IMG_FILE" ]]; then
        print_status "INFO" "✅ Image file already exists. Skipping download."
    else
        if ! download_image "$IMG_URL" "$IMG_FILE"; then
            print_status "ERROR" "❌ Failed to download image from $IMG_URL"
            # Try alternative URL for Debian daily builds
            if [[ "$CODENAME" == "trixie" ]]; then
                print_status "INFO" "🔄 Trying alternative Debian 13 URL..."
                ALT_URL="https://cloud.debian.org/images/cloud/trixie/latest/debian-13-generic-amd64.qcow2"
                if download_image "$ALT_URL" "$IMG_FILE"; then
                    IMG_URL="$ALT_URL"
                else
                    exit 1
                fi
            else
                exit 1
            fi
        fi
    fi
    
    # Check image format
    if ! qemu-img info "$IMG_FILE" 2>/dev/null | grep -q "file format:"; then
        print_status "WARN" "⚠️  Image format detection failed, attempting conversion..."
        if qemu-img convert -f raw -O qcow2 "$IMG_FILE" "$IMG_FILE.converted" 2>/dev/null; then
            mv "$IMG_FILE.converted" "$IMG_FILE"
        fi
    fi
    
    # Resize the disk image if needed
    if ! qemu-img resize "$IMG_FILE" "$DISK_SIZE" 2>/dev/null; then
        print_status "WARN" "⚠️  Failed to resize disk image. Creating new image with specified size..."
        # Create a new image with the specified size
        rm -f "$IMG_FILE"
        qemu-img create -f qcow2 "$IMG_FILE" "$DISK_SIZE"
    fi

    # cloud-init configuration
    cat > user-data <<EOF
#cloud-config
hostname: $HOSTNAME
ssh_pwauth: true
disable_root: false
users:
  - name: $USERNAME
    sudo: ALL=(ALL) NOPASSWD:ALL
    shell: /bin/bash
    password: $(openssl passwd -6 "$PASSWORD" | tr -d '\n')
chpasswd:
  list: |
    root:$PASSWORD
    $USERNAME:$PASSWORD
  expire: false
EOF

    cat > meta-data <<EOF
instance-id: iid-$VM_NAME
local-hostname: $HOSTNAME
EOF

    # Create seed image using available tool
    if command -v cloud-localds &> /dev/null; then
        cloud-localds "$SEED_FILE" user-data meta-data
    elif command -v genisoimage &> /dev/null; then
        genisoimage -output "$SEED_FILE" -volid cidata -joliet -rock user-data meta-data
    else
        print_status "ERROR" "❌ No tool available to create seed image"
        exit 1
    fi
    
    if [[ $? -eq 0 ]]; then
        print_status "SUCCESS" "🎉 VM '$VM_NAME' created successfully."
        print_status "INFO" "🔑 Login with: username=$USERNAME, password=$PASSWORD"
        print_status "INFO" "🔌 SSH: ssh -p $SSH_PORT $USERNAME@localhost"
    else
        print_status "ERROR" "❌ Failed to create cloud-init seed image"
        exit 1
    fi
}

# Function to create VM on Proxmox
create_proxmox_vm() {
    print_status "INFO" "☁️  Creating VM on Proxmox..."
    
    if [[ -z "$PROXMOX_HOST" ]] || [[ -z "$PROXMOX_VMID" ]]; then
        print_status "ERROR" "❌ Proxmox configuration incomplete"
        return 1
    fi
    
    # Convert disk size to GB for Proxmox
    local disk_size_gb=$(echo "$DISK_SIZE" | sed 's/[^0-9]*//g')
    if [[ "$DISK_SIZE" =~ G$ ]]; then
        disk_size_gb=$((disk_size_gb))
    elif [[ "$DISK_SIZE" =~ M$ ]]; then
        disk_size_gb=$((disk_size_gb / 1024))
    fi
    
    # Convert memory to MB for Proxmox
    local memory_mb=$MEMORY
    
    # Prepare image for Proxmox
    local proxmox_image="$PROXMOX_DIR/$VM_NAME.qcow2"
    cp "$IMG_FILE" "$proxmox_image"
    
    # Create Proxmox VM using qm command
    if command -v qm &>/dev/null; then
        # Create VM
        qm create $PROXMOX_VMID --name $VM_NAME --memory $memory_mb --cores $CPUS \
            --net0 virtio,bridge=vmbr0 --scsihw virtio-scsi-pci
        
        # Import disk
        qm importdisk $PROXMOX_VMID "$proxmox_image" $STORAGE --format qcow2
        
        # Attach disk
        qm set $PROXMOX_VMID --scsi0 $STORAGE:vm-${PROXMOX_VMID}-disk-0
        
        # Set boot order
        qm set $PROXMOX_VMID --boot order=scsi0
        
        # Add cloud-init drive
        qm set $PROXMOX_VMID --ide2 $STORAGE:cloudinit
        
        # Configure cloud-init
        qm set $PROXMOX_VMID --cipassword "$PASSWORD" --ciuser "$USERNAME" \
            --ipconfig0 ip=dhcp --searchdomain "local" --nameserver "8.8.8.8"
        
        # Start VM if desired
        read -p "$(print_status "INPUT" "🚀 Start VM on Proxmox now? (y/N): ")" start_now
        if [[ "$start_now" =~ ^[Yy]$ ]]; then
            qm start $PROXMOX_VMID
            print_status "SUCCESS" "✅ VM $VM_NAME (ID: $PROXMOX_VMID) started on Proxmox"
        else
            print_status "INFO" "💤 VM $VM_NAME (ID: $PROXMOX_VMID) created on Proxmox but not started"
        fi
    else
        print_status "WARN" "⚠️  qm command not found. Creating Proxmox VM configuration file instead."
        
        # Create Proxmox VM configuration file
        local proxmox_conf="$PROXMOX_DIR/$VM_NAME.conf"
        cat > "$proxmox_conf" <<EOF
# Proxmox VM Configuration for $VM_NAME
VMID: $PROXMOX_VMID
name: $VM_NAME
memory: $memory_mb
cores: $CPUS
net0: virtio,bridge=vmbr0
scsi0: $STORAGE:vm-${PROXMOX_VMID}-disk-0,size=${disk_size_gb}G
ide2: $STORAGE:cloudinit
boot: order=scsi0
ostype: l26
smbios1: uuid=$(uuidgen)
EOF
        
        print_status "INFO" "📄 Proxmox configuration saved to $proxmox_conf"
        print_status "INFO" "💡 Import manually using: qm importovf $PROXMOX_VMID $proxmox_image $STORAGE"
    fi
}

# Function to start a VM (Pure QEMU version without KVM)
start_vm() {
    local vm_name=$1
    
    if load_vm_config "$vm_name"; then
        # Check if Proxmox VM should be started
        if [[ "$PROXMOX_ENABLED" == true ]] && [[ -n "$PROXMOX_VMID" ]]; then
            print_status "INFO" "☁️  Starting Proxmox VM: $vm_name (ID: $PROXMOX_VMID)"
            if command -v qm &>/dev/null; then
                qm start $PROXMOX_VMID
                print_status "SUCCESS" "✅ Proxmox VM started"
                print_status "INFO" "🌐 Check Proxmox web interface: https://$PROXMOX_HOST:8006"
                return 0
            fi
        fi
        
        # Continue with local QEMU VM
        # Check if image is already in use
        if ! check_image_lock "$IMG_FILE" "$vm_name"; then
            print_status "ERROR" "🔒 Cannot start VM: Image file is locked by another process"
            read -p "$(print_status "INPUT" "🔄 Do you want to force kill all QEMU processes using this image? (y/N): ")" force_kill
            if [[ "$force_kill" =~ ^[Yy]$ ]]; then
                pkill -f "qemu-system.*$IMG_FILE"
                sleep 2
                if pgrep -f "qemu-system.*$IMG_FILE" >/dev/null; then
                    pkill -9 -f "qemu-system.*$IMG_FILE"
                fi
                print_status "SUCCESS" "✅ Terminated processes using the image"
                # Remove any lock files
                rm -f "${IMG_FILE}.lock" 2>/dev/null
            else
                return 1
            fi
        fi
        
        # Check if VM is already running
        if is_vm_running "$vm_name"; then
            print_status "WARN" "⚠️  VM '$vm_name' is already running"
            read -p "$(print_status "INPUT" "🔄 Stop and restart? (y/N): ")" restart_choice
            if [[ "$restart_choice" =~ ^[Yy]$ ]]; then
                stop_vm "$vm_name"
                sleep 2
            else
                return 1
            fi
        fi
        
        print_status "INFO" "🚀 Starting VM: $vm_name"
        print_status "INFO" "🔌 SSH: ssh -p $SSH_PORT $USERNAME@localhost"
        print_status "INFO" "🔑 Password: $PASSWORD"
        print_status "INFO" "🐌 Running in pure software emulation mode (no KVM)"
        
        # Check if image file exists
        if [[ ! -f "$IMG_FILE" ]]; then
            print_status "ERROR" "❌ VM image file not found: $IMG_FILE"
            return 1
        fi
        
        # Check if seed file exists
        if [[ ! -f "$SEED_FILE" ]]; then
            print_status "WARN" "⚠️  Seed file not found, recreating..."
            setup_vm_image
        fi
        
        # Pure QEMU command without KVM
        local qemu_cmd=(
            qemu-system-x86_64
            -m "$MEMORY"
            -smp "$CPUS"
            -cpu qemu64
            -machine type=pc,accel=tcg
            -drive "file=$IMG_FILE,format=qcow2,if=virtio"
            -drive "file=$SEED_FILE,format=raw,if=virtio"
            -boot order=c
            -device virtio-net-pci,netdev=n0
        )

        # Add networking based on configuration
        if [[ "$USE_BRIDGE" == true ]]; then
            print_status "INFO" "🔗 Using bridge networking with $BRIDGE_IF"
            qemu_cmd+=(-netdev "bridge,id=n0,br=$BRIDGE_IF")
        else
            qemu_cmd+=(-netdev "user,id=n0,hostfwd=tcp::$SSH_PORT-:22")
        fi

        # Add port forwards if specified (only for user mode networking)
        if [[ -n "$PORT_FORWARDS" ]] && [[ "$USE_BRIDGE" != true ]]; then
            IFS=',' read -ra forwards <<< "$PORT_FORWARDS"
            for forward in "${forwards[@]}"; do
                IFS=':' read -r host_port guest_port <<< "$forward"
                qemu_cmd+=(-device "virtio-net-pci,netdev=n${#qemu_cmd[@]}")
                qemu_cmd+=(-netdev "user,id=n${#qemu_cmd[@]},hostfwd=tcp::$host_port-:$guest_port")
            done
        fi

        # Add GUI or console mode
        if [[ "$GUI_MODE" == true ]]; then
            qemu_cmd+=(-vga virtio -display gtk,gl=on)
            print_status "INFO" "🖥️  Starting in GUI mode..."
        else
            qemu_cmd+=(-nographic -serial mon:stdio)
            print_status "INFO" "📟 Starting in console mode..."
            print_status "INFO" "🛑 Press Ctrl+A then X to exit QEMU console"
        fi

        # Add performance enhancements for software emulation
        qemu_cmd+=(
            -device virtio-balloon-pci
            -object rng-random,filename=/dev/urandom,id=rng0
            -device virtio-rng-pci,rng=rng0
            -no-hpet
            -rtc base=utc,clock=host
        )

        print_status "INFO" "⚡ Starting QEMU in pure software emulation mode..."
        echo "📊 Configuration: ${MEMORY}MB RAM, ${CPUS} CPUs, ${DISK_SIZE} disk"
        echo "ℹ️  Note: For better performance, consider enabling KVM if your system supports it"
        
        # Start the VM
        if ! "${qemu_cmd[@]}"; then
            print_status "ERROR" "❌ Failed to start VM. There might be a problem with the image file or configuration."
            # Try to clean up lock files
            rm -f "${IMG_FILE}.lock" 2>/dev/null
            return 1
        fi
        
        print_status "INFO" "🛑 VM $vm_name has been shut down"
    fi
}

# Function to stop a VM
stop_vm() {
    local vm_name=$1
    
    if load_vm_config "$vm_name"; then
        # Stop Proxmox VM if enabled
        if [[ "$PROXMOX_ENABLED" == true ]] && [[ -n "$PROXMOX_VMID" ]]; then
            print_status "INFO" "☁️  Stopping Proxmox VM: $vm_name (ID: $PROXMOX_VMID)"
            if command -v qm &>/dev/null; then
                qm stop $PROXMOX_VMID
                print_status "SUCCESS" "✅ Proxmox VM stopped"
            fi
        fi
        
        # Stop local QEMU VM
        if is_vm_running "$vm_name"; then
            print_status "INFO" "🛑 Stopping VM: $vm_name"
            
            # Try graceful shutdown first
            pkill -f "qemu-system.*$IMG_FILE"
            sleep 2
            
            # Check if it stopped
            if is_vm_running "$vm_name"; then
                print_status "WARN" "⚠️  VM did not stop gracefully, forcing termination..."
                pkill -9 -f "qemu-system.*$IMG_FILE"
                sleep 1
            fi
            
            # Clean up lock files
            rm -f "${IMG_FILE}.lock" 2>/dev/null
            
            if is_vm_running "$vm_name"; then
                print_status "ERROR" "❌ Failed to stop VM"
                return 1
            else
                print_status "SUCCESS" "✅ VM $vm_name stopped"
            fi
        else
            print_status "INFO" "💤 VM $vm_name is not running"
            # Still try to clean up any lock files
            rm -f "${IMG_FILE}.lock" 2>/dev/null
        fi
    fi
}

# Function to check if VM is running
is_vm_running() {
    local vm_name=$1
    
    # Check Proxmox VM first
    if load_vm_config "$vm_name" 2>/dev/null; then
        if [[ "$PROXMOX_ENABLED" == true ]] && [[ -n "$PROXMOX_VMID" ]]; then
            if command -v qm &>/dev/null; then
                if qm status $PROXMOX_VMID 2>/dev/null | grep -q "status: running"; then
                    return 0
                fi
            fi
        fi
    fi
    
    # Check local QEMU VM
    # Multiple methods to check
    # Method 1: Check by process name
    if pgrep -f "qemu-system.*$vm_name" >/dev/null; then
        return 0
    fi
    
    # Method 2: Check by image file
    if load_vm_config "$vm_name" 2>/dev/null; then
        if [[ -f "$IMG_FILE" ]] && lsof "$IMG_FILE" 2>/dev/null | grep -q qemu-system; then
            return 0
        fi
        
        # Method 3: Check by port usage (only for user mode networking)
        if [[ "$USE_BRIDGE" != true ]] && ss -tln 2>/dev/null | grep -q ":$SSH_PORT "; then
            return 0
        fi
    fi
    
    return 1
}

# Function to delete a VM
delete_vm() {
    local vm_name=$1
    
    print_status "WARN" "⚠️  ⚠️  ⚠️  This will permanently delete VM '$vm_name' and all its data!"
    read -p "$(print_status "INPUT" "🗑️  Are you sure? (y/N): ")" -n 1 -r
    echo
    if [[ $REPLY =~ ^[Yy]$ ]]; then
        if load_vm_config "$vm_name"; then
            # Delete from Proxmox if enabled
            if [[ "$PROXMOX_ENABLED" == true ]] && [[ -n "$PROXMOX_VMID" ]]; then
                print_status "INFO" "☁️  Deleting Proxmox VM: $vm_name (ID: $PROXMOX_VMID)"
                if command -v qm &>/dev/null; then
                    qm stop $PROXMOX_VMID 2>/dev/null
                    qm destroy $PROXMOX_VMID --purge
                    print_status "SUCCESS" "✅ Proxmox VM deleted"
                fi
            fi
            
            # Check if VM is running
            if is_vm_running "$vm_name"; then
                print_status "WARN" "⚠️  VM is currently running. Stopping it first..."
                stop_vm "$vm_name"
                sleep 2
            fi
            
            rm -f "$IMG_FILE" "$SEED_FILE" "$VM_DIR/$vm_name.conf" "${IMG_FILE}.lock" 2>/dev/null
            rm -f "$PROXMOX_DIR/$vm_name.qcow2" "$PROXMOX_DIR/$vm_name.conf" 2>/dev/null
            print_status "SUCCESS" "✅ VM '$vm_name' has been deleted"
        fi
    else
        print_status "INFO" "👍 Deletion cancelled"
    fi
}

# Function to show VM info
show_vm_info() {
    local vm_name=$1
    
    if load_vm_config "$vm_name"; then
        echo
        print_status "INFO" "📊 VM Information: $vm_name"
        echo "🔹🔹🔹🔹🔹🔹🔹🔹🔹🔹🔹🔹🔹🔹🔹🔹🔹🔹🔹🔹"
        echo "🌍 OS: $OS_TYPE"
        echo "🏷️  Hostname: $HOSTNAME"
        echo "👤 Username: $USERNAME"
        echo "🔑 Password: $PASSWORD"
        echo "🔌 SSH Port: $SSH_PORT"
        echo "🧠 Memory: $MEMORY MB"
        echo "⚡ CPUs: $CPUS"
        echo "💾 Disk: $DISK_SIZE"
        echo "🖥️  GUI Mode: $GUI_MODE"
        echo "🌐 Network: $([ "$USE_BRIDGE" = true ] && echo "Bridge ($BRIDGE_IF)" || echo "User mode")"
        echo "🌐 Port Forwards: ${PORT_FORWARDS:-None}"
        echo "📅 Created: $CREATED"
        echo "💿 Image File: $IMG_FILE"
        echo "🌱 Seed File: $SEED_FILE"
        
        if [[ "$PROXMOX_ENABLED" == true ]]; then
            echo "☁️  Proxmox: Enabled"
            echo "🔢 Proxmox VMID: $PROXMOX_VMID"
            if command -v qm &>/dev/null; then
                local proxmox_status=$(qm status $PROXMOX_VMID 2>/dev/null | grep "status:" | cut -d: -f2 | xargs)
                echo "📊 Proxmox Status: ${proxmox_status:-Unknown}"
            fi
        else
            echo "☁️  Proxmox: Not configured"
        fi
        
        # Show lock status
        if check_image_lock "$IMG_FILE" "$vm_name" >/dev/null 2>&1; then
            echo "🔓 Image Status: Unlocked"
        else
            echo "🔒 Image Status: Locked (possibly in use)"
        fi
        
        # Show if VM is running
        if is_vm_running "$vm_name"; then
            echo "🚀 Status: Running"
        else
            echo "💤 Status: Stopped"
        fi
        
        echo "🔹🔹🔹🔹🔹🔹🔹🔹🔹🔹🔹🔹🔹🔹🔹🔹🔹🔹🔹🔹"
        echo
        read -p "$(print_status "INPUT" "⏎ Press Enter to continue...")"
    fi
}

# Function to migrate VM to Proxmox
migrate_to_proxmox() {
    local vm_name=$1
    
    if load_vm_config "$vm_name"; then
        print_status "INFO" "🔄 Migrating VM to Proxmox: $vm_name"
        
        if [[ -z "$PROXMOX_HOST" ]]; then
            print_status "ERROR" "❌ Proxmox not configured. Please configure Proxmox first."
            configure_proxmox
            if [[ -z "$PROXMOX_HOST" ]]; then
                return 1
            fi
        fi
        
        # Get VMID
        while true; do
            read -p "$(print_status "INPUT" "🔢 Enter Proxmox VMID (100-999999999): ")" PROXMOX_VMID
            if validate_input "vmid" "$PROXMOX_VMID"; then
                # Check if VMID already exists
                if command -v pvesh &>/dev/null; then
                    if pvesh get /nodes/$PROXMOX_NODE/qemu/$PROXMOX_VMID/status --no-header 2>/dev/null; then
                        print_status "ERROR" "⚠️  VMID $PROXMOX_VMID already exists on Proxmox"
                    else
                        break
                    fi
                else
                    break
                fi
            fi
        done
        
        PROXMOX_ENABLED=true
        create_proxmox_vm
        
        # Update configuration
        save_vm_config
        
        print_status "SUCCESS" "✅ VM migrated to Proxmox"
        print_status "INFO" "🌐 Check Proxmox web interface: https://$PROXMOX_HOST:8006"
    fi
}

# Function to sync with Proxmox
sync_with_proxmox() {
    local vm_name=$1
    
    if load_vm_config "$vm_name"; then
        if [[ "$PROXMOX_ENABLED" != true ]] || [[ -z "$PROXMOX_VMID" ]]; then
            print_status "ERROR" "❌ VM is not configured for Proxmox"
            read -p "$(print_status "INPUT" "🔄 Migrate to Proxmox now? (y/N): ")" migrate_now
            if [[ "$migrate_now" =~ ^[Yy]$ ]]; then
                migrate_to_proxmox "$vm_name"
            fi
            return 1
        fi
        
        print_status "INFO" "🔄 Syncing VM with Proxmox: $vm_name"
        
        if command -v qm &>/dev/null; then
            # Get current Proxmox VM status
            local proxmox_status=$(qm status $PROXMOX_VMID 2>/dev/null)
            
            if [[ -n "$proxmox_status" ]]; then
                print_status "INFO" "📊 Proxmox VM Status:"
                echo "$proxmox_status"
                
                # Sync configuration
                read -p "$(print_status "INPUT" "🔄 Sync configuration from Proxmox? (y/N): ")" sync_config
                if [[ "$sync_config" =~ ^[Yy]$ ]]; then
                    # Get VM config from Proxmox
                    local vm_config=$(qm config $PROXMOX_VMID 2>/dev/null)
                    
                    # Update memory if different
                    local proxmox_memory=$(echo "$vm_config" | grep "memory:" | awk '{print $2}')
                    if [[ -n "$proxmox_memory" ]] && [[ "$proxmox_memory" != "$MEMORY" ]]; then
                        MEMORY="$proxmox_memory"
                        print_status "INFO" "🔄 Updated memory to $MEMORY MB"
                    fi
                    
                    # Update CPUs if different
                    local proxmox_cores=$(echo "$vm_config" | grep "cores:" | awk '{print $2}')
                    if [[ -n "$proxmox_cores" ]] && [[ "$proxmox_cores" != "$CPUS" ]]; then
                        CPUS="$proxmox_cores"
                        print_status "INFO" "🔄 Updated CPUs to $CPUS"
                    fi
                    
                    save_vm_config
                fi
                
                # Start/stop VM based on Proxmox state
                local status=$(echo "$proxmox_status" | grep "status:" | awk '{print $2}')
                if [[ "$status" == "running" ]] && ! is_vm_running "$vm_name"; then
                    read -p "$(print_status "INPUT" "🚀 Proxmox VM is running. Start local VM too? (y/N): ")" start_local
                    if [[ "$start_local" =~ ^[Yy]$ ]]; then
                        start_vm "$vm_name"
                    fi
                fi
            else
                print_status "ERROR" "❌ Could not get Proxmox VM status"
            fi
        else
            print_status "WARN" "⚠️  qm command not found"
        fi
    fi
}

# Function to edit VM configuration
edit_vm_config() {
    local vm_name=$1
    
    if load_vm_config "$vm_name"; then
        print_status "INFO" "✏️  Editing VM: $vm_name"
        
        while true; do
            echo "📝 What would you like to edit?"
            echo "  1) 🏷️  Hostname"
            echo "  2) 👤 Username"
            echo "  3) 🔑 Password"
            echo "  4) 🔌 SSH Port"
            echo "  5) 🖥️  GUI Mode"
            echo "  6) 🌐 Port Forwards"
            echo "  7) 🧠 Memory (RAM)"
            echo "  8) ⚡ CPU Count"
            echo "  9) 💾 Disk Size"
            echo "  10) 🔗 Network Mode"
            echo "  11) ☁️  Proxmox VMID"
            echo "  0) ↩️  Back to main menu"
            
            read -p "$(print_status "INPUT" "🎯 Enter your choice: ")" edit_choice
            
            case $edit_choice in
                1)
                    while true; do
                        read -p "$(print_status "INPUT" "🏷️  Enter new hostname (current: $HOSTNAME): ")" new_hostname
                        new_hostname="${new_hostname:-$HOSTNAME}"
                        if validate_input "name" "$new_hostname"; then
                            HOSTNAME="$new_hostname"
                            break
                        fi
                    done
                    ;;
                2)
                    while true; do
                        read -p "$(print_status "INPUT" "👤 Enter new username (current: $USERNAME): ")" new_username
                        new_username="${new_username:-$USERNAME}"
                        if validate_input "username" "$new_username"; then
                            USERNAME="$new_username"
                            break
                        fi
                    done
                    ;;
                3)
                    while true; do
                        read -s -p "$(print_status "INPUT" "🔑 Enter new password (current: ****): ")" new_password
                        new_password="${new_password:-$PASSWORD}"
                        echo
                        if [ -n "$new_password" ]; then
                            PASSWORD="$new_password"
                            break
                        else
                            print_status "ERROR" "❌ Password cannot be empty"
                        fi
                    done
                    ;;
                4)
                    while true; do
                        read -p "$(print_status "INPUT" "🔌 Enter new SSH port (current: $SSH_PORT): ")" new_ssh_port
                        new_ssh_port="${new_ssh_port:-$SSH_PORT}"
                        if validate_input "port" "$new_ssh_port"; then
                            # Check if port is already in use
                            if [ "$new_ssh_port" != "$SSH_PORT" ] && ss -tln 2>/dev/null | grep -q ":$new_ssh_port "; then
                                print_status "ERROR" "🚫 Port $new_ssh_port is already in use"
                            else
                                SSH_PORT="$new_ssh_port"
                                break
                            fi
                        fi
                    done
                    ;;
                5)
                    while true; do
                        read -p "$(print_status "INPUT" "🖥️  Enable GUI mode? (y/n, current: $GUI_MODE): ")" gui_input
                        gui_input="${gui_input:-}"
                        if [[ "$gui_input" =~ ^[Yy]$ ]]; then 
                            GUI_MODE=true
                            break
                        elif [[ "$gui_input" =~ ^[Nn]$ ]]; then
                            GUI_MODE=false
                            break
                        elif [ -z "$gui_input" ]; then
                            # Keep current value if user just pressed Enter
                            break
                        else
                            print_status "ERROR" "❌ Please answer y or n"
                        fi
                    done
                    ;;
                6)
                    read -p "$(print_status "INPUT" "🌐 Additional port forwards (current: ${PORT_FORWARDS:-None}): ")" new_port_forwards
                    PORT_FORWARDS="${new_port_forwards:-$PORT_FORWARDS}"
                    ;;
                7)
                    while true; do
                        read -p "$(print_status "INPUT" "🧠 Enter new memory in MB (current: $MEMORY): ")" new_memory
                        new_memory="${new_memory:-$MEMORY}"
                        if validate_input "number" "$new_memory"; then
                            MEMORY="$new_memory"
                            break
                        fi
                    done
                    ;;
                8)
                    while true; do
                        read -p "$(print_status "INPUT" "⚡ Enter new CPU count (current: $CPUS): ")" new_cpus
                        new_cpus="${new_cpus:-$CPUS}"
                        if validate_input "number" "$new_cpus"; then
                            CPUS="$new_cpus"
                            break
                        fi
                    done
                    ;;
                9)
                    while true; do
                        read -p "$(print_status "INPUT" "💾 Enter new disk size (current: $DISK_SIZE): ")" new_disk_size
                        new_disk_size="${new_disk_size:-$DISK_SIZE}"
                        if validate_input "size" "$new_disk_size"; then
                            DISK_SIZE="$new_disk_size"
                            break
                        fi
                    done
                    ;;
                10)
                    read -p "$(print_status "INPUT" "🔗 Use bridge networking? (current: $USE_BRIDGE, y/N): ")" new_bridge
                    if [[ "$new_bridge" =~ ^[Yy]$ ]]; then
                        USE_BRIDGE=true
                        read -p "$(print_status "INPUT" "🔌 Bridge interface (current: $BRIDGE_IF): ")" new_bridge_if
                        BRIDGE_IF="${new_bridge_if:-$BRIDGE_IF}"
                    elif [[ "$new_bridge" =~ ^[Nn]$ ]]; then
                        USE_BRIDGE=false
                    fi
                    ;;
                11)
                    if [[ "$PROXMOX_ENABLED" == true ]]; then
                        while true; do
                            read -p "$(print_status "INPUT" "🔢 Enter new Proxmox VMID (current: $PROXMOX_VMID): ")" new_vmid
                            new_vmid="${new_vmid:-$PROXMOX_VMID}"
                            if validate_input "vmid" "$new_vmid"; then
                                PROXMOX_VMID="$new_vmid"
                                break
                            fi
                        done
                    else
                        print_status "INFO" "ℹ️  Proxmox not enabled for this VM"
                        read -p "$(print_status "INPUT" "🔄 Enable Proxmox integration? (y/N): ")" enable_proxmox
                        if [[ "$enable_proxmox" =~ ^[Yy]$ ]]; then
                            migrate_to_proxmox "$vm_name"
                        fi
                    fi
                    ;;
                0)
                    return 0
                    ;;
                *)
                    print_status "ERROR" "❌ Invalid selection"
                    continue
                    ;;
            esac
            
            # Recreate seed image with new configuration if user/password/hostname changed
            if [[ "$edit_choice" -eq 1 || "$edit_choice" -eq 2 || "$edit_choice" -eq 3 ]]; then
                print_status "INFO" "🔄 Updating cloud-init configuration..."
                setup_vm_image
            fi
            
            # Save configuration
            save_vm_config
            
            read -p "$(print_status "INPUT" "🔄 Continue editing? (y/N): ")" continue_editing
            if [[ ! "$continue_editing" =~ ^[Yy]$ ]]; then
                break
            fi
        done
    fi
}

# Function to resize VM disk
resize_vm_disk() {
    local vm_name=$1
    
    if load_vm_config "$vm_name"; then
        # Check if VM is running
        if is_vm_running "$vm_name"; then
            print_status "ERROR" "❌ Cannot resize disk while VM is running. Please stop the VM first."
            return 1
        fi
        
        print_status "INFO" "💾 Current disk size: $DISK_SIZE"
        
        while true; do
            read -p "$(print_status "INPUT" "📈 Enter new disk size (e.g., 50G): ")" new_disk_size
            if validate_input "size" "$new_disk_size"; then
                if [[ "$new_disk_size" == "$DISK_SIZE" ]]; then
                    print_status "INFO" "ℹ️  New disk size is the same as current size. No changes made."
                    return 0
                fi
                
                # Check if new size is smaller than current (not recommended)
                local current_size_num=${DISK_SIZE%[GgMm]}
                local new_size_num=${new_disk_size%[GgMm]}
                local current_unit=${DISK_SIZE: -1}
                local new_unit=${new_disk_size: -1}
                
                # Convert both to MB for comparison
                if [[ "$current_unit" =~ [Gg] ]]; then
                    current_size_num=$((current_size_num * 1024))
                fi
                if [[ "$new_unit" =~ [Gg] ]]; then
                    new_size_num=$((new_size_num * 1024))
                fi
                
                if [[ $new_size_num -lt $current_size_num ]]; then
                    print_status "WARN" "⚠️  Shrinking disk size is not recommended and may cause data loss!"
                    read -p "$(print_status "INPUT" "⚠️  Are you sure you want to continue? (y/N): ")" confirm_shrink
                    if [[ ! "$confirm_shrink" =~ ^[Yy]$ ]]; then
                        print_status "INFO" "👍 Disk resize cancelled."
                        return 0
                    fi
                fi
                
                # Resize the disk
                print_status "INFO" "📈 Resizing disk to $new_disk_size..."
                if qemu-img resize "$IMG_FILE" "$new_disk_size"; then
                    DISK_SIZE="$new_disk_size"
                    save_vm_config
                    print_status "SUCCESS" "✅ Disk resized successfully to $new_disk_size"
                else
                    print_status "ERROR" "❌ Failed to resize disk"
                    return 1
                fi
                break
            fi
        done
    fi
}

# Function to backup VM
backup_vm() {
    local vm_name=$1
    
    if load_vm_config "$vm_name"; then
        local backup_dir="$VM_DIR/backups"
        mkdir -p "$backup_dir"
        local timestamp=$(date +%Y%m%d_%H%M%S)
        local backup_file="$backup_dir/${vm_name}_${timestamp}.tar.gz"
        
        print_status "INFO" "💾 Creating backup of VM: $vm_name"
        
        if is_vm_running "$vm_name"; then
            print_status "WARN" "⚠️  VM is running. For consistent backup, stop VM first."
            read -p "$(print_status "INPUT" "Continue anyway? (y/N): ")" continue_backup
            if [[ ! "$continue_backup" =~ ^[Yy]$ ]]; then
                return 1
            fi
        fi
        
        tar -czf "$backup_file" -C "$VM_DIR" \
            "$(basename "$IMG_FILE")" \
            "$(basename "$SEED_FILE")" \
            "$(basename "$VM_DIR/$vm_name.conf")" 2>/dev/null
        
        if [ $? -eq 0 ]; then
            print_status "SUCCESS" "✅ Backup created: $backup_file"
            echo "📦 Size: $(du -h "$backup_file" | cut -f1)"
        else
            print_status "ERROR" "❌ Backup failed"
        fi
    fi
}

# Function to restore VM from backup
restore_vm() {
    local backup_file=$1
    
    if [[ ! -f "$backup_file" ]]; then
        print_status "ERROR" "❌ Backup file not found: $backup_file"
        return 1
    fi
    
    print_status "INFO" "🔄 Restoring VM from backup: $backup_file"
    
    # Extract to temporary directory
    local temp_dir=$(mktemp -d)
    tar -xzf "$backup_file" -C "$temp_dir"
    
    # Find config file
    local config_file=$(find "$temp_dir" -name "*.conf" | head -1)
    if [[ -z "$config_file" ]]; then
        print_status "ERROR" "❌ No configuration file found in backup"
        rm -rf "$temp_dir"
        return 1
    fi
    
    # Load config to get VM name
    source "$config_file"
    
    # Check if VM already exists
    if [[ -f "$VM_DIR/$VM_NAME.conf" ]]; then
        print_status "WARN" "⚠️  VM '$VM_NAME' already exists"
        read -p "$(print_status "INPUT" "Overwrite existing VM? (y/N): ")" overwrite
        if [[ ! "$overwrite" =~ ^[Yy]$ ]]; then
            rm -rf "$temp_dir"
            return 1
        fi
        # Stop VM if running
        if is_vm_running "$VM_NAME"; then
            stop_vm "$VM_NAME"
        fi
    fi
    
    # Copy files to VM directory
    cp -r "$temp_dir"/* "$VM_DIR/" 2>/dev/null
    rm -rf "$temp_dir"
    
    print_status "SUCCESS" "✅ VM '$VM_NAME' restored from backup"
}

# Function to show VM performance metrics
show_vm_performance() {
    local vm_name=$1
    
    if load_vm_config "$vm_name"; then
        if is_vm_running "$vm_name"; then
            print_status "INFO" "📊 Performance metrics for VM: $vm_name"
            echo "📈📈📈📈📈📈📈📈📈📈📈📈📈📈📈"
            
            # Get QEMU process ID
            local qemu_pid=$(pgrep -f "qemu-system.*$IMG_FILE")
            if [[ -n "$qemu_pid" ]]; then
                # Show process stats
                echo "⚡ QEMU Process Stats:"
                ps -p "$qemu_pid" -o pid,%cpu,%mem,sz,rss,vsz,cmd --no-headers
                echo
                
                # Show memory usage
                echo "🧠 Memory Usage:"
                free -h
                echo
                
                # Show disk usage
                echo "💾 Disk Usage:"
                df -h "$IMG_FILE" 2>/dev/null || du -h "$IMG_FILE"
            else
                print_status "ERROR" "❌ Could not find QEMU process for VM $vm_name"
            fi
        else
            print_status "INFO" "💤 VM $vm_name is not running"
            echo "⚙️  Configuration:"
            echo "  🧠 Memory: $MEMORY MB"
            echo "  ⚡ CPUs: $CPUS"
            echo "  💾 Disk: $DISK_SIZE"
        fi
        echo "📈📈📈📈📈📈📈📈📈📈📈📈📈📈📈"
        read -p "$(print_status "INPUT" "⏎ Press Enter to continue...")"
    fi
}

# Function to fix VM issues
fix_vm_issues() {
    local vm_name=$1
    
    if load_vm_config "$vm_name"; then
        print_status "INFO" "🔧 Fixing issues for VM: $vm_name"
        
        echo "🔧 Select issue to fix:"
        echo "  1) 🔓 Remove lock files"
        echo "  2) 🗑️  Recreate seed image"
        echo "  3) 🔄 Recreate configuration"
        echo "  4) 💀 Kill stuck processes"
        echo "  5) 🔧 Check and repair disk image"
        echo "  6) ☁️  Sync with Proxmox"
        echo "  0) ↩️  Back"
        
        read -p "$(print_status "INPUT" "🎯 Enter your choice: ")" fix_choice
        
        case $fix_choice in
            1)
                print_status "INFO" "🔓 Removing lock files..."
                rm -f "${IMG_FILE}.lock" 2>/dev/null
                rm -f "${IMG_FILE}"*.lock 2>/dev/null
                print_status "SUCCESS" "✅ Lock files removed"
                ;;
            2)
                print_status "INFO" "🔄 Recreating seed image..."
                if [[ -f "$SEED_FILE" ]]; then
                    rm -f "$SEED_FILE"
                fi
                setup_vm_image
                print_status "SUCCESS" "✅ Seed image recreated"
                ;;
            3)
                print_status "INFO" "🔄 Recreating configuration..."
                save_vm_config
                print_status "SUCCESS" "✅ Configuration recreated"
                ;;
            4)
                print_status "INFO" "💀 Killing stuck processes..."
                pkill -f "qemu-system.*$IMG_FILE" 2>/dev/null
                sleep 1
                if pgrep -f "qemu-system.*$IMG_FILE" >/dev/null; then
                    pkill -9 -f "qemu-system.*$IMG_FILE" 2>/dev/null
                    print_status "SUCCESS" "✅ Forcefully killed stuck processes"
                else
                    print_status "INFO" "💤 No stuck processes found"
                fi
                ;;
            5)
                print_status "INFO" "🔧 Checking disk image..."
                if qemu-img check "$IMG_FILE" 2>/dev/null; then
                    print_status "SUCCESS" "✅ Disk image is healthy"
                else
                    print_status "WARN" "⚠️  Disk image may be corrupted"
                    read -p "$(print_status "INPUT" "Try to repair? (y/N): ")" repair
                    if [[ "$repair" =~ ^[Yy]$ ]]; then
                        qemu-img amend -f qcow2 "$IMG_FILE" -o "compat=1.1" 2>/dev/null
                        print_status "INFO" "🔄 Attempted repair. Please verify VM functionality."
                    fi
                fi
                ;;
            6)
                if [[ "$PROXMOX_ENABLED" == true ]]; then
                    sync_with_proxmox "$vm_name"
                else
                    print_status "INFO" "ℹ️  Proxmox not enabled for this VM"
                fi
                ;;
            0)
                return 0
                ;;
            *)
                print_status "ERROR" "❌ Invalid selection"
                ;;
        esac
    fi
}

# Function to show Proxmox VMs
show_proxmox_vms() {
    if [[ -z "$PROXMOX_HOST" ]]; then
        print_status "ERROR" "❌ Proxmox not configured"
        read -p "$(print_status "INPUT" "🔧 Configure Proxmox now? (y/N): ")" configure_now
        if [[ "$configure_now" =~ ^[Yy]$ ]]; then
            configure_proxmox
        fi
        return 1
    fi
    
    print_status "INFO" "☁️  Proxmox VMs on $PROXMOX_HOST"
    echo "🌐🌐🌐🌐🌐🌐🌐🌐🌐🌐🌐🌐🌐🌐"
    
    if command -v pvesh &>/dev/null; then
        pvesh get /nodes/$PROXMOX_NODE/qemu --no-header 2>/dev/null | jq -r '.[] | "\(.vmid) - \(.name) (\(.status))"' 2>/dev/null || \
        echo "Could not fetch VM list. Check Proxmox connection."
    elif command -v qm &>/dev/null; then
        qm list 2>/dev/null | grep -v "VMID" || echo "No VMs found or qm command failed"
    else
        print_status "ERROR" "❌ Proxmox tools not found"
        echo "💡 Install Proxmox tools:"
        echo "  Debian/Ubuntu: sudo apt install proxmox-ve"
        echo "  Or download from: https://www.proxmox.com/en/downloads"
    fi
    
    echo "🌐🌐🌐🌐🌐🌐🌐🌐🌐🌐🌐🌐🌐🌐"
    read -p "$(print_status "INPUT" "⏎ Press Enter to continue...")"
}

# Main menu function
main_menu() {
    while true; do
        display_header
        
        local vms=($(get_vm_list))
        local vm_count=${#vms[@]}
        
        if [ $vm_count -gt 0 ]; then
            print_status "INFO" "📁 Found $vm_count local VM(s):"
            for i in "${!vms[@]}"; do
                local status="💤"
                if is_vm_running "${vms[$i]}"; then
                    status="🚀"
                fi
                printf "  %2d) %s %s\n" $((i+1)) "${vms[$i]}" "$status"
            done
            echo
        fi
        
        if [[ -n "$PROXMOX_HOST" ]]; then
            print_status "INFO" "☁️  Proxmox: $PROXMOX_HOST"
        else
            print_status "INFO" "☁️  Proxmox: Not configured"
        fi
        
        echo "📋 Main Menu:"
        echo "  1) 🆕 Create a new VM"
        if [ $vm_count -gt 0 ]; then
            echo "  2) 🚀 Start a VM"
            echo "  3) 🛑 Stop a VM"
            echo "  4) 📊 Show VM info"
            echo "  5) ✏️  Edit VM configuration"
            echo "  6) 🗑️  Delete a VM"
            echo "  7) 📈 Resize VM disk"
            echo "  8) 💾 Backup VM"
            echo "  9) 🔄 Restore VM from backup"
            echo "  10) 📊 Show VM performance"
            echo "  11) 🔧 Fix VM issues"
            echo "  12) ☁️  Migrate to Proxmox"
            echo "  13) 🔄 Sync with Proxmox"
        fi
        echo "  14) 🔧 Configure Proxmox"
        echo "  15) ☁️  Show Proxmox VMs"
        echo "  16) 🛠️  Check system requirements"
        echo "  0) 👋 Exit"
        echo
        
        read -p "$(print_status "INPUT" "🎯 Enter your choice: ")" choice
        
        case $choice in
            1)
                create_new_vm
                ;;
            2)
                if [ $vm_count -gt 0 ]; then
                    read -p "$(print_status "INPUT" "🚀 Enter VM number to start: ")" vm_num
                    if [[ "$vm_num" =~ ^[0-9]+$ ]] && [ "$vm_num" -ge 1 ] && [ "$vm_num" -le $vm_count ]; then
                        start_vm "${vms[$((vm_num-1))]}"
                    else
                        print_status "ERROR" "❌ Invalid selection"
                    fi
                fi
                ;;
            3)
                if [ $vm_count -gt 0 ]; then
                    read -p "$(print_status "INPUT" "🛑 Enter VM number to stop: ")" vm_num
                    if [[ "$vm_num" =~ ^[0-9]+$ ]] && [ "$vm_num" -ge 1 ] && [ "$vm_num" -le $vm_count ]; then
                        stop_vm "${vms[$((vm_num-1))]}"
                    else
                        print_status "ERROR" "❌ Invalid selection"
                    fi
                fi
                ;;
            4)
                if [ $vm_count -gt 0 ]; then
                    read -p "$(print_status "INPUT" "📊 Enter VM number to show info: ")" vm_num
                    if [[ "$vm_num" =~ ^[0-9]+$ ]] && [ "$vm_num" -ge 1 ] && [ "$vm_num" -le $vm_count ]; then
                        show_vm_info "${vms[$((vm_num-1))]}"
                    else
                        print_status "ERROR" "❌ Invalid selection"
                    fi
                fi
                ;;
            5)
                if [ $vm_count -gt 0 ]; then
                    read -p "$(print_status "INPUT" "✏️  Enter VM number to edit: ")" vm_num
                    if [[ "$vm_num" =~ ^[0-9]+$ ]] && [ "$vm_num" -ge 1 ] && [ "$vm_num" -le $vm_count ]; then
                        edit_vm_config "${vms[$((vm_num-1))]}"
                    else
                        print_status "ERROR" "❌ Invalid selection"
                    fi
                fi
                ;;
            6)
                if [ $vm_count -gt 0 ]; then
                    read -p "$(print_status "INPUT" "🗑️  Enter VM number to delete: ")" vm_num
                    if [[ "$vm_num" =~ ^[0-9]+$ ]] && [ "$vm_num" -ge 1 ] && [ "$vm_num" -le $vm_count ]; then
                        delete_vm "${vms[$((vm_num-1))]}"
                    else
                        print_status "ERROR" "❌ Invalid selection"
                    fi
                fi
                ;;
            7)
                if [ $vm_count -gt 0 ]; then
                    read -p "$(print_status "INPUT" "📈 Enter VM number to resize disk: ")" vm_num
                    if [[ "$vm_num" =~ ^[0-9]+$ ]] && [ "$vm_num" -ge 1 ] && [ "$vm_num" -le $vm_count ]; then
                        resize_vm_disk "${vms[$((vm_num-1))]}"
                    else
                        print_status "ERROR" "❌ Invalid selection"
                    fi
                fi
                ;;
            8)
                if [ $vm_count -gt 0 ]; then
                    read -p "$(print_status "INPUT" "💾 Enter VM number to backup: ")" vm_num
                    if [[ "$vm_num" =~ ^[0-9]+$ ]] && [ "$vm_num" -ge 1 ] && [ "$vm_num" -le $vm_count ]; then
                        backup_vm "${vms[$((vm_num-1))]}"
                    else
                        print_status "ERROR" "❌ Invalid selection"
                    fi
                fi
                ;;
            9)
                if [ $vm_count -gt 0 ]; then
                    read -p "$(print_status "INPUT" "🔄 Enter backup file path: ")" backup_file
                    restore_vm "$backup_file"
                fi
                ;;
            10)
                if [ $vm_count -gt 0 ]; then
                    read -p "$(print_status "INPUT" "📊 Enter VM number to show performance: ")" vm_num
                    if [[ "$vm_num" =~ ^[0-9]+$ ]] && [ "$vm_num" -ge 1 ] && [ "$vm_num" -le $vm_count ]; then
                        show_vm_performance "${vms[$((vm_num-1))]}"
                    else
                        print_status "ERROR" "❌ Invalid selection"
                    fi
                fi
                ;;
            11)
                if [ $vm_count -gt 0 ]; then
                    read -p "$(print_status "INPUT" "🔧 Enter VM number to fix issues: ")" vm_num
                    if [[ "$vm_num" =~ ^[0-9]+$ ]] && [ "$vm_num" -ge 1 ] && [ "$vm_num" -le $vm_count ]; then
                        fix_vm_issues "${vms[$((vm_num-1))]}"
                    else
                        print_status "ERROR" "❌ Invalid selection"
                    fi
                fi
                ;;
            12)
                if [ $vm_count -gt 0 ]; then
                    read -p "$(print_status "INPUT" "☁️  Enter VM number to migrate to Proxmox: ")" vm_num
                    if [[ "$vm_num" =~ ^[0-9]+$ ]] && [ "$vm_num" -ge 1 ] && [ "$vm_num" -le $vm_count ]; then
                        migrate_to_proxmox "${vms[$((vm_num-1))]}"
                    else
                        print_status "ERROR" "❌ Invalid selection"
                    fi
                fi
                ;;
            13)
                if [ $vm_count -gt 0 ]; then
                    read -p "$(print_status "INPUT" "🔄 Enter VM number to sync with Proxmox: ")" vm_num
                    if [[ "$vm_num" =~ ^[0-9]+$ ]] && [ "$vm_num" -ge 1 ] && [ "$vm_num" -le $vm_count ]; then
                        sync_with_proxmox "${vms[$((vm_num-1))]}"
                    else
                        print_status "ERROR" "❌ Invalid selection"
                    fi
                fi
                ;;
            14)
                configure_proxmox
                ;;
            15)
                show_proxmox_vms
                ;;
            16)
                check_dependencies
                echo "✅ System requirements check complete"
                ;;
            0)
                print_status "INFO" "👋 Goodbye!"
                exit 0
                ;;
            *)
                print_status "ERROR" "❌ Invalid option"
                ;;
        esac
        
        read -p "$(print_status "INPUT" "⏎ Press Enter to continue...")"
    done
}

# Set trap to cleanup on exit
trap cleanup EXIT

# Load configuration
load_config

# Check dependencies
check_dependencies

# Supported OS list
declare -A OS_OPTIONS=(
    ["Ubuntu 22.04"]="ubuntu|jammy|https://cloud-images.ubuntu.com/jammy/current/jammy-server-cloudimg-amd64.img|ubuntu22|ubuntu|ubuntu"
    ["Ubuntu 24.04"]="ubuntu|noble|https://cloud-images.ubuntu.com/noble/current/noble-server-cloudimg-amd64.img|ubuntu24|ubuntu|ubuntu"
    ["Debian 11"]="debian|bullseye|https://cloud.debian.org/images/cloud/bullseye/latest/debian-11-generic-amd64.qcow2|debian11|debian|debian"
    ["Debian 12"]="debian|bookworm|https://cloud.debian.org/images/cloud/bookworm/latest/debian-12-generic-amd64.qcow2|debian12|debian|debian"
    ["Debian 13"]="debian|trixie|https://cloud.debian.org/images/cloud/trixie/daily/latest/debian-13-generic-amd64-daily.qcow2|debian13|debian|debian"
    ["Fedora 40"]="fedora|40|https://download.fedoraproject.org/pub/fedora/linux/releases/40/Cloud/x86_64/images/Fedora-Cloud-Base-40-1.14.x86_64.qcow2|fedora40|fedora|fedora"
    ["CentOS Stream 9"]="centos|stream9|https://cloud.centos.org/centos/9-stream/x86_64/images/CentOS-Stream-GenericCloud-9-latest.x86_64.qcow2|centos9|centos|centos"
    ["AlmaLinux 9"]="almalinux|9|https://repo.almalinux.org/almalinux/9/cloud/x86_64/images/AlmaLinux-9-GenericCloud-latest.x86_64.qcow2|almalinux9|alma|alma"
    ["Rocky Linux 9"]="rockylinux|9|https://download.rockylinux.org/pub/rocky/9/images/x86_64/Rocky-9-GenericCloud.latest.x86_64.qcow2|rocky9|rocky|rocky"
)

# Start the main menu
main_menu
