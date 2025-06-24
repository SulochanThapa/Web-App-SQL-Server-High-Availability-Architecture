# Web App + SQL Server High Availability Architecture

## Overview

This document outlines a comprehensive architecture for deploying a high-availability web application with Microsoft SQL Server across two physical servers. The architecture leverages virtualization to host multiple VMs on each physical server, creating a resilient infrastructure with redundancy at both the web and database tiers.

## Physical Infrastructure

| Physical Server | Resources | Purpose |
|-----------------|-----------|---------|
| Server A | CPU, RAM, Storage | Hosts Web VM1 and SQL VM3 (Primary) |
| Server B | CPU, RAM, Storage | Hosts Web VM2 and SQL VM4 (Secondary) |

## Logical Architecture

| VM | Hosted On | Purpose | IP Address |
|----|-----------|---------|------------|
| VM1 | Server A | Web/App Server | 192.168.1.11 |
| VM2 | Server B | Web/App Server | 192.168.1.12 |
| VM3 | Server A | SQL Server (Primary) | 192.168.1.13 |
| VM4 | Server B | SQL Server (Secondary/HA) | 192.168.1.14 |
| ADC | Either Server | Load Balancer (VIP) | 192.168.1.10 |
| NAS | Shared/Any | Storage for backups | 192.168.1.20 |

## Network Diagram

```
                           +----------------+
                           |                |
                           |  ADC/Load      |
                           |  Balancer      |
                           |  192.168.1.10  |
                           |                |
                           +--------+-------+
                                    |
                 +------------------+------------------+
                 |                                     |
        +--------+-------+                   +---------+------+
        |                |                   |                |
        |  Web Server    |                   |  Web Server    |
        |  VM1           |                   |  VM2           |
        |  192.168.1.11  |                   |  192.168.1.12  |
        |                |                   |                |
        +--------+-------+                   +---------+------+
                 |                                     |
                 +------------------+------------------+
                                    |
           +----------------------+-+----------------------+
           |                      |                        |
+----------+---------+   +--------+---------+    +---------+--------+
|                    |   |                  |    |                  |
| SQL Server Primary |   | SQL Server       |    | NAS Storage      |
| VM3                |   | Secondary VM4    |    | Backup Location  |
| 192.168.1.13       |   | 192.168.1.14     |    | 192.168.1.20     |
|                    |   |                  |    |                  |
+--------------------+   +------------------+    +------------------+
```

## Server Specifications

### Virtualization Platform

- **Recommendation**: Microsoft Hyper-V, VMware vSphere, or Proxmox VE
- **Considerations**:
  - Hyper-V if Windows-centric environment
  - VMware for enterprise-grade features
  - Proxmox for open-source alternative

### Hardware Recommendations

**Each Physical Server**:
- **CPU**: Minimum 8 cores (16+ recommended for production)
- **RAM**: Minimum 32GB (64GB+ recommended for production)
- **Storage**:
  - System: 2x SSD in RAID 1 (200GB+)
  - Data: 4x SSD/NVMe in RAID 10 (1TB+)
- **Network**: Dual 10Gbps NICs (recommended)
- **Power**: Redundant power supplies

## Web/App Tier Configuration

### VM1 & VM2 (Web/App Servers)

- **OS**: Windows Server 2022 or Ubuntu Server 22.04 LTS
- **Resources**:
  - vCPUs: 4-8
  - RAM: 8-16GB
  - Storage: 100GB+
- **Software**:
  - IIS/Apache/Nginx
  - .NET Core/.NET Framework/Java/Node.js (as required)
  - Application monitoring agent
- **Configuration**:
  - Web farm configuration
  - Shared application code deployment
  - Session state management (SQL Server or Redis)

### ADC/Load Balancer

- **Option 1**: Virtual appliance (Citrix ADC VPX, F5 BIG-IP Virtual Edition)
- **Option 2**: Software load balancer (NGINX Plus, HAProxy)
- **Option 3**: Windows Network Load Balancing (NLB)
- **Configuration**:
  - Health monitoring
  - SSL offloading
  - Session persistence
  - Round-robin or least connections algorithms

## Database Tier Configuration

### VM3 & VM4 (SQL Servers)

