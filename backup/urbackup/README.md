# Centralized Automated Backup — Achahib Lab

UrBackup centralizes server backups while AD GPO, Rudder and SSH automation handle deployment and network configuration collection.

| Scope | Automation | Result |
|---|---|---|
| Windows servers | Domain membership → Backup_Servers security group → startup GPO → PowerShell silent MSI installation | UrBackup service installed and client registered centrally |
| Linux servers | Rudder install_urbackup technique and Deploy UrBackup Client rule | Conditional installation, service configuration and compliance reporting |
| Network equipment | Daily cron job on UrBackup server using SSH | Cisco running configuration and firewall XML archived with timestamps, compression and logs |

## Windows deployment

The MSI and PowerShell script are held on a central share. The script checks installation state and package availability, performs silent installation, configures automatic service startup and restart on failure, starts the service and writes deployment logs.

## Linux deployment

Rudder applies the install_urbackup technique to selected Linux nodes. The script checks for the client, installs if missing, sets the server connection and enables/restarts the service. Rudder reports execution and compliance per node.

## Network configuration archives

The SSH script covers core/gateway routers, switches, an access point, pfSense and OPNsense. Archives are organized per device under /backup/networking/ with dated compressed files. Central logs record start/end times and SUCCESS or ERROR for each device.

UrBackup provides centralized backup scheduling and retention management. Successful collection and registration are documented; recovery objectives and restore-test results should be recorded separately when validated.

Based on the supplied Centralized Automated Backup & Configuration Management Architecture report. Credentials and private addresses are omitted.

[Back to lab overview](../../README.md)
