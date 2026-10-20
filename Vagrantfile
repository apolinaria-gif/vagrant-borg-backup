Vagrant.configure("2") do |config|
  config.vm.box = "bento/ubuntu-22.04"

  config.vm.define "backup_server" do |server|
    server.vm.hostname = "backup-server"
    server.vm.network "private_network", ip: "192.168.56.10"
    server.vm.provider "vmware_desktop" do |v|
      v.memory = 1024
      v.cpus = 1
    end
    
    server.vm.provision "shell", inline: <<-SHELL
      apt-get update
      apt-get install -y borgbackup
      mkdir -p /var/backup
      id -u borg &>/dev/null || useradd -m -s /bin/bash borg
      chown -R borg:borg /var/backup
      chmod 0770 /var/backup
      mkdir -p /home/borg/.ssh
      chmod 0700 /home/borg/.ssh
      chown -R borg:borg /home/borg/.ssh
    SHELL
  end

  config.vm.define "client" do |client|
    client.vm.hostname = "client"
    client.vm.network "private_network", ip: "192.168.56.20"
    client.vm.provider "vmware_desktop" do |v|
      v.memory = 1024
      v.cpus = 1
    end
    
    client.vm.provision "shell", inline: <<-SHELL
      apt-get update
      apt-get install -y borgbackup sshpass

      
      [ -f /root/.ssh/id_rsa ] || ssh-keygen -t rsa -b 2048 -N "" -f /root/.ssh/id_rsa

      
      export SSHPASS='vagrant'
      sshpass -e ssh-copy-id -o StrictHostKeyChecking=no -i /root/.ssh/id_rsa.pub borg@192.168.56.10

      
      cat << 'EOF' > /usr/local/bin/borg-backup.sh
#!/bin/bash
export BORG_REPO="borg@192.168.56.10:/var/backup/etc-repo"
export BORG_PASSPHRASE="your_secure_password_here"
export BORG_RSH="ssh -o StrictHostKeyChecking=no"
TAG="borg-backup"

if ! borg list $BORG_REPO >/dev/null 2>&1; then
    borg init --encryption=repokey-blake2 $BORG_REPO
    logger -t "$TAG" "Borg repository initialized."
fi

BACKUP_NAME="etc-backup-$(date +%Y-%m-%d_%H-%M-%S)"
logger -t "$TAG" "Starting backup: $BACKUP_NAME"

borg create --stats $BORG_REPO::"$BACKUP_NAME" /etc
if [ $? -eq 0 ]; then
    logger -t "$TAG" "Backup $BACKUP_NAME successfully created."
else
    logger -t "$TAG" "ERROR: Backup $BACKUP_NAME failed!"
    exit 1
fi

logger -t "$TAG" "Starting prune process..."
borg prune --list --keep-daily=90 --keep-monthly=12 $BORG_REPO
if [ $? -eq 0 ]; then
    logger -t "$TAG" "Prune completed successfully."
else
    logger -t "$TAG" "ERROR: Prune failed!"
    exit 1
fi
EOF

      chmod +x /usr/local/bin/borg-backup.sh

      
      cat << 'EOF' > /etc/systemd/system/borg-backup.service
[Unit]
Description=Borg Backup Service for /etc
After=network.target

[Service]
Type=oneshot
ExecStart=/usr/local/bin/borg-backup.sh
EOF

      
      cat << 'EOF' > /etc/systemd/system/borg-backup.timer
[Unit]
Description=Run Borg Backup every 5 minutes

[Timer]
OnCalendar=*:0/5
Persistent=true

[Install]
WantedBy=timers.target
EOF

      systemctl daemon-reload
      systemctl enable --now borg-backup.timer
    SHELL
  end
end