- **OS**: Windows Server 2022
- **Resources**:
  - vCPUs: 8-16
  - RAM: 16-32GB (production environments may require more)
  - Storage:
    - OS: 100GB+
    - Data: 500GB+
    - Logs: 200GB+
    - TempDB: 100GB+
- **Software**:
  - SQL Server 2022 Enterprise Edition (for full HA features)
  - SQL Server Management Studio

### High Availability Options

#### Option 1: Always On Availability Groups

- **Configuration**:
  - Synchronous commit mode between primary and secondary
  - Automatic failover capability
  - Readable secondary for reporting workloads
  - Requires Windows Server Failover Clustering (WSFC)

#### Option 2: Failover Cluster Instance (FCI)

- **Configuration**:
  - Shared storage requirement (SAN or S2D)
  - Single SQL Server instance that can fail over
  - Requires Windows Server Failover Clustering (WSFC)

#### Option 3: Log Shipping

- **Configuration**:
  - Scheduled log backups from primary to secondary
  - Manual failover process
  - Lower licensing costs if using Standard Edition

## Backup Strategy

### SQL Server Backups

- **Full Backups**: Daily to NAS (192.168.1.20)
- **Differential Backups**: Every 6 hours to NAS
- **Transaction Log Backups**: Every 15-30 minutes to NAS
- **Retention**: 30 days

### Web/App Server Backups

- **System State**: Weekly to NAS
- **Application Code**: With each deployment
- **Configuration**: Weekly to NAS

## Monitoring & Management

### Monitoring Tools

- **Infrastructure**: Prometheus + Grafana, PRTG, or Zabbix
- **SQL Server**: SQL Server Management Studio, DBATools
- **Web Servers**: Application Insights or custom monitoring

### Key Metrics

- **Web Tier**:
  - CPU, Memory, Disk usage
  - Request rates and response times
  - Error rates
  - Connection pool status

- **SQL Tier**:
  - CPU, Memory, Disk usage
  - Buffer cache hit ratio
  - Page life expectancy
  - Active sessions
  - Replication/AG health status

## Disaster Recovery

### Scenario: Primary Site Failure

1. ADC directs traffic to surviving web server
2. SQL Server fails over to secondary node
3. Alert administrators
4. Rebuild failed components as needed

### Scenario: Database Corruption

1. Stop application tier
2. Restore from latest backup
3. Apply transaction logs to minimize data loss
4. Verify data integrity
5. Resume application tier

## Implementation Plan

### Phase 1: Infrastructure Setup

1. Install and configure virtualization platform on both servers
2. Create VMs according to specifications
3. Set up network infrastructure and VLANs
4. Configure NAS for backup storage

### Phase 2: Web/App Tier Setup

1. Install OS and required software on VM1 and VM2
2. Configure web servers and deploy application code
3. Set up load balancer and test traffic distribution
4. Configure health checks and monitoring

### Phase 3: Database Tier Setup

1. Install OS and SQL Server on VM3 and VM4
2. Configure chosen high availability solution
3. Set up backup jobs to NAS
4. Validate failover and failback procedures

### Phase 4: Testing and Optimization

1. Perform load testing
2. Conduct failover testing
3. Optimize configurations based on test results
4. Document final architecture and procedures

## Security Considerations

- Implement network segmentation
- Restrict access to management interfaces
- Configure SQL Server security
- Implement encryption for data in transit and at rest
- Regular security patching

## Maintenance Procedures

### Routine Maintenance

- OS patching schedule
- SQL Server updates
- Application updates
- Backup verification

### Performance Tuning

- Regular review of performance metrics
- SQL Server index maintenance
- Application code optimization

## Conclusion

This architecture provides a robust, high-availability solution for hosting web applications with SQL Server backends. The design balances performance, availability, and cost considerations while providing multiple layers of redundancy to protect against various failure scenarios.

## Appendix

### References

- [Microsoft SQL Server Always On documentation](https://docs.microsoft.com/en-us/sql/database-engine/availability-groups/windows/overview-of-always-on-availability-groups-sql-server)
- [Web Farm Framework documentation](https://www.iis.net/downloads/microsoft/web-farm-framework)
- [Windows Server documentation](https://docs.microsoft.com/en-us/windows-server/)
