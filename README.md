# Bobcat300-DebianMinimalDocker
A minimal docker-compose config for running a helium lora gateway on a bobcat 300.

## Details
Currently only tested on the `G285/29X`

Uses the following docker images:
- `heliumdiy/sx1302_hal` - Fast and common pf used by most diy miners.
- `quay.io/team-helium/miner:gateway-latest` - Official gateway image

### Credits:
- r00t1ng - suppling the initial missing ldo reset script
- [sicXnull](https://github.com/sicXnull) has done some amazing work on reverse engineering the bobcat 300. Check out their repos:
  - https://github.com/sicXnull/Bobcat300-Debian
  - https://github.com/sicXnull/BobcatDashboard

## Ansible config:
1. Install the debian image provided by https://github.com/sicXnull/Bobcat300-Debian
2. Run a `ssh-copy-id root@192.168.x.x` to the host, update `hosts.ini` file, change the bobcat root password with `passwd`
3. Set `region` and `device_model` in `bobcat.yml` file
4. Install:  
```bash
ansible-playbook -i hosts.ini bobcat.yml
```

## Manual config: 

### Preparations
1. Install the debian image provided by https://github.com/sicXnull/Bobcat300-Debian

2. Remove the dashboard and additional packages like strongswan, nginx-common, php etc (to your liking)  
```bash
sh /var/dashboard/uninstall.sh
systemctl disable timezone-config-check.timer server-detection.timer bobcat-watchdog.timer
apt remove -y php* nginx* strongswan* openvpn*
docker rm -f helium-miner portainer pktfwd
apt -y autoremove
```

3. Reduce SD wear
```bash
systemctl disable rsyslog
echo "Storage=volatile" >> /etc/systemd/journald.conf
```

4. Disable wifi chip (wifi chip is not functional on this firmware):
```bash
nmcli radio wifi off
systemctl disable iwd wpa_supplicant
```

5. Ensure latest update: 
```bash
apt update && apt upgrade -y && reboot
```

6. Reboot and install docker compose and chrony for time sync 
```bash
apt -y install docker-compose chrony
```

7. Check `chrony` ntp sync status: 
```bash
chronyc tracking
```

8. Set root user password 
```bash
passwd
```

9. Consider adding ssh-keys and disabling password based auth by adding `PasswordAuthentication no` to `/etc/ssh/ssh_config` 

### Install

Clone the project:
```bash
git clone https://github.com/metrafonic/Bobcat300-DebianMinimalDocker bobcat && cd bobcat
```

**Select the appropriate docker-compose file for your device:**
- **G285/G280**: Use `docker-compose-G285.yml`
- **G29X**: Use `docker-compose-G29X.yml`

Modify the `REGION` value in your selected docker-compose file. The `REGION` can be one of the following values:  
`US915 | EU868 | EU433 | CN470 | CN779 | AU915 | AS923 | KR920 | IN865`

Start the pf once, so that it creates the region file. We must modify it.

**For G285/G280:**
```bash
docker-compose -f docker-compose-G285.yml up -d pktfwd && sleep 1 && docker-compose -f docker-compose-G285.yml down
```

**For G29X:**
```bash
docker-compose -f docker-compose-G29X.yml up -d pktfwd && sleep 1 && docker-compose -f docker-compose-G29X.yml down
```

Edit `packet_forwarder/configs/global_conf.json` and replace the `spidev_path:` value:

**For G285:**
```json
"spidev_path": "/dev/spidev1.0",
```

**For G29X:**
```json
"spidev_path": "/dev/spidev5.0",
```

Remove the line:
```json
"gps_i2c_path": "/dev/i2c-1",
```

Start again with your selected docker-compose file:

**For G285:**
```bash
docker-compose -f docker-compose-G285.yml up -d 
```

**For G29X:**
```bash
docker-compose -f docker-compose-G29X.yml up -d 
```

Check the logs (replace with your compose file):
```bash
docker-compose -f docker-compose-G285.yml logs -f --tail=1000
```

Check miner animal name (replace with your compose file):
```bash
docker-compose -f docker-compose-G285.yml exec miner helium_gateway key info
```

## Troubleshooting

If you encounter issues:
- Verify the correct docker-compose file is being used for your device model
- Check that the `spidev_path` matches your device (spidev1.0 for G285/G280, spidev5.0 for G29X)
- Ensure the region is set correctly in your docker-compose file
- Check container logs for errors: `docker-compose -f <your-compose-file> logs`
