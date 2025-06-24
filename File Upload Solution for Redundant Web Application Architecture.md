# File Upload Solution for Redundant Web Application Architecture

## Current Challenge

In your existing setup:
- Users upload documents through the web app or API on VM1 (192.168.1.11)
- Files are stored locally on VM1's file system
- With the new load-balanced architecture, files uploaded to one server won't be accessible from the other

## Solution Options

### Option 1: Centralized Network File System (Recommended)

![Network File System Diagram](https://example.com/nfs-diagram.png)

1. **Set up a Shared Network File System**
   - Configure the NAS (192.168.1.20) to expose a file share via NFS or SMB
   - Mount this share on both VM1 and VM2 at the same path (e.g., `/mnt/uploads` or `D:\uploads\`)
   - Redirect all file uploads to this mounted path

2. **Implementation Steps**
   ```bash
   # On NAS (example for Linux-based NAS)
   mkdir -p /exports/webapploads
   chown -R webappuser:webappgroup /exports/webapploads
   chmod -R 755 /exports/webapploads
   
   # Configure NFS exports
   echo "/exports/webapploads 192.168.1.0/24(rw,sync,no_subtree_check)" >> /etc/exports
   exportfs -a
   systemctl restart nfs-kernel-server
   ```

   ```bash
   # On VM1 and VM2 (Linux example)
   mkdir -p /mnt/uploads
   mount -t nfs 192.168.1.20:/exports/webapploads /mnt/uploads
   
   # Add to /etc/fstab for persistence
   echo "192.168.1.20:/exports/webapploads /mnt/uploads nfs defaults 0 0" >> /etc/fstab
   ```

   ```powershell
   # On VM1 and VM2 (Windows example)
   New-Item -Path "D:\uploads" -ItemType Directory -Force
   New-SmbMapping -LocalPath "D:\uploads" -RemotePath "\\192.168.1.20\webapploads" -Persistent $true
   ```

3. **Application Code Update**
   ```csharp
   // Example C# code change
   // OLD: string uploadPath = Server.MapPath("~/uploads");
   // NEW: 
   string uploadPath = ConfigurationManager.AppSettings["SharedUploadPath"]; // "/mnt/uploads" or "D:\uploads"
   ```

   ```xml
   <!-- Web.config addition -->
   <appSettings>
     <add key="SharedUploadPath" value="/mnt/uploads" />
   </appSettings>
   ```

### Option 2: Object Storage Solution

1. **Implement an Object Storage Server**
   - Set up MinIO, Ceph, or similar on the NAS or a dedicated server
   - Configure with S3-compatible API
   - Update application to use object storage SDK

2. **Code Example**
   ```csharp
   // Using AWS SDK with S3-compatible storage
   var config = new AmazonS3Config
   {
       ServiceURL = "http://192.168.1.20:9000",
       ForcePathStyle = true
   };
   
   var client = new AmazonS3Client(
       "accessKey", 
       "secretKey", 
       config
   );
   
   await client.PutObjectAsync(new PutObjectRequest
   {
       BucketName = "uploads",
       Key = "user123/document.pdf",
       InputStream = fileStream
   });
   ```

### Option 3: Database BLOB Storage

1. **Store Files in SQL Server**
   - Modify database schema to include file data
   - Use FILESTREAM for improved performance with large files
   - Benefit: files automatically included in database high availability

2. **Implementation Example**
   ```sql
   -- SQL Server table creation with FILESTREAM
   CREATE TABLE UserDocuments
   (
       Id UNIQUEIDENTIFIER PRIMARY KEY,
       UserId INT NOT NULL,
       FileName NVARCHAR(255) NOT NULL,
       ContentType NVARCHAR(100) NOT NULL,
       UploadDate DATETIME NOT NULL DEFAULT GETDATE(),
       FileData VARBINARY(MAX) FILESTREAM NULL
   )
   ```

## Recommended Architecture

Based on your existing setup, I recommend Option 1 (Centralized Network File System) for the following reasons:

1. **Minimal code changes** - Simply redirect file paths to the mounted location
2. **Leverages existing NAS** - Uses the NAS (192.168.1.20) already in your architecture
3. **Simple implementation** - No need for complex object storage APIs
4. **Performance** - Good performance for typical document uploads

## Updated Architecture Diagram

```
                              INTERNET
                                 |
                                 |
                         +-------+--------+
                         |                |
                         |    Firewall    |
                         |                |
                         +-------+--------+
                                 |
                                 | (Traffic for abc.com)
                                 |
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
        |  VM1 (ACTIVE)  |                   |  VM2           |
        |  192.168.1.11  |                   |  192.168.1.12  |
        |  Web App+API   |                   |  Web App+API   |
        +--------+-------+                   +---------+------+
                 |                                     |
                 +------------------+------------------+
                                    |
           +----------------------+-+----------------------+
           |                      |                        |
+----------+---------+   +--------+---------+    +---------+--------+
|                    |   |                  |    |                  |
| SQL Server Primary |   | SQL Server       |    | NAS Storage      |
| VM3 (ACTIVE)       |   | Secondary VM4    |    | - Backups        |
| 192.168.1.13       |   | 192.168.1.14     |    | - Shared Files   |
|                    |   |                  |    | 192.168.1.20     |
+--------------------+   +------------------+    +------------------+
                                                    |
                         +------------------------------+
                         |                             |
                         |  /exports/webapploads       |
                         |  Mounted on VM1 and VM2     |
                         |  as /mnt/uploads            |
                         |                             |
                         +-----------------------------+
```

## Implementation Timeline

1. **Preparation (Day 1)**
   - Configure NAS file sharing
   - Test file access from both web servers
   - Back up existing uploaded files from VM1

2. **Code Updates (Day 2)**
   - Modify application code to use shared file path
   - Test upload and retrieval functionality
   - Deploy updated code to VM1

3. **Migration (Day 3)**
   - Copy existing uploads from VM1 to shared storage
   - Deploy updated code to VM2
   - Verify file access from both servers

4. **Testing (Day 4)**
   - Perform load testing with file uploads
   - Test failover scenarios
   - Verify file consistency

## Security Considerations

1. **Access Control**
   - Implement proper file permissions on the NAS
   - Use a dedicated service account for mounting shares
   - Consider encryption for sensitive files

2. **Network Security**
   - Restrict NFS/SMB access to only web server IPs
   - Implement firewall rules to protect file share traffic
   - Consider using a private network segment for file sharing

3. **Monitoring**
   - Set up alerts for share availability
   - Monitor disk space usage
   - Configure logging for file operations

## Backup Strategy

1. **File Backups**
   - Include the shared file storage in regular backup routines
   - Consider incremental backups for large file repositories
   - Test file restoration procedures

2. **Retention Policy**
   - Implement the same 30-day retention as other backups
   - Consider longer retention for critical documents

## Conclusion

This solution addresses the file upload requirements for your redundant web architecture. By centralizing file storage on your existing NAS, you ensure that files uploaded to either web server are immediately available to all servers in the farm, maintaining a consistent user experience regardless of which server handles a request.
