ENV_FILE ?= .env
envget = $(shell [ -f $(ENV_FILE) ] && sed -n 's/^$(1)=//p' $(ENV_FILE) | tr -d "'\"")

VM							?= $(call envget,VM_HOSTNAME)
USER_  						?= $(call envget,ADMIN_USER)
IMAGE     					?= 24.04
TEMPLATE  					?= tofu/cloud-init.tpl.yaml
RENDERED  					?= tofu/cloud-init.rendered.yaml
SSH_PRIVATE_KEY_PATH		?= $(call envget,SSH_PRIVATE_KEY_PATH)
VM_IP      					= $$(multipass list --format csv | awk -F, '$$1=="$(VM)"{print $$3}')

VPS_IP               ?= $(call envget,VPS_IP)
VPS_HOSTNAME         ?= $(call envget,VPS_HOSTNAME)
VPS_DEFAULT_USER     ?= ubuntu
PG_PORT               ?= 5432
SSH_OPTS              = -o IdentitiesOnly=yes

.PHONY: help config render \
  vm-up vm-ip vm-ssh vm-stop vm-start vm-status vm-snapshot vm-snapshots vm-restore vm-down vm-inventory vm-ping vm-k3s vm-logs \
  vps-bootstrap vps-inventory vps-ssh vps-k3s vps-ping vps-backup vps-backup-now vps-db vps-tailscale vps-db-secret

help: ## Show available commands
	@grep -E '^[a-zA-Z_-]+:.*?## ' $(MAKEFILE_LIST) | awk 'BEGIN {FS = ":.*?## "}; {printf "  make %-14s %s\n", $$1, $$2}'

render: ## Fill credentials from .env into the cloud-init template
	@test -f $(ENV_FILE) || (echo "missing $(ENV_FILE) (see step 4)" && exit 1)
	@set -a && . ./$(ENV_FILE) && set +a && \
		envsubst '$${SSH_PUB_KEY} $${SUDO_PASSWD_HASH} $${VM_HOSTNAME} $${ADMIN_USER}' < $(TEMPLATE) > $(RENDERED)
	@chmod 600 $(RENDERED)


# ----------------- MULTIPASS -----------------
vm-up: render ## Create the local VM with cloud-init
	multipass launch $(IMAGE) --name $(VM) --cpus $(CPUS) --memory $(MEM) --disk $(DISK) --timeout 1200 --cloud-init $(RENDERED)
	@rm -f $(RENDERED)
	@echo "IP: $(VM_IP)"
vm-ip: ## Print the local VM IP
	@echo $(VM_IP)
vm-ssh: ## SSH into the local VM
	ssh -i $(SSH_PRIVATE_KEY_PATH) -o IdentitiesOnly=yes $(USER_)@$(VM_IP)
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
	@printf "[k3s]\n$(VM) ansible_host=%s ansible_user=$(USER_) ansible_ssh_private_key_file=$(SSH_PRIVATE_KEY_PATH) ansible_ssh_common_args='-o IdentitiesOnly=yes'\n" "$(VM_IP)" > ansible/inventory.local.ini
	@cat ansible/inventory.local.ini
vm-ping: vm-inventory ## Check Ansible can reach the local VM
	cd ansible && ansible k3s -m ping
vm-k3s: vm-inventory ## Install k3s on the local VM with Ansible
	cd ansible && ansible-playbook k3s.yaml --ask-become-pass
vm-logs:
	@multipass exec $(VM) -- sudo tail -f /var/log/cloud-init-output.log


# ----------------- REAL REMOTE -----------------
vps-bootstrap: ## First setup of a new VPS (logs in as the provider's default user)
	@test -n "$(VPS_IP)" || (echo "VPS_IP is missing in $(ENV_FILE)" && exit 1)
	@printf "[bootstrap]\n$(VPS_HOSTNAME) ansible_host=$(VPS_IP) ansible_user=$(VPS_DEFAULT_USER) ansible_ssh_private_key_file=$(SSH_PRIVATE_KEY_PATH) ansible_ssh_common_args='$(SSH_OPTS)'\n" > ansible/inventory.bootstrap.ini
	@set -a && . ./$(ENV_FILE) && set +a && \
		cd ansible && ansible-playbook -i inventory.bootstrap.ini bootstrap.yaml --ask-become-pass
vps-inventory: ## Write the Ansible inventory for the VPS
	@printf "[k3s]\n$(VPS_HOSTNAME) ansible_host=$(VPS_IP) ansible_user=$(USER_) ansible_ssh_private_key_file=$(SSH_PRIVATE_KEY_PATH) ansible_ssh_common_args='$(SSH_OPTS)'\n" > ansible/inventory.vps.ini
	@cat ansible/inventory.vps.ini
vps-ping: vps-inventory ## Check Ansible can reach the VPS as your admin user
	cd ansible && ansible -i inventory.vps.ini k3s -m ping
vps-ssh: ## SSH into the VPS as your admin user
	ssh -i $(SSH_PRIVATE_KEY_PATH) $(SSH_OPTS) $(USER_)@$(VPS_IP)
vps-k3s: vps-inventory ## Install k3s and the platform on the VPS
	@set -a && . ./$(ENV_FILE) && set +a && \
		cd ansible && ansible-playbook -i inventory.vps.ini k3s.yaml --ask-become-pass
vps-backup: vps-inventory ## Install the GCS backup (every 3 days) of all app data on the VPS
	@set -a && . ./$(ENV_FILE) && set +a && \
		cd ansible && ansible-playbook -i inventory.vps.ini backup.yaml --ask-become-pass
vps-backup-now: ## Run the GCS backup on the VPS right now
	ssh -t -i $(SSH_PRIVATE_KEY_PATH) $(SSH_OPTS) $(USER_)@$(VPS_IP) sudo /usr/local/sbin/nyehrver-backup

vps-db: ## Tunnel the in-cluster Postgres to localhost:$(PG_PORT) (Ctrl-C to close)
	ssh -t -L $(PG_PORT):localhost:15432 -i $(SSH_PRIVATE_KEY_PATH) $(SSH_OPTS) $(USER_)@$(VPS_IP) \
		sudo k3s kubectl port-forward -n postgres svc/postgres 15432:5432
vps-tailscale: vps-inventory ## Join the VPS to Tailscale and serve Postgres on its tailnet IP:5432
	@set -a && . ./$(ENV_FILE) && set +a && \
		cd ansible && ansible-playbook -i inventory.vps.ini tailscale.yaml --ask-become-pass
vps-db-secret: ## Create the Postgres secret in the cluster from POSTGRES_* in .env (no kubectl/kubeseal needed locally)
	@test -n "$(call envget,POSTGRES_PASSWORD)" || (echo "POSTGRES_PASSWORD is missing in $(ENV_FILE)" && exit 1)
	@grep -E '^POSTGRES_(USER|PASSWORD|DB)=' $(ENV_FILE) | tr -d "'\"" | \
		ssh -i $(SSH_PRIVATE_KEY_PATH) $(SSH_OPTS) $(USER_)@$(VPS_IP) 'umask 077 && cat > ~/.pg.env'
	ssh -t -i $(SSH_PRIVATE_KEY_PATH) $(SSH_OPTS) $(USER_)@$(VPS_IP) \
		'sudo k3s kubectl create secret generic postgres-env -n postgres --from-env-file=$$HOME/.pg.env --dry-run=client -o yaml | sudo k3s kubectl apply -f -; rm -f ~/.pg.env'
