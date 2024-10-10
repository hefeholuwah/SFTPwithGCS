# Documentation for Setting Up SFTP with Google Cloud Storage

## Step 1: Create a Google Cloud Compute Engine VM (SFTP Server)

### Go to Google Cloud Console:
1. Visit Google Cloud Console.
2. Make sure you are logged in with your Google Cloud account.

### Create a New Compute Engine Instance:
1. Navigate to Compute Engine > VM Instances > Create Instance.
2. Configure your instance:
   - **Name**: Give your VM a recognizable name like `sftp-server`.
   - **Region and Zone**: Choose a region near you.
   - **Machine type**: Select an appropriate machine type (e.g., `e2-micro` for small usage).
   - **Boot disk**: Choose a Linux distribution (e.g., Ubuntu 20.04).
   - **Firewall**: Enable Allow HTTP traffic and Allow HTTPS traffic for future needs.
3. Click Create to launch the instance.

## Step 2: Set Up SFTP on the VM

Once the VM is created, you’ll need to set up the SFTP server on it.

### SSH into your VM:
1. In the Google Cloud Console, navigate to your VM instance and click SSH to open a terminal.

### Install OpenSSH if it's not already installed:
SSH is likely installed by default, but if it’s missing, you can install it using:
```bash
sudo apt update
sudo apt install openssh-server -y
```

### Configure SFTP on the VM:
1. Create a directory where you want to upload files via SFTP. For example:
   ```bash
   sudo mkdir -p /srv/sftp/uploads
   sudo chown $USER:$USER /srv/sftp/uploads
   ```

2. Create an SFTP user:
   ```bash
   sudo adduser sftpuser
   sudo passwd sftpuser  # Set a password
   ```

3. Add the user to the SFTP group:
   ```bash
   sudo groupadd sftp
   sudo usermod -aG sftp sftpuser
   ```

### Configure SSH to use SFTP:
1. Edit the SSH configuration file:
   ```bash
   sudo nano /etc/ssh/sshd_config
   ```
2. Add or update the following lines in the file to restrict the user to SFTP and their directory:
   ```yaml
   Match User sftpuser
     ForceCommand internal-sftp
     PasswordAuthentication yes
     ChrootDirectory /srv/sftp/uploads
     PermitTunnel no
     AllowAgentForwarding no
     AllowTcpForwarding no
     X11Forwarding no
   ```
3. Save the file and exit.

### Restart the SSH service:
```bash
sudo systemctl restart sshd
```

## Step 3: Install Google Cloud SDK on the VM

To interact with Google Cloud Storage, install the Google Cloud SDK (gcloud tool) on your VM.

### Install Google Cloud SDK:
```bash
sudo apt update
sudo apt install curl -y
curl https://sdk.cloud.google.com | bash
exec -l $SHELL
gcloud init
```
Follow the prompts to authenticate and configure the SDK.

### Authenticate to Google Cloud:
```bash
gcloud auth login
```

## Step 4: Install and Configure gcsfuse (Mount Cloud Storage to VM)

To use Google Cloud Storage as a file system on your VM, install gcsfuse to mount your Cloud Storage bucket.

### Install gcsfuse:
```bash
export GCSFUSE_REPO=gcsfuse-`lsb_release -c -s`
echo "deb http://packages.cloud.google.com/apt $GCSFUSE_REPO main" | sudo tee /etc/apt/sources.list.d/gcsfuse.list
curl https://packages.cloud.google.com/apt/doc/apt-key.gpg | sudo apt-key add -
sudo apt update
sudo apt install gcsfuse
```

### Create a Cloud Storage bucket:
If you haven't already created a Cloud Storage bucket, you can do it now:
```bash
gsutil mb gs://your-bucket-name
```

### Mount your Cloud Storage bucket:
1. Create a directory to mount your bucket:
   ```bash
   sudo mkdir /mnt/gcs-bucket
   ```
2. Mount the Cloud Storage bucket:
   ```bash
   gcsfuse your-bucket-name /mnt/gcs-bucket
   ```
3. Test the mount:
   You can now navigate to `/mnt/gcs-bucket` and see the contents of your Cloud Storage bucket.
   ```bash
   cd /mnt/gcs-bucket
   ls
   ```

## Step 5: Automate Mounting (Optional)

To automatically mount the bucket whenever the VM restarts, follow these steps:

### Edit /etc/fstab:
```bash
sudo nano /etc/fstab
```
Add the following line at the bottom:
```shell
gcsfuse#your-bucket-name /mnt/gcs-bucket fuse rw,allow_other,uid=1000,gid=1000 0 0
```
Save and exit.

## Step 6: Connect via SFTP

Use an SFTP client like FileZilla or Cyberduck:

- **Host**: Your Compute Engine VM's external IP address (found in the VM instance details).
- **Port**: 22
- **Username**: `sftpuser`
- **Password**: The password you set for `sftpuser`.

After logging in, navigate to `/mnt/gcs-bucket`, and you will be able to upload files via SFTP directly into your Google Cloud Storage bucket.

## Step 7: Security Considerations

- **Firewall Rules**: Ensure port 22 (SSH) is open in your VPC Network > Firewall rules. You can restrict access to your IP or range of IPs.
- **SSH Key Authentication**: Consider disabling password authentication and using SSH keys for better security.

## Additional Information

### Troubleshooting Permissions

If you encounter **"Permission denied"** errors when trying to create directories or access files, follow these steps:

1. **Unmount the directory** if it is already mounted:
   ```bash
   sudo umount /mnt/sparkxplorer
   ```

2. **Change ownership and permissions of the mount point**:
   ```bash
   sudo gcsfuse -o allow_other --uid=0 --gid=0 sparkxplorer /mnt/sparkxplorer
   sudo chown root:root /mnt/sparkxplorer
   sudo chmod 755 /mnt/sparkxplorer
   ```

3. **Create the uploads directory** if it doesn't exist:
   ```bash
   sudo mkdir -p /mnt/sparkxplorer/uploads
   sudo chown xplorer:xplorer /mnt/sparkxplorer/uploads
   sudo chmod 755 /mnt/sparkxplorer/uploads
   ```

4. **Check the SSH configuration for the `xplorer` user**.

5. **Restart the SSH service** after making changes:
   ```bash
   sudo systemctl restart sshd
   ```

6. **Monitor logs for errors**:
   ```bash
   sudo tail -f /var/log/auth.log
   ```

