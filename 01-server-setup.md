# Digital Ocean Server Implementation
  1. Digital Ocean Droplet & Initial Setup Implementation
This section documents the implementation steps to set up the Digital Ocean droplet.
## A.Droplet Creation
The following configuration is implemented for the droplet:
  1. Log in to Digital Ocean Control Panel.
  2. Navigate to Manage > Droplets > Create
  3. Region: Banglore & Data Centre: BR1
  4. OS Image: Rocky Linux Version: 10x64
  5. Plan: Baisc Shared CPU with 2GB/1CPU Plan.
  6. Authenthication: Add SSH key for secure access.
  7. IPv6: Configure IPv6 by checking Advanced Options > Enable IPv6.
  8. Hostname: Set Hostname as fossee-sso-task.



B. Initial Server Hardening:

Connect to the droplet as root user for initial setup:
```
ssh root@droplet_ip 
```









