# REG.RU Dynamic DNS Update Script for Asuswrt-Merlin

Confirmed works on the following model routers:
  - GT-AX11000

Features include:
  - Handling of Merlin firmware success/failure callbacks
  - Logging of JSON responses for later inspection, and
  - Configurable rate-limiting

## How to Configure

### Prerequisites
You should have your Merlin-enabled ASUS router configured for your network with Internet access. Since you've found this guide, it's also assumed you have a REG.RU account managing your own domain, and you've already created a subdomain you will use for dynamic DNS.

### Configuration Overview
Configuration of REG.RU DDNS involves changes through the router web portal as well as changes made through the router shell.
1. [Enable shell access and JFFS partition](#enable-shell-access-and-jffs-partition)
2. [Install REG.RU DDNS script](#install-regru-ddns-script)
3. [Enable custom DDNS](#enable-custom-ddns)
4. [Verification](#verification)
5. [Clean up](#clean-up)

Directions for disabling dynamic DNS and removal of the script and related files are at bottom.

##### Enable Shell Access and JFFS Partition
In the router portal, under Administration -> System,
- Persistent JFFS2 partition -> Enable JFFS custom scripts and configs: Yes
- Service -> Enable SSH: LAN only

Save the configuration. Ensure you are able to SSH into your router using your router portal credentials (or via public key crypto, depending on configuration) before continuing.

> Note: If SSH will be left enabled after installation, disallow password login, enable brute force protection, and use public keys for login to enhance security.

##### Install REG.RU DDNS Script
1. Log into your router via SSH, and navigate to `/jffs/scripts`.
2. Copy the `regru_ddns` and `.regru.example` files to that directory.
3. Rename `.regru.example` to `.regru`.
4. Edit `.regru` with your login and password.
5. Run `chmod 700 regru_ddns`.
6. Run `chmod 600 .regru`.
7. Run `./regru_ddns list`.
8. Step 7 should have resulted in the creation of a log file named `regru_ddns.log`. Open the log file and review the JSON response object, which should be a listing of your REG.RU DNS records for the zone ID specified in Step 4.
> Note: If there is an error in the log file or no log file is present, ensure permissions are correct and that the text of the script is copied accurately. Double-check your REG.RU credentials. If the error is from REG.RU, you can review the text of the error in the JSON response and look for any error code online.
9. Edit `.regru` with the DNS record information (i.e. ID, name and type) obtained from Step 8. Ensure your text matches exactly.
10. Run `./regru_ddns 8.8.8.8`.
11. Review the log file for the result of the last execution. If you see a successful response, verify against the REG.RU portal. Otherwise, review the errors and correct as necessary.
> Note: You may get a throttled response if you have queried too quickly after Step 7. The script rate-limits to one query every 5 minutes. This is configurable in the `regru_ddns` script or you can simply wait.
12. Ensure rate-limiting is working as expected by re-issuing the command in Step 10 a couple of times in quick succession and verifying that the log file shows frequent invocations are throttled.
13. If Steps 11 and 12 were successful, run `ln -s regru_ddns ddns-start`. This creates a symbolic link with the name expected by the router firmware.
##### Enable Custom DDNS
In the router portal, under WAN -> DDNS,
- Enable the DDNS client: Yes
- Server: Custom
- Host name: `the DNS host name you're using in REG.RU`
- HTTPS/SSL Certificate: None

Save the configuration.

##### Verification
If all is configured correctly, you should see:
1. A "successful" message on the router portal on saving the configuration. I believe this is determined by the `/sbin/ddns_custom_updated` commands being called properly within the `regru_ddns` script.
2. In the router portal, under System Log, you should see entries as below.
```
Nov 5 6:57 start_ddns: update CUSTOM , wan_unit 0
Nov 5 6:57 custom_script: Running /jffs/scripts/ddns-start (args: x.x.x.x ) - max timeout = 120s
Nov 5 6:57 ddns: Completed custom ddns update
```
3. A new log file for the script should have been created in a /tmp folder and it should contain a successful log entry. Find the log file by running `find / -name ddns-start.log 2>&1`.
4. The REG.RU portal should reflect the updated public IP address of your router.

> Note: If any errors occur, review the router log file and the script log file for an indication of the error or manually re-run `./regru_ddns list` and `./regru_ddns 8.8.8.8` to identify and troubleshoot.

##### Clean Up
Once everything is configured and working properly, you may delete the `regru_ddns.log` file from the `/jffs/scripts/` directory on the router. If SSH access is no longer needed, disable SSH on the router portal for security (especially if password authentication was used).

## Script Removal
To remove the script, the process is essentially reversed.
1. In the router portal, disable DDNS client and save. It may be worthwhile to restart your router to ensure any in-memory settings are cleared.
2. Log into the router via SSH and delete (in order): a) ddns-start, b) regru_ddns, c) .regru, d) regru_ddns.log, and e) ddns-start.log.
