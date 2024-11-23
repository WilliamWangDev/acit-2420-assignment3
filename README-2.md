# ACIT-2420-Assignment-3

This project sets up a systemd service and timer to generate an `index.html` file containing some system information, which is served by an `nginx` web server on an Arch Linux droplet. A firewall is configured using `ufw` for security.

## Setup Instructions

### 1. System User Setup

1. **Create the system user `webgen`** -*[1]*
   ```bash
   sudo useradd -r -d /var/lib/webgen -s /usr/sbin/nologin webgen
   ```
   - `-r` creates a system user
   - `-d` specifies the home directory
   - `-s` specifies the login shell (a login shell for a non-login user) -*[2]*

2. **Create the directories**
   ```bash
   sudo mkdir /var/lib/webgen
   sudo mkdir /var/lib/webgen/html
   ```

3. **Change the ownership of the directories**
   ```bash
   sudo chown -R webgen:webgen /var/lib/webgen
   ```

### 2. Service and Timer Setup

1. **Create the service file**
   
   Save the following as `/etc/systemd/system/generate-index.service` -*[5]*
   ```bash
   [Unit]
   Description=Generate system information HTML file
   After=network-online.target
   Wants=network-online.target

   [Service]
   User=webgen
   Group=webgen
   ExecStart=/var/lib/webgen/bin/generate_index
   ```

2. **Create the timer file**
   
   Save the following as `/etc/systemd/system/generate-index.timer`
   ```bash
   [Unit]
   Description=Run generate-index service everyday at 05:00

   [Timer]
   OnCalendar=*-*-* 05:00:00
   Persistent=true

   [Install]
   WantedBy=timers.target
   ```

3. **Enable and start the timer**
   ```bash
   sudo systemctl daemon-reload
   sudo systemctl enable generate-index.timer
   sudo systemctl start generate-index.timer
   ```

### 3. Nginx Setup

1. **Install nginx**
   ```bash
   sudo pacman -S nginx
   ```

2. **Modify nginx.conf** -*[6]*
   
   Open `/etc/nginx/nginx.conf` and set:
   ```bash
   user webgen;
   ```

3. **Create required directories**
   ```bash
   sudo mkdir /etc/nginx/sites-available 
   sudo mkdir /etc/nginx/sites-enabled
   ```

4. **Edit the Main nginx.conf** -*[7]*
   ```bash
   sudo nvim /etc/nginx/nginx.conf
   ```
   Add to the end of the http block:
   ```bash
   include /etc/nginx/sites-enabled/*;
   ```
   Example http block:
   ```bash
   http {
       ...
       include /etc/nginx/sites-enabled/*;
   }
   ```

5. **Create a Server Block File**
   ```bash
   sudo nvim /etc/nginx/sites-available/server-setup
   ```
   Add the following content:
   ```bash
   server {
       listen 80;
       listen [::]:80;
       server_name 164.92.126.16; # Your droplet's IP address

       root /var/lib/webgen/HTML;
       index index.html;

       location / {
           try_files $uri $uri/ =404;
       }
   }
   ```

6. **Enable the site**
   ```bash
   sudo ln -s /etc/nginx/sites-available/server-setup /etc/nginx/sites-enabled/
   ```

7. **Test and restart nginx**
   ```bash
   sudo nginx -t
   sudo systemctl restart nginx
   ```

### 4. Firewall Setup

1. **Install ufw**
   ```bash
   sudo pacman -S ufw
   ```

2. **Enable and start the service**
   ```bash
   sudo systemctl enable --now ufw.service
   ```

3. **Configure firewall rules**
   ```bash
   sudo ufw allow ssh
   sudo ufw allow http
   sudo ufw limit ssh
   sudo ufw enable
   ```

4. **Check firewall status**
   ```bash
   sudo ufw status verbose
   ```
   Expected output:
   ```bash
   To                         Action      From
   --                         ------      ----
   80/tcp                     ALLOW IN    Anywhere
   22/tcp                     LIMIT IN    Anywhere
   80/tcp (v6)                ALLOW IN    Anywhere (v6)
   22/tcp (v6)                LIMIT IN    Anywhere (v6)
   ```

### 5. Generate HTML

1. **Create the generate_index script**
   
   Create `/var/lib/webgen/bin/generate_index`:
   ```bash
   #!/bin/bash

   set -euo pipefail

   # this is the generate_index script
   # you shouldn't have to edit this script

   # Variables
   KERNEL_RELEASE=$(uname -r)
   OS_NAME=$(grep '^PRETTY_NAME' /etc/os-release | cut -d '=' -f2 | tr -d '"')
   DATE=$(date +"%d/%m/%Y")
   PACKAGE_COUNT=$(pacman -Q | wc -l)
   OUTPUT_DIR="/var/lib/webgen/HTML"
   OUTPUT_FILE="$OUTPUT_DIR/index.html"

   # Ensure the target directory exists
   if [[ ! -d "$OUTPUT_DIR" ]]; then
       echo "Error: Failed to create or access directory $OUTPUT_DIR." >&2
       exit 1
   fi

   # Create the index.html file
   cat <<EOF > "$OUTPUT_FILE"
   <!DOCTYPE html>
   <html lang="en">
   <head>
       <meta charset="UTF-8">
       <meta name="viewport" content="width=device-width, initial-scale=1.0">
       <title>System Information</title>
   </head>
   <body>
       <h1>System Information</h1>
       <p><strong>Kernel Release:</strong> $KERNEL_RELEASE</p>
       <p><strong>Operating System:</strong> $OS_NAME</p>
       <p><strong>Date:</strong> $DATE</p>
       <p><strong>Number of Installed Packages:</strong> $PACKAGE_COUNT</p>
   </body>
   </html>
   EOF

   # Check if the file was created successfully
   if [ $? -eq 0 ]; then
       echo "Success: File created at $OUTPUT_FILE."
   else
       echo "Error: Failed to create the file at $OUTPUT_FILE." >&2
       exit 1
   fi
   ```

2. **Configure and test the script**
   ```bash
   sudo chmod +x /var/lib/webgen/bin/generate_index
   /var/lib/webgen/bin/generate_index
   nvim /var/lib/webgen/HTML/index.html
   ```

Visit your droplet's IP address in a web browser to see the generated `index.html` file.

**Congratulations!** If you see the system information displayed, you have successfully set up all the components of this project!

## References

1. [Create System User](https://unix.stackexchange.com/questions/233064/how-to-add-system-local-user-like-mysql-or-tomcat)
2. [The nologin Shell](https://www.baeldung.com/linux/create-non-login-user)
3. [Check System User Creation](https://unix.stackexchange.com/questions/233064/how-to-add-system-local-user-like-mysql-or-tomcat)
4. [Find Out to What Group a File Belongs](https://www.oreilly.com/library/view/wyntk-unix-system/1565921046/ch04s07.html#:~:text=of%20O'Reilly.-,Use%20ls%20%2Dlg%20to%20find%20out%20to%20what%20group%20a,ls%20%2Dlg%20on%20the%20file.)
5. [Unit File User/Group Identity](https://www.freedesktop.org/software/systemd/man/latest/systemd.exec.html)
6. [Nginx - Running Under Different User](https://wiki.archlinux.org/title/Nginx#Running)
7. [Nginx - Managing Server Entries](https://wiki.archlinux.org/title/Nginx)
