# Install Bitwarden on Ubuntu Server

1. Ensure packages are all up to date
```Bash
sudo apt -y update && sudo apt -y upgrade
```

<br>

2. Install Docker and Docker compose
```Bash
sudo apt -y install ca-certificates curl gnupg lsb-release

curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu \
  $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

sudo apt-get -y update
sudo apt-get -y install -y docker-ce docker-ce-cli containerd.io docker-compose-plugin
```

<br>

3. Create a user dedicated to Bitwarden and add it to the docker and sudo groups
```Bash
sudo useradd -m -s /bin/bash bitwarden
sudo usermod -aG docker,sudo bitwarden
echo -e "YourSecurePassword\nYourSecurePassword" | sudo passwd bitwarden
```

<br>

4. Set the necessary directory permissions
```Bash
sudo mkdir /opt/bitwarden
sudo chmod -R 700 /opt/bitwarden
sudo chown -R bitwarden:bitwarden /opt/bitwarden
```

<br>

5. Download the official Bitwarden installation script using the Bitwarden user you just created
```Bash
su - bitwarden
curl -Lso bitwarden.sh "https://func.bitwarden.com/api/dl/?app=self-host&platform=linux" && chmod 700 bitwarden.sh
```

<br>

6. Run the Bitwarden installation script you just downloaded
```Bash
sudo mkdir -p /home/bitwarden/bwdata/docker
sudo chown -R bitwarden:bitwarden /home/bitwarden/bwdata

DOMAIN="bitwarden.yourdomain.com"
USE_LETSENCRYPT="n"
INSTALL_ID="your_installation_id"
INSTALL_KEY="your_installation_key"
REGION="US"

printf "$DOMAIN\n$USE_LETSENCRYPT\n$INSTALL_ID\n$INSTALL_KEY\n$REGION\n" | ./bitwarden.sh install
```

<br>

7. Start the Bitwarden service after it's finished installing
```Bash
./bitwarden.sh start
```

<br>

8. Access Bitwarden over at `http://localhost:80`

<br>

```Bash
variable "bitwarden-startup-script" {

}
```
