ENV_FILE ?= .env
envget = $(shell [ -f $(ENV_FILE) ] && sed -n 's/^$(1)=//p' $(ENV_FILE) | tr -d "'\"")

VM			?= $(call envget,VM_HOSTNAME)
USER_  		?= $(call envget,ADMIN_USER)
CPUS 		?= 2
MEM       	?= 4G
DISK      	?= 20G
IMAGE     	?= 24.04
TEMPLATE  	?= tofu/cloud-init.tpl.yaml
RENDERED  	?= tofu/cloud-init.rendered.yaml
SNAP      	?= clean
VM_IP      	= $$(multipass info $(VM) --format csv | tail -1 | cut -d, -f3)


.PHONY: help config render \
  vm-up vm-ip vm-ssh vm-stop vm-start vm-status vm-snapshot vm-snapshots vm-restore vm-down vm-inventory vm-ansible

config: ## Show the local VM settings make will use
	@echo "VM=$(VM) CPUS=$(CPUS) MEM=$(MEM) DISK=$(DISK) IMAGE=$(IMAGE) USER=$(USER_)"

help: ## Show available commands
	@grep -E '^[a-zA-Z_-]+:.*?## ' $(MAKEFILE_LIST) | awk 'BEGIN {FS = ":.*?## "}; {printf "  make %-14s %s\n", $$1, $$2}'

render: ## Fill credentials from .env into the cloud-init template
	@test -f $(ENV_FILE) || (echo "missing $(ENV_FILE) (see step 4)" && exit 1)
	@set -a && . ./$(ENV_FILE) && set +a && \
		envsubst '$${SSH_PUB_KEY} $${SUDO_PASSWD_HASH} $${VM_HOSTNAME} $${ADMIN_USER}' < $(TEMPLATE) > $(RENDERED)
	@chmod 600 $(RENDERED)


# ----------------- MULTIPASS -----------------
vm-up: render ## Create the local VM with cloud-init
	multipass launch $(IMAGE) --name $(VM) --cpus $(CPUS) --memory $(MEM) --disk $(DISK) --cloud-init $(RENDERED)
	@rm -f $(RENDERED)
	@echo "IP: $(VM_IP)"
vm-ip: ## Print the local VM IP
	@echo $(VM_IP)
vm-ssh: ## SSH into the local VM
	ssh $(USER_)@$(VM_IP)
vm-stop: ## Stop the local VM (frees RAM/CPU, keeps the disk)
	multipass stop $(VM)
vm-start: ## Start the local VM
	multipass start $(VM)
vm-status: ## List local VMs
	multipass list
vm-snapshot: ## Save a checkpoint: make vm-snapshot SNAP=after-phase2
	multipass stop $(VM)
	multipass snapshot $(VM) --name $(SNAP)
vm-snapshots: ## List checkpoints
	multipass list --snapshots
vm-restore: ## Roll back: make vm-restore SNAP=after-phase2
	multipass stop $(VM)
	multipass restore $(VM).$(SNAP) --destructive
	multipass start $(VM)
vm-down: ## Delete the local VM and free its disk
	multipass delete --purge $(VM)
vm-inventory: ## Write the Ansible inventory for the local VM
	@printf "[k8s]\n$(VM) ansible_host=%s ansible_user=$(USER_)\n" "$(VM_IP)" > ansible/inventory.local.ini
vm-ansible: vm-inventory ## Run the playbook against the local VM
	ansible-playbook -i ansible/inventory.local.ini ansible/site.yml --ask-become-pass