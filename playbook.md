saarCTF server playbook
=======================

This guide will create ready-to-go servers hosted on [Hetzner Cloud](https://www.hetzner.de/cloud). Account there (and services) are required.
If you want a local test environment check out the [Virtualbox instructions](virtualbox.md).


### Before the CTF
If you are following the [general playbook](https://gitlab.saarsec.rocks/saarctf/wiki/-/wikis/Playbook), you should already have done these steps.

1. Create a new folder in the configuration repository, copying from last year's iteration. Adjust settings if necessary.
2. Setup the saarctf registration website somewhere ("registration server", saarsec server for example). 
3. Checkout config repo on registration server, point the registration website there. 


### Preparations (a few days before start)
We will now prepare server images and setup VPN servers (to test against).

1. Go to `basis/basic-setup.sh` in this repository and edit the first line. Set `CONFIG_SUBDIR` to the name of your new config repo subfolder (created in the very first step). 
2. Use [packer](https://www.packer.io/) to build all images except debian (Hetzner version only, Virtualbox is not required). Run these commands (basis build before other builds): 
   ```
   export HCLOUD_TOKEN=<...>
   cd basis      ; packer build -only=hcloud -on-error=ask -var-file=../config.json basis.json      ; cd ..
   cd controller ; packer build -only=hcloud -on-error=ask -var-file=../config.json controller.json ; cd ..
   cd vpn        ; packer build -only=hcloud -on-error=ask -var-file=../config.json vpn.json        ; cd ..
   cd checker    ; packer build -only=hcloud -on-error=ask -var-file=../config.json checker.json    ; cd ..
   ```
   After this step your Hetzner account should have at least snapshots for controller, vpn and checker. 
3. Create a new private network for the game (or reuse the old one). Configure:
   - IP Range: `10.32.0.0/11`
   - One subnet: `10.32.250.0/24`
4. Create a VPN server (first) and a Controller server (second). Use the smallest possible server type, and add both to the network (can be selected when creating a server). 
5. Check the IPs of the machines from the previous step: Controller must be `10.32.250.2`, VPN should be `10.32.250.1`. If IPs are wrong remove and reattach the server from the network.
6. Add routes to the network:
   - `10.32.0.0/17` => VPN server
   - `10.32.128.0/18` => VPN server
   - `10.32.192.0/19` => VPN server
   - `10.33.0.0/16` => VPN server
   - `10.48.0.0/15` => VPN server
7. Configure the domains:
   - `vpn.ctf.saarland` should point to the public IP of VPN
   - `scoreboard.ctf.saarland` should point to the public IP of Controller
   - `cp.ctf.saarland` can point to Controller or to a failover Controller server (to be added later)
8. Let's configure SSL:
   1. if necessary: on saarsec server copy the certs to the configuration repo (`cp -L /etc/letsencrypt/live/ctf.saarland/* /opt/saarctf-config/certs/`) and push.
   2. Pull repo on Controller and Vpn server server
   3. Run `~/nginx-enable-ssl.sh` on Controller and VPN server
9. Check that [https://scoreboard.ctf.saarland](https://scoreboard.ctf.saarland) and [https://vpn.ctf.saarland](https://vpn.ctf.saarland) both have meaningful output. If scoreboard bugs issue `update-server` on Controller. If Flower / RabbitMQ bug issue `~/rabbitmq-setup.sh`. 
10. Enable the link to scoreboard on the saarCTF website (`TODO`).
11. Configure synchronization (team logos to scoreboard, VPN config): enable root's cronjob on VPN server and enable saarctf's cronjob on controller.


#### Configure Services
Once ready, you can **setup the services** on the controller:

1. On controller: `sudo -u saarctf -i` and `cd /home/saarctf/checkers`
2. Git clone all repos with checker scripts to a folder in `/home/saarctf/checkers`, use user account "saarctf" for that. 
3. Add the service to the `services` table using `/root/postgres-psql.sh`. See `services.sql` in the configuration repo.
4. Check that the service is recognized in the control panel. 
5. Add all service to `/etc/ports.conf` on the VPN server,so that their traffic will be monitored and accounted.


#### Synchronize teams
After IPs etc. have been fixed, you can **import teams** and **create VPN configs**. You can repeat this step any time to update configs.
Step 2-6 are scripted by `~/configs-create.sh` on VPN server.

1. On the registration server, commit and push the database and all team logos. On saarsec there's a cronjob for that.
2. Pull configuration repo on controller server or vpn server (both will work for now)
3. `cd /opt/gameserver`
4. `python3 scripts/sync_teams.py` (import new teams and changed team logos)
5. `vpn/build-openvpn-config-oneperteam.py` (create configs server- and clientside)
6. Commit and push generated VPN config files (config repo)
7. Back on registration server, pull config repo (to make VPN configs available)


### Just before the CTF
A few hours before the game starts, its time to build the real infrastructure. 
The existing servers will be upscaled, redundancy and checkers will be added. 
These steps involve a downtime of a few minutes, existing VPN connections will break.
What we will build: 

- 1x CCX41 Controller
- 1x CX51  Backup (of controller)
- 1x CCX41 VPN Gateway
- ?x CX51  Checker


1. Close the registration and complete the synchronization steps above a final time. 
2. On the controller server open the control panel and create a first scoreboard.
3. In the cloud control panel, create a new volume (for pcaps). I suggest at least 1TB. 
4. Create the backup/monitoring server: Spawn a new server from the controller image, type CX51 or so, choose to have a larger disk. 
5. Once the backup server is up, run `/root/failover-set-slave.sh`
6. Shutdown VPN server, then Controller server. 
7. Add the Volume from (3) to the VPN server. 
8. Rescale Controller and VPN server (for example to CCX41). Start with Controller (which needs the additional disk space). 
9. Once both are up again reset connectivity status (`vpn/reset-connection-status.py` on VPN server), mention restart in IRC, and check that the replication databases from the backup system are reconnecting. Also check that Controller and Backup have their additional disk space (if not try `resize2fs /dev/sda1`). 
10. Mount the volume on the VPN server (follow instructions on Hetzner page).
11. Create folders and symlinks: `mkdir -p /mnt/pcaps/gametraffic /mnt/pcaps/teamtraffic ; ln -s /mnt/pcaps/gametraffic /tmp/ ; ln -s /mnt/pcaps/teamtraffic /tmp/`
12. On the VPN server start tcpdump (`systemctl start tcpdump-game tcpdump-team`) and check that the pcaps are stored on the mounted volume. 
13. Start the CTF timer on the Controller server (`systemctl start ctftimer`) and check the results in dashboard. There should be no warning. 
14. On the backup server, enable the Scoreboard daemon (`systemctl start scoreboard`). 
15. Create some checker servers from the checker snapshot, CX51, disk doesn't matter. Connect to each new checker server and run `celery-configure <server-number>`.
16. Install all missing dependencies on the checker servers. Check that the celery workers are connecting. 
17. Go to the backup/monitoring server. Use `/root/prometheus-add-server.sh <ip>` to add: controller server, vpn server, all checker servers.
18. Open Grafana and check that you receive data from all servers, and all are healty.
19. You could do some final tests with the NOP team here (`TODO`).
20. Finally: Open the CTF controlpanel and set autostart and last round. Pay attention to the server's timezone.
21. Watch the countdown for the game to start...
22. Once the key is released (or shortly before), you can open `/etc/systemd/services/vpnboad.service` and add the command line argument `--check-vulnbox`. Run `systemctl restart vpnboard`


### During CTF
Network opens and closes automatically. Checkers and scoring work automatically. Your job is it to monitor the game and step in if things get out of control. Some remarks: 

- Checker servers can be added any time, but need the setup command described above. 
- Before removing a checker server, gracefully stop the celery daemon there.
- Get familiar with the scripts in `/opt/gameserver/scripts`. They can manually trigger actions in the game (for example: repeat stuff that failed) or recalculate things (for example after SLA recalculation). 
- Make a backup (database dump) before every manual change.
- If the load on the main Controller gets too high, you can move submitter and scoreboard to a dedicated machine: Boot a new controller, disable databases and monitoring there, enable the scoreboard daemon and redirect traffic. 



### After the CTF
Hetzner servers cost money per hour, even if they're powered off. After the CTF it's crucial to get rid of them as fast as possible. 

1. Shutdown and remove the checker servers. You shouldn't need any backup from there, except if celery-related issued occured. Remember: remove, not just power off.
2. Stop tcpdump on VPN and ctftimer on Controller. 
3. Run the provided backup scripts on VPN (`backup-logs.sh`), controller and backup (`backup-all.sh`). 
4. Download all backups (`/root/backups` on all machines).
5. Deploy the final scoreboard (from backup archive): upload the archive on the registration page and change `CONFIG_SCOREBOARD_URL` in the registration page config (`/etc/uwsgi/apps-available/saarctf-webpage.ini`). Spread the word and change `scoreboard.ctf.saarland` DNS entry (point it to saarsec again).
6. Take a screenshot from the Hetzner cloud console showing the accounted traffic used by all machines.
7. Remove controller and backup server. 
8. Depending on your internet speed:
   - Download the pcaps from VPN server, then remove VPN server
   - Remove VPN server, remount the volume to a cheap server, and download pcaps from there.
9. After you validated the integrity of the pcaps, remove the volume (also billed hourly).
10. If you want you can also remove the snapshots/images. 
11. Ensure no billable resources retain in the Cloud console (except the network, that's free). 
12. Upload your results to CTFTime (find a JSON file in the controller backup folder). 


