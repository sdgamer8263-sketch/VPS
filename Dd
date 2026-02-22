#!/bin/bash
export DEBIAN_FRONTEND=noninteractive

# -------------------------------
# Utils
# -------------------------------
cmd_exists() {
    command -v "$1" >/dev/null 2>&1
}

# -------------------------------
# Detect OS
# -------------------------------
detect_os() {
    if [ -f /etc/os-release ]; then
        . /etc/os-release
        OS=$ID
        VERSION_ID=${VERSION_ID%%.*}
    else
        echo "❌ Cannot detect OS"
        exit 1
    fi
    echo "✔ Detected OS: $OS $VERSION_ID"
}

# -------------------------------
# Install packages (SMART)
# -------------------------------
install_packages() {
    INSTALL=()

    case "$OS" in
        ubuntu|debian)
            cmd_exists qemu-system-x86_64 || INSTALL+=(qemu-kvm qemu-utils)
            cmd_exists wget || INSTALL+=(wget)
            cmd_exists lsof || INSTALL+=(lsof)
            cmd_exists growpart || INSTALL+=(cloud-guest-utils)
            cmd_exists cloud-localds || INSTALL+=(cloud-image-utils)
            cmd_exists genisoimage || INSTALL+=(genisoimage)

            [ ${#INSTALL[@]} -eq 0 ] && echo "✔ All packages already installed" || \
            (sudo apt update -y && sudo apt install -y "${INSTALL[@]}")
            ;;

        fedora)
            cmd_exists qemu-system-x86_64 || INSTALL+=(qemu-kvm qemu-img)
            cmd_exists wget || INSTALL+=(wget)
            cmd_exists lsof || INSTALL+=(lsof)
            cmd_exists growpart || INSTALL+=(cloud-utils-growpart)
            cmd_exists genisoimage || INSTALL+=(genisoimage)

            if [ ${#INSTALL[@]} -gt 0 ]; then
                sudo dnf install -y epel-release
                sudo dnf install -y "${INSTALL[@]}" cloud-utils
            else
                echo "✔ All packages already installed"
            fi
            ;;

        centos|rocky|almalinux|rhel)
            cmd_exists qemu-system-x86_64 || INSTALL+=(qemu-kvm qemu-img)
            cmd_exists wget || INSTALL+=(wget)
            cmd_exists lsof || INSTALL+=(lsof)
            cmd_exists growpart || INSTALL+=(cloud-utils-growpart)
            cmd_exists genisoimage || INSTALL+=(genisoimage)

            if [ ${#INSTALL[@]} -gt 0 ]; then
                if ! rpm -q epel-release >/dev/null 2>&1; then
                    sudo dnf install -y https://dl.fedoraproject.org/pub/epel/epel-release-latest-${VERSION_ID}.noarch.rpm
                fi
                sudo dnf install -y "${INSTALL[@]}" cloud-utils
            else
                echo "✔ All packages already installed"
            fi
            ;;

        amzn)
            cmd_exists qemu-system-x86_64 || INSTALL+=(qemu-kvm qemu-img)
            cmd_exists wget || INSTALL+=(wget)
            cmd_exists lsof || INSTALL+=(lsof)
            cmd_exists growpart || INSTALL+=(cloud-utils-growpart)

            [ ${#INSTALL[@]} -eq 0 ] && echo "✔ All packages already installed" || \
            sudo dnf install -y "${INSTALL[@]}" cloud-utils
            ;;

        arch)
            cmd_exists qemu-system-x86_64 || INSTALL+=(qemu-full)
            cmd_exists wget || INSTALL+=(wget)
            cmd_exists lsof || INSTALL+=(lsof)
            cmd_exists growpart || INSTALL+=(cloud-guest-utils)
            cmd_exists genisoimage || INSTALL+=(cdrtools)

            [ ${#INSTALL[@]} -eq 0 ] && echo "✔ All packages already installed" || \
            (sudo pacman -Sy --noconfirm "${INSTALL[@]}")
            ;;

        opensuse*|sles)
            cmd_exists qemu-system-x86_64 || INSTALL+=(qemu-tools qemu-kvm)
            cmd_exists wget || INSTALL+=(wget)
            cmd_exists lsof || INSTALL+=(lsof)
            cmd_exists growpart || INSTALL+=(cloud-utils-growpart)
            cmd_exists genisoimage || INSTALL+=(genisoimage)

            [ ${#INSTALL[@]} -eq 0 ] && echo "✔ All packages already installed" || \
            sudo zypper --non-interactive install "${INSTALL[@]}"
            ;;

        *)
            echo "❌ Unsupported OS: $OS"
            exit 1
            ;;
    esac
}

# -------------------------------
# Links
# -------------------------------
setup_links() {
    if [ -f /usr/libexec/qemu-kvm ] && ! cmd_exists qemu-system-x86_64; then
        sudo ln -sf /usr/libexec/qemu-kvm /usr/bin/qemu-system-x86_64
    fi

    if cmd_exists cloud-localds && [ -f /usr/bin/cloud-localds ]; then
        sudo ln -sf /usr/bin/cloud-localds /usr/local/bin/cloud-localds
    fi
}

# -------------------------------
# Main
# -------------------------------
main() {
    echo "========================================="
    echo " MULTI-OS VM TOOLS INSTALLER"
    echo "========================================="

    detect_os
    install_packages
    setup_links

    echo "✔ Completed for $OS $VERSION_ID"
}

main "$@"
