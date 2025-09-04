# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

This is the Kubernetes SIGs image-builder project, which creates VM images for Kubernetes clusters across multiple infrastructure providers. The resulting VM images are specifically intended for use with Cluster API (CAPI) but are suitable for other Kubeadm-based setups.

## Build Commands

### Prerequisites
```bash
# Install all dependencies for a specific provider
make deps-<provider>  # e.g., deps-azure, deps-gce, deps-aws

# Install common dependencies (required for most builds)
make deps-common
```

### Building Images

Images are built per provider and OS combination. Use the pattern:
```bash
make build-<provider>-<os>
```

Examples:
```bash
# Azure images
make build-azure-sig-ubuntu-2404
make build-azure-sig-ubuntu-2404-gen2-arm64
make build-azure-sig-windows-2022-containerd

# AWS AMIs
make build-ami-ubuntu-2404
make build-ami-rhel-8

# GCE images
make build-gce-ubuntu-2404
```

### Validation and Testing

```bash
# Validate Packer configuration before building
make validate-<provider>-<os>  # e.g., validate-azure-sig-ubuntu-2404

# Validate all configurations for a provider
make validate-<provider>-all    # e.g., validate-azure-all

# Validate all configurations
make validate-all

# Run linters on the codebase
make lint

# Run linters and automatically fix issues
make lint-fix

# Run Azure-specific tests
make test-azure
```

### Update ISO Checksums
```bash
make update-all-iso-checksums
make update-ubuntu-iso-checksums
make update-rockylinux-iso-checksums
```

## Architecture

### Directory Structure

- `images/capi/` - Main directory for CAPI image building
  - `packer/` - Packer templates organized by provider
    - `azure/`, `ami/`, `gce/`, etc. - Provider-specific Packer configurations
    - `config/` - Shared Packer configuration files
    - `goss/` - Goss test specifications for image validation
  - `ansible/` - Ansible playbooks and roles for image configuration
    - `roles/` - Ansible roles for various components (containerd, node, kubernetes, etc.)
  - `scripts/` - CI and utility scripts
  - `Makefile` - Primary build orchestration

### Key Components

1. **Packer Templates**: Each provider has JSON/HCL templates defining how to build images. Common patterns:
   - Base image selection
   - Provisioner configuration (Ansible)
   - Post-processors for image publication

2. **Ansible Roles**: Modular configuration management:
   - `containerd` - Container runtime installation
   - `node` - Base node configuration
   - `kubernetes` - Kubernetes components installation
   - Provider-specific roles for cloud-init, drivers, etc.

3. **Build Variables**: Configured via environment variables or `packer.json` files:
   - `KUBERNETES_VERSION` - Target Kubernetes version
   - `CONTAINERD_VERSION` - Container runtime version
   - Provider-specific variables (subscription ID, project, region, etc.)

## Azure-Specific Configuration

### Environment Variables for Azure Builds
```bash
export AZURE_SUBSCRIPTION_ID="your-subscription-id"
export AZURE_RESOURCE_GROUP_NAME="your-resource-group"
export AZURE_LOCATION="eastus2"
export GALLERY_NAME="your-gallery-name"
export USE_AZURE_CLI_AUTH=True  # Use Azure CLI authentication
```

### Azure Image Types
- **VHD**: Virtual Hard Disk images
- **SIG (Shared Image Gallery)**: Managed images with versioning
- **Gen2**: Generation 2 VMs with UEFI boot
- **ARM64**: ARM64 architecture support
- **CVM**: Confidential VMs with enhanced security

## Common Development Tasks

### Adding Support for a New OS Version

1. Create Packer template in `packer/<provider>/<os>-<version>.json`
2. Add corresponding Makefile targets for build and validate
3. Update Ansible roles if OS-specific configuration is needed
4. Add to CI matrix if applicable

### Modifying Kubernetes Configuration

1. Edit relevant Ansible roles in `ansible/roles/`
2. Common modification points:
   - `ansible/roles/node/defaults/main.yml` - Node configuration defaults
   - `ansible/roles/containerd/tasks/main.yml` - Container runtime setup
   - `ansible/roles/kubernetes/` - Kubernetes component configuration

### Testing Changes Locally

1. Use validate targets to check Packer syntax
2. Build with specific debug flags:
   ```bash
   PACKER_FLAGS="-on-error=ask" make build-<target>
   ```
3. Use Goss tests for validation during build

## Important Notes

- Always validate Packer configurations before building
- The project uses Ansible for configuration management - ensure playbooks are idempotent
- Images are intended for Cluster API but should work with any Kubeadm-based setup
- Each provider may have specific authentication requirements - check provider documentation
- Build times vary significantly by provider and OS (15 minutes to 2+ hours)