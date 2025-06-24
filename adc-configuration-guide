# ADC Configuration Guide for Web and SQL Server Architecture

**Date:** 2025-06-24
**Prepared for:** RaiPragati

## Overview

This guide provides step-by-step instructions to reconfigure your existing infrastructure according to the high-availability architecture. We'll redirect traffic from your firewall to the ADC load balancer and configure proper redundancy for both web and database tiers.

## Current State
- VM1 (192.168.1.11): Web app and API running
- VM3 (192.168.1.13): SQL Database (primary)
- Firewall: Currently redirecting abc.com directly to VM1

## Target State
- Traffic for abc.com redirected from firewall to ADC (192.168.1.10)
- ADC distributing traffic between VM1 and VM2
- SQL Server configured with high availability between VM3 and VM4

## Configuration Steps

### 1. Set Up the ADC Load Balancer

1. **Install and Configure ADC**
   - Deploy ADC as a virtual appliance on either physical server
   - Assign the static IP address 192.168.1.10
   - Configure network settings (subnet mask, gateway, DNS)
   - Install necessary SSL certificates for abc.com

2. **Create Web Server Pool**
   - Log into ADC management interface
   - Navigate to Load Balancing > Server Pools
   - Create a new server pool named "WebAppPool"
   - Add VM1 (192.168.1.11) as the first server
   - Configure health checks (HTTP/HTTPS probe to specific endpoint)
   - Set initial weight to 100%

3. **Create Virtual Service**
   - Navigate to Load Balancing > Virtual Services
   - Create a new virtual service for abc.com
   - Bind to IP 192.168.1.10
   - Configure ports (80/443)
   - Select "WebAppPool" as the server pool
   - Configure SSL offloading if desired
   - Set session persistence based on your application needs

### 2. Modify Firewall Rules

1. **Update External Firewall Rule**
   - Log into your firewall management interface
   - Locate the rule that forwards abc.com traffic to VM1
   - Modify the destination to point to ADC (192.168.1.10) instead of VM1
   - Update both HTTP (port 80) and HTTPS (port 443) rules
   - Apply changes and verify traffic flows to ADC

2. **Configure Internal Firewall Rules**
   - Ensure the following communication paths are allowed:
     - Firewall → ADC (192.168.1.10): ports 80, 443
     - ADC → VM1 (192.168.1.11): ports 80, 443
     - ADC → VM2 (192.168.1.12): ports 80, 443
     - Web VMs → SQL VMs: port 1433 (SQL)

### 3. Set Up VM2 as a Redundant Web Server

1. **Provision VM2**
   - Create VM2 (192.168.1.12) on Server B if not already created
   - Install the same OS as VM1
   - Configure network settings

2. **Deploy Web Application**
   - Install required web server software (IIS/Apache/Nginx)
   - Deploy application code (same version as VM1)
   - Configure application settings to point to the correct database

3. **Synchronize Configuration**
   - Ensure both web servers have identical configurations
   - Set up a file synchronization mechanism for content updates
   - Configure session state to be shared or sticky sessions on ADC

4. **Add VM2 to ADC Pool**
   - In the ADC management interface
   - Navigate to the "WebAppPool" created earlier
   - Add VM2 (192.168.1.12) as a second server
   - Start with a lower weight (e.g., 20%) and gradually increase
   - Verify health checks pass for VM2

### 4. Configure SQL Server High Availability

1. **Provision VM4 for Secondary SQL Server**
   - Create VM4 (192.168.1.14) on Server B if not already created
   - Install the same OS and SQL Server version as VM3
   - Configure network settings

2. **Prepare SQL Server for High Availability**
   - On both VM3 and VM4:
     - Install Windows Server Failover Clustering feature
     - Configure SQL Server service accounts
     - Enable SQL Server Network Configuration

3. **Choose and Configure HA Solution**

   **Option 1: Always On Availability Groups (Recommended)**
   ```
   -- On Primary (VM3)
   -- Enable Always On
   sp_configure 'show advanced options', 1;
   GO
   RECONFIGURE;
   GO
   sp_configure 'hadr enabled', 1;
   GO
   RECONFIGURE;
   GO

   -- Create the Availability Group
   CREATE AVAILABILITY GROUP [AG_WebApp]
   WITH (AUTOMATED_BACKUP_PREFERENCE = SECONDARY)
   FOR DATABASE [YourDatabaseName]
   REPLICA ON 
      'VM3' WITH (ENDPOINT_URL = 'TCP://VM3:5022', 
         FAILOVER_MODE = AUTOMATIC,
         AVAILABILITY_MODE = SYNCHRONOUS_COMMIT,
         BACKUP_PRIORITY = 50,
         SECONDARY_ROLE(ALLOW_CONNECTIONS = READ_ONLY)),
      'VM4' WITH (ENDPOINT_URL = 'TCP://VM4:5022',
         FAILOVER_MODE = AUTOMATIC,
         AVAILABILITY_MODE = SYNCHRONOUS_COMMIT,
         BACKUP_PRIORITY = 50,
         SECONDARY_ROLE(ALLOW_CONNECTIONS = READ_ONLY));
   GO
   ```

   **Option 2: Failover Cluster Instance**
   - Create a Windows Server Failover Cluster
   - Set up shared storage accessible by both VM3 and VM4
   - Install SQL Server as a clustered instance

   **Option 3: Log Shipping**
   - Configure full backup on primary
   - Set up log shipping jobs with 15-minute frequency

4. **Update Web Application Connection Strings**
   - Modify connection strings on VM1 and VM2 to use the listener address (for AG) or virtual network name (for FCI)
   - Example for Always On AG:
   ```
   Server=AG_WebApp_Listener,1433;Database=YourDatabaseName;Integrated Security=True;MultiSubnetFailover=True
   ```

### 5. Testing and Verification

1. **Test Web Tier Redundancy**
   - Verify application works through ADC load balancer
   - Test failover by shutting down VM1
   - Confirm application remains accessible through VM2

2. **Test SQL Server Redundancy**
   - Test automatic failover (if using Always On AG or FCI)
   - Verify application continues functioning after failover
   - Test manual failback procedure

3. **Load Testing**
   - Conduct load tests to ensure proper distribution
   - Monitor response times and server health

## Monitoring Configuration

1. **Set Up Monitoring**
   - Configure monitoring for:
     - ADC health and performance
     - Web server metrics
     - SQL Server metrics
     - Overall application performance

2. **Configure Alerts**
   - Set up notifications for:
     - Server down events
     - High CPU/memory usage
     - Failover events
     - Application errors

## Next Steps

1. **Documentation**
   - Update network diagrams
   - Document IP addresses and server roles
   - Create runbooks for common operations

2. **Regular Testing**
   - Schedule monthly failover tests
   - Document results and improvements

3. **Backup Verification**
   - Ensure backups are running to NAS (192.168.1.20)
   - Test restoration procedures

## Support

For assistance with this configuration, please contact your system administrator or IT support team.
