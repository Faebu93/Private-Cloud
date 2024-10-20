# Install Nextcloud and backup services

## Backup to coud & monitoring

### Configure rclone

```bash
sudo apt install -y rclone
```

On your development host, open an SSH connection forwarding port 53682 with the command `ssh -L localhost:53682:localhost:53682 username@remote_server`, in order to be able to see the authentication web page. See https://rclone.org/remote_setup/ for other options.

If you have already created a Client ID for OneDrive according to https://rclone.org/onedrive/#getting-your-own-client-id-and-key before, you might be able to extend the validity of the token. New registrations seem disabled, if the documentation has not been updated you might have to (and I think can) just go with the default rclone client ID. It might be throttled, but short tests have shown no difference.

Run `sudo rclone config` and complete the following steps:

- encrypt the configuration with a password of your choice, save the password: s -> a
- Create all remotes, where you want the backup to be stored (maybe distributed to several remotes)
- - If you have stored the backup in several locations: 
    
    Create a remote of type [`union`](https://rclone.org/union/) with the name `remote-nc-bkp`, with the remotes and paths where you want the backup stored. The other settings should be ok with the default values. 
  - If the backup is only in one location, name the remote directly `remote-nc-bkp`. The backup will be stored in the root of this remote, if the scripts are not adjusted accordingly.

Save the rclone configuration passwort to a root-only readable file, so that scripts can use rclone:

```bash
read -sp "Enter the password for your rclone configuration: " RCLONE_CONFIG_PASS
sudo install -m 640 -g prometheus /dev/null /etc/rclone.configpass
echo "$RCLONE_CONFIG_PASS" | sudo tee /etc/rclone.configpass > /dev/null
```

### Create backup job

Install the backup script and services which trigger it

```bash
sudo apt install -y inotify-tools 
sudo install -d /opt/private-cloud/scripts /var/lib/private-cloud/stats
sudo install ./resources/scripts/backup-nc-bkp.sh ./resources/scripts/mount-cloud-nc-bkp.sh ./resources/scripts/mount-disc-nc-bkp.sh /opt/private-cloud/scripts
sudo install ./resources/services/backup-cloud.service ./resources/services/backup-cloud.path /etc/systemd/system
sudo systemctl daemon-reload
sudo systemctl enable backup-cloud.path
sudo systemctl start backup-cloud.path
```

## Export metrics to Prometheus

https://github.com/xperimental/nextcloud-exporter

Create a random token, e.g. with `TOKEN=$(openssl rand -hex 32)`, and run `sudo docker exec --user www-data -it nextcloud-aio-nextcloud php occ config:app:set serverinfo token --value "$TOKEN"`.

Create the file with environment variables for configuration:

```bash
read -sp "Enter the public domain of your nextcloud installation" NC_FQDN
sudo tee -a /etc/prometheus-nextcloud-exporter.env <<EOF
NEXTCLOUD_SERVER=${NC_FQDN}
NEXTCLOUD_AUTH_TOKEN=$TOKEN
NEXTCLOUD_INFO_APPS=true
EOF
sudo chmod 600 /etc/prometheus-nextcloud-exporter.env
```

Install service for starting prometheus-nextcloud-exporter and its alerting rules

```bash
sudo install ./resources/services/prometheus-nextcloud-exporter.service /etc/systemd/system
sudo systemctl daemon-reload
sudo systemctl enable prometheus-nextcloud-exporter.service
sudo systemctl start prometheus-nextcloud-exporter.service
sudo install --mode=644 ./resources/prometheus/nextcloud-exporter.yml /etc/prometheus/alerts.d
```

Add a section to the scrape_configs section of Prometheus and reload:

```bash
sudo tee -a /etc/prometheus/prometheus.yml <<EOF
  - job_name: 'nextcloud'
    static_configs:
      - targets: ['$(hostname):9205']
EOF
sudo systemctl reload prometheus
```

Add the dashboard with ID 20716 to Grafana.