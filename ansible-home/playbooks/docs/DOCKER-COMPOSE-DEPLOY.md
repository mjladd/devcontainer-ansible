# Docker Compose Deployment with Ansible

This directory contains Ansible playbooks for deploying and managing Docker Compose applications.

## Playbooks

### 1. deploy-docker-compose.yml
Uses the Ansible `community.docker` collection (recommended for production).

**Requirements:**
- Install the collection: `ansible-galaxy collection install community.docker`

**Usage:**
```bash
ansible-playbook playbooks/deploy-docker-compose.yml \
  -e compose_file_src=path/to/your/docker-compose.yml \
  -e compose_dest_dir=/opt/myapp \
  -e compose_project_name=myapp
```

### 2. deploy-docker-compose-cli.yml
Uses Docker Compose CLI commands directly (no collection required).

**Usage:**
```bash
ansible-playbook playbooks/deploy-docker-compose-cli.yml \
  -e compose_file_src=path/to/your/docker-compose.yml \
  -e compose_dest_dir=/opt/myapp \
  -e compose_project_name=myapp
```

## Variables

| Variable | Description | Default |
|----------|-------------|---------|
| `compose_file_src` | Path to docker-compose.yml on the control node | `docker-compose.yml` |
| `compose_dest_dir` | Destination directory on target host | `/opt/docker-compose-app` |
| `compose_project_name` | Docker Compose project name | `myapp` |

## Quick Start

1. Copy the example docker-compose file:
   ```bash
   cp playbooks/docker-compose.yml.example playbooks/docker-compose.yml
   ```

2. Edit the docker-compose.yml file with your services

3. Run the playbook:
   ```bash
   ansible-playbook playbooks/deploy-docker-compose-cli.yml
   ```

## Examples

### Deploy a custom application
```bash
ansible-playbook playbooks/deploy-docker-compose-cli.yml \
  -e compose_file_src=/home/user/myapp/docker-compose.yml \
  -e compose_dest_dir=/opt/myapp \
  -e compose_project_name=myapp \
  -i inventory/inventory.ini
```

### Deploy to specific hosts
```bash
ansible-playbook playbooks/deploy-docker-compose-cli.yml \
  --limit server1 \
  -e compose_file_src=./my-stack.yml
```

### Check what would change (dry run)
```bash
ansible-playbook playbooks/deploy-docker-compose-cli.yml --check
```

## What the Playbooks Do

1. **Install Docker** - Ensures Docker is installed and running
2. **Install Docker Compose** - Installs Docker Compose if not present
3. **Create directories** - Sets up the deployment directory structure
4. **Copy files** - Transfers docker-compose.yml to the target host
5. **Pull images** - Downloads the required Docker images
6. **Stop old containers** - Stops existing containers if the compose file changed
7. **Start services** - Brings up the Docker Compose stack
8. **Verify status** - Shows the status of running containers

## Troubleshooting

### Check if containers are running
```bash
ansible servers -m shell -a "docker ps" -b
```

### View container logs
```bash
ansible servers -m shell -a "cd /opt/docker-compose-app && docker-compose logs --tail=50" -b
```

### Restart services
```bash
ansible servers -m shell -a "cd /opt/docker-compose-app && docker-compose restart" -b
```

## Notes

- The playbooks will automatically restart containers if the docker-compose.yml file changes
- Environment variables can be passed via `.env` file
- For production, consider using secrets management (Ansible Vault, external secrets)
- Adjust the `hosts` target in the playbook as needed (currently set to `servers`)
