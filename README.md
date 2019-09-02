# new_installer

#!/usr/bin/env bash

## 1. Prepare Raspberry Pi
## This script is the first step for running your Bitcoin / LND / Electrum node
########################################################################################################################################
    ## It will configure the following - for both mainnet and testnet
    ## Change root password - create  user 'bitcoin' for daemon services to run under 
    ## Set the visible name your Pi on the network - Set boot to GUI - Expand Filesystem - Set Memory Split to 16
    ## Update system and install necessary software packages [apt-get install htop git curl bash-completion jq dphys-swapfile dirmngr]
    ## Provision external drive - format with ext4 - edit fstab for mounting the drive to the file system - mount the drive and set owner 
    ## Create Swap File on external drive to prolong life of the SD card 
    ## Configure Uncomplicated Firewall for Bitcoin - LND - Electrum Personal Server - Electrum Wallet for both mainnet and testnet
    ## Insall fail2ban 
    ## Increate open files limit
########################################################################################################################################

## Functions
# Hides the output of shell command 
    function suppress () { 
        /bin/rm --force /tmp/suppress.out 2> /dev/null; ${1+"$@"} > /tmp/suppress.out 2>&1 || cat /tmp/suppress.out; /bin/rm /tmp/suppress.out
    }
# Script to run 'su root -c' command in bash function script
    function su_Root {
        local firstArg=$1
        if [ $(type -t $firstArg) = function ]
        then
                shift && command sudo bash -c "$(declare -f $firstArg);$firstArg $*"
        else
                command su root -c "$@"
        fi
    }  
# Script to run 'su bitcoin -c' command in bash function script
    function su_Bitcoin {
        local firstArg=$1
        if [ $(type -t $firstArg) = function ]
        then
                shift && command sudo bash -c "$(declare -f $firstArg);$firstArg $*"
        else
                command su bitcoin -c "$@"
        fi
    }     
# Script to run 'sudo' command in bash function script
    function sudo_Root {
        local firstArg=$1
        if [ $(type -t $firstArg) = function ]
        then
                shift && command sudo bash -c "$(declare -f $firstArg);$firstArg $*"
        else
                command sudo "$@"
        fi
    }  
# Sets user 'root' and 'pi' and creates 'bitcoin user and password'
    function create_Passwords_Users {
        echo ""
        echo "The first step we will set users 'pi' and 'root to password [A]"
        echo "Then we will create a user 'bitcoin' to run the daemons (services) "
        echo "BTC - LND - Electrum all without admin rights or sys config editing" 
        echo "************************************************************************"
        echo "************************************************************************"
        echo ""
        echo "Set your user 'root' password to password [A] as explained in the directions"
            sudo passwd root
            sleep 1.0
            echo ""
        echo "Set your user 'pi' password to password [A] as well"
            sudo passwd
            sleep 1.0
            echo "" 
        echo ""
        echo "Next we will add create the user 'bitcoin', set it also to password [A]"
        echo ""
            sleep 1.0
            sudo adduser bitcoin
            sleep 3.0    
        echo ""
        echo " Users 'root' - 'pi- 'bitcoin' configured and created successfully "
    }  

# Creates a text doc on desktop with the hostname provided by the user - GLOBAL VARABLE -> USERGIVENHOSTNAME    
    function create_Hostname () {
        cd ~/Desktop
        touch hostname
        echo "$USERGIVENHOSTNAME" >> /home/pi/Desktop/hostname
        sudo rm -rf /etc/hostname./
        sudo mv /home/pi/Desktop/hostname /etc/hostname
        source /home/pi/.bashrc
        source ~/.profile
        sleep 2.0
        
    }  
# Removes default /etc/hostname file and replaces it with one creaded above - edits the /etc/hosts file - sets hostname - reboots bash / .profile / avahi-deamon      
    function change_Hosts {
        sudo sed -i -e "s/raspberrypi/$USERGIVENHOSTNAME/" /etc/hosts
        hostnamectl set-hostname "$USERGIVENHOSTNAME"
        systemctl restart avahi-daemon
    }         

# Install the necessary software packages: [apt-get install htop git curl bash-completion jq dphys-swapfile dirmngr] then apt-get update / upgrade   
    function install_Packges {
        apt-get install htop git curl bash-completion jq dphys-swapfile dirmngr --install-recommends -y    
        apt-get update -y
        apt-get upgrade -y
        apt autoremove -y
        sleep 2.0
    }    
# Install the necessary software packages: START
    function install_Packages_Start {
        echo ""
        echo "Download necessary software packages"
        echo "This will take about ~10 minutes to complete depending on your internet connection"
        echo ""
        echo "Installing packages"
        echo ""
    }
# Install the necessary software packages: SUCCESSFULLY COMPLETD
    function install_Packages_Successfull {
        echo ""
        echo "Packages installed and updated the system successfully"
        sleep 2.0
    }
# Install software packages encapsuling fuction 
    function install_Packages_Parrent {
        install_Packages_Start
        sudo_Root install_Packges
        install_Packages_Successfull
    }    
# Provision external drive - format with ext4 - edit fstab for symbolic link pointing to external drive at /media/hdd  
    function make_Drive {
        echo ""
        UUIDEXTERNALDRIVE=$( sudo blkid -s UUID -o value /dev/sda)
        echo UUID="$UUIDEXTERNALDRIVE" /media/hdd ext4 rw,nosuid,dev,noexec,noatime,nodiratime,auto,nouser,async,nofail 0 2 >> /etc/fstab

    }           
# Creating directory to add the hard disk and set the correct owner - verify the drive is mounted at /media/hdd and set owner as the user 'bitcoin' 
    function mount_Filesystem {
        echo ""
        sudo mkdir /media/hdd
        sudo mount -a   

    }
# Verify Filesystem is mounted at /media/hdd
    function verify_Mount_AND_Set_Ownership {
        echo ""
        echo "Verify 'Filesystem' on the far left is mounted at '/media/hdd' on the the far right"
        echo "___________________________________________________________________________________"
        df /media/hdd
        read -p "Press any key to proceed..."
        sudo chown -R bitcoin:bitcoin /media/hdd/
    
    }     
# Create directory 'bitcoin' on external drive and set the ownership to user 'bitcoin' 
    function makeBitcoinDirectory {
        echo ""
        echo "Please enter password [A] for user 'bitcoin'"
        su_Bitcoin "mkdir /media/hdd/bitcoin"
    }             
          

# Moving system swap file from SD card to external drive
    function move_Swap_File_Child { 
        dphys-swapfile swapoff
        dphys-swapfile uninstall
        rm -rf /etc/dphys-swapfile
        touch /etc/dphys-swapfile
        echo CONF_SWAPFILE=/media/swapfile >> /etc/dphys-swapfile
        dd if=/dev/zero of=/media/swapfile count=1000 bs=1MiB
        chmod 600 /media/swapfile
        mkswap /media/swapfile
        dphys-swapfile setup
        dphys-swapfile swapon
    }
# Install the necessary software packages: START
    function move_Swap_File_Start {
        echo ""
        echo "Moving system Swap File to external drive, this will preserve SD card life"
        echo "This will take about ~8 minutes to complete"
        echo "..."
    }    
# Encapsulating function for suppressed echo
    function move_Swap_File_Parrent {
        su_Root move_Swap_File_Child
        echo "System Swap File successfully create on /media/swapfile"
        sleep 2.0
    }
# Start the install of Uncomplicated Firewall - configure for Bitcoin, Lightning, and Electrum, - install fail2ban
    function ufw_Install_Start () {
        echo "IMPORTANT - You must forward ports on your router and / or modem as explained in the instructions"
        echo ""
        sleep 3.0
        echo "ONLY SSH and LND will be accessible from outside your home network over the WAN port"
        echo ""
        sleep 3.0
        echo "This is the prefered security model for Bitcoin"
        echo ""
        echo "Installing Uncompicated Firewall and fail2ban - this will take about ~5 minutes"
        echo ""
    }  
# Install and Configure Uncomplicated Firewall
    function ufw_Install {
        sudo apt-get install ufw
        sudo ufw default deny incoming
        sudo ufw default allow outgoing
        sudo ufw allow 22 comment 'allow SSH from anywhere over WAN'
        sudo ufw allow 23 comment 'allow SSH from anywhere over WAN'
        sudo ufw allow proto udp from 192.168.2.0/24 port 1900 to any comment 'allow local LAN SSDP for UPnP discovery'
        sudo ufw allow 9735  comment 'allow Lightning mainnet'
        sudo ufw allow 19735  comment 'allow Lightning testnet'
        sudo ufw allow 8333  comment 'allow Bitcoin mainnet'
        sudo ufw allow 18333  comment 'allow Bitcoin testnet' 
        sudo ufw allow 10009 comment 'allow LND mainnet wallet from anywhere over WAN'
        sudo ufw allow 10010 comment 'allow LND testnet wallet from anywhere over WAN'
        sudo ufw allow from 192.168.2.0/24 to any port 50002 comment 'allow eps mainnet from local lan'
        sudo ufw allow from 192.168.2.0/24 to any port 50003 comment 'allow eps mainnet from local lan'
        sudo apt-get install fail2ban -y
    }
# Finished installing and configuring Uncomplicated Firewall and fail2ban
    function ufw_Enable {
        echo ""
        echo "Enabling Uncomplicated Firewall - enter 'y' below"
        ufw enable
        echo ""
        echo "" 
    }
# Enable systemctl enable Uncomplicated Firewall 
    function ufw_Systemctrl_Enable {
        systemctl enable ufw 
    }    
# Encapsulated Uncomplicated Firewall install and configure function 
    function ufw_Install_Parrent_Function_Suppressed {
        su_Root ufw_Install  
    }
# Uncomplicated Firewall enable function 
    function ufw_Enable_Parrent_Function {
        su_Root ufw_Enable           
    }    

# Encapsulated Uncomplicated Firewall systemctrl enable 
    function ufw_Systemctrl_Parrent_Function_Suppressed {   
        su_Root ufw_Systemctrl_Enable     
    }    

# Increase File Limit - handles the six 'su root -c' commands
    function increase_File_Limit () {
        echo *    soft nofile 128000 >> /etc/security/limits.conf
        echo *    hard nofile 128000 >> /etc/security/limits.conf
        echo root soft nofile 128000 >> /etc/security/limits.conf
        echo root hard nofile 128000 >> /etc/security/limits.conf
        sed -i '/^# and here are more*/i session required pam_limits.so' /etc/pam.d/common-session
        sed -i '/^# end of pam-auth-update*/i session required pam_limits.so' /etc/pam.d/common-session-noninteractive
    
    }    
# Make a directory, '/home/pi/download' - download Bitcoin Core v0.18.1 32 bit ARM Linux 
# verify fingerprints and chucksum on-screen - then install Bitcoin Core
    function download_AND_Install_Bitcoin {
        
        mkdir /home/pi/download
        cd /home/pi/download
        wget https://bitcoincore.org/bin/bitcoin-core-0.18.1/bitcoin-0.18.1-arm-linux-gnueabihf.tar.gz
        wget https://bitcoincore.org/bin/bitcoin-core-0.18.1/SHA256SUMS.asc
        wget https://bitcoin.org/laanwj-releases.asc
        echo ""
        echo ""
        echo "Bitcoin downloaded successfully - verifing binaries next"
        echo "********************************************************"
        echo ""
        echo ""
        echo "Ensure the output lists "OK" after bitcoin-0.18.1-arm"
        echo "----------------- Right below this line here |"
        echo "----------------------------------------------------"
        sha256sum --ignore-missing --check SHA256SUMS.asc
        echo "----------------------------------------------------"
        echo ""
        echo "You can safely ignore any warnings and failures"
        echo ""
        read -p "Press any key to continue as long as the output says "OK" or ctrl + z to exit if not"
        gpg --verify SHA256SUMS.asc
        echo ""
        echo "Import the public key of Wladimir van der Laan and verify the signed checksum file"
        gpg --import ./laanwj-releases.asc
        echo ""
        gpg --refresh-keys
        echo ""
    
        echo "----------------------------------------------------------------------------"
        echo "Fingerprint MUST match ^^ :01EA 5486 DE18 A882 D4C2 6845 90C8 019E 36C2 E964"
        echo ""
        echo ""
        read -p "Press any key to proceed ONLY if the fingerprint matches and gpg says 'Good signature from Wladimir J. van der Laan'"
        echo "Type 'ctrl + z' to exit if not"
        echo "Extracting Bitcoin Core binaries"
        tar -xvf bitcoin-0.18.1-arm-linux-gnueabihf.tar.gz  
        sudo install -m 0755 -o root -g root -t /usr/local/bin bitcoin-0.18.1/bin/*
        echo ""
        echo "    Verify version below as v0.18.1"
        sleep 3.0
        echo ""
        bitcoind --version

    }
# 
# Create Bitcoin Core Directory and add a symbolic link that points to the external drive
    function create_Bitcoin_Core_Directory {
        echo ""
        
        sleep 2.5
        mkdir /media/hdd/bitcoin
        ln -s /media/hdd/bitcoin /home/bitcoin/.bitcoin
        sudo chown -R bitcoin:bitcoin /home/bitcoin/.bitcoin
        
    }   
# Start configuration download
    function download_Bitcoin_Conf_File_Start {
        echo ""
        echo "Configuring Bitcoin Core for mainnet and testnet, this will take about ~25 seconds..."
        echo ""
    }    
# Download alread configured Bitcoin configuration files for mainnet and testnet
    function download_Bitcoin_Conf_File {
        rm -rf /media/hdd/bitcoin
        mkdir /media/hdd/bitcoin
        cd /home/bitcoin/.bitcoin
        git clone https://github.com/bitlinc/bitcoin_mainnet_conf.git
        mv /home/bitcoin/.bitcoin/bitcoin_mainnet_conf/README.md /home/bitcoin/.bitcoin/bitcoin.conf
        rm -rf bitcoin_mainnet_conf 
        git clone https://github.com/bitlinc/bitcoin_testnet_conf.git
        mv /home/bitcoin/.bitcoin/bitcoin_testnet_conf/README.md /home/bitcoin/.bitcoin/bitcoin_testnet_start.conf
        rm -rf bitcoin_testnet_conf
        bitcoind -testnet -conf=/home/bitcoin/.bitcoin/bitcoin_testnet_start.conf
        sleep 20.0
        bitcoin-cli -testnet -conf=/home/bitcoin/.bitcoin/bitcoin_testnet_start.conf stop
        cd /home/bitcoin/.bitcoin
        rm -rf bitcoin_testnet_start.conf
        cd /home/bitcoin/.bitcoin/testnet3
        git clone https://github.com/bitlinc/bitcoin_testnet_conf.git
        mv /home/bitcoin/.bitcoin/testnet3/bitcoin_testnet_conf/README.md /home/bitcoin/.bitcoin/testnet3/bitcoin.conf
        rm -rf bitcoin_testnet_conf 
        

    }
# Parent function for download_Bitcoin_Conf_File function
    function download_Bitcoin_Conf_File_Complete_Parent {
        su_Bitcoin "download_Bitcoin_Conf_File"
    }
# Ensure Bitcoin is owner of all Bitcoin Core files 
    function set_Bitcoin_Owner {
        su_Root "chown -R bitcoin:bitcoin /home/bitcoin/.bitcoin/testnet3"
        su_Root "chown -R bitcoin:bitcoin /media/hdd/bitcoin"
    }          
    
# Set the RPC Username and password 
    function set_RPC_User_Pass {
        echo ""
        read -r -p "Enter the MAINNET 'rpc' username for your Pi `echo $'\n> '`" USERGIVENRPCNAMEMAINNET
        read -r -p "Enter the MAINNET 'rpc' password for your Pi `echo $'\n> '`" USERGIVENRPCPASSWORDMAINNET
        echo ""
        sed -i -e "s/username/$USERGIVENRPCNAMEMAINNET/" /home/bitcoin/.bitcoin/bitcoin.conf
        sed -i -e "s/userpassword/$USERGIVENRPCPASSWORDMAINNET/" /home/bitcoin/.bitcoin/bitcoin.conf
        echo ""
        read -r -p "Enter the TESTNET 'rpc' username for your Pi `echo $'\n> '`" USERGIVENRPCNAMETESTNET
        read -r -p "Enter the TESTNET 'rpc' password for your Pi `echo $'\n> '`" USERGIVENRPCPASSWORDTESTNET
        echo ""
        sed -i -e "s/username/$USERGIVENRPCNAMETESTNET/" /home/bitcoin/.bitcoin/testnet3/bitcoin.conf
        sed -i -e "s/password/$USERGIVENRPCPASSWORDTESTNET/" /home/bitcoin/.bitcoin/testnet3/bitcoin.conf
        echo ""
    
    }
# Set the RPC Username and password 
    function set_Bitcoind_Auto {
        sudo nano /etc/systemd/system/bitcoind_testnet.service
        
        echo ""
    
    }













## Body of code
echo ""
echo ""
echo "****************************************************" 
echo "*  Bitcoin - LND - Electrum Wallet Install Script  *"
echo "****************************************************"

sleep 0.25
# - 1: set users 'root' & 'pi' to password [A] and create user 'bitcoin' and assign it password [A]
echo "--------------------------------------------------------------------------"
echo "Step 1: Set your password [A] to 'pi' - 'root' - 'bitcoin' users"
echo "--------------------------------------------------------------------------"
    echo ""
    create_Passwords_Users   
    echo ""
    echo ""
# set system host name for Pi    
echo "--------------------------------------------------------------------------"
echo "Step 2: Set system host name used to identify your Pi over a network" 
echo "--------------------------------------------------------------------------"
    echo ""
    read -r -p "Enter the host name for your Pi `echo $'\n> '`" USERGIVENHOSTNAME
    suppress create_Hostname
    suppress change_Hosts
    echo "Your Pi host name updated was successfully set to "$USERGIVENHOSTNAME""
    sleep 2.5
    echo ""
    echo ""
# installing necessary software packages then update and upgrade the system [apt-get install htop git curl bash-completion jq dphys-swapfile dirmngr] then apt-get update / upgrade   
echo "--------------------------------------------------------------------------"
echo "Step 3: Install the necessary software packages for your Pi and updating"
echo "--------------------------------------------------------------------------"
    echo ""
    install_Packages_Start
    suppress install_Packages_Parrent
    install_Packages_Successfull
    echo ""
    echo ""
# Provision external drive - format with ext4 - edit fstab for symbolic link pointing to external drive at /media/hdd - mount the drive and set owner as the user 'bitcoin'  
echo "--------------------------------------------------------------------------"
echo "Step 4: Provisioning your external drive, "
echo "--------------------------------------------------------------------------"
echo ""
echo "VERY IMPORTANT - PLEASE READ"
    sleep 2.5
echo ""         
echo "This step will require that you have only ONE external drive connected to your Pi"
echo "The contents of the drive will be wiped clean and the drive will be formatted for Ext4"
echo "--------------------------------------------------------------------------------------"
echo "--------------------------------------------------------------------------------------"
echo ""
read -p "Press any key to resume install after you have confimred only one external drive is connected to your Pi..."
echo ""
    sleep 2.0
echo ""
echo "You will be asked to confirm you want to proceed and format the drive, enter "y" "
    echo ""
    echo ""
    sudo mkfs.ext4 /dev/sda
    sudo_Root make_Drive    
    echo "Drive created successfully"
    sleep 2.0
    echo ""
# Switching to user 'bitcoin' and creating a directiry called "bitcoin" on the external drive
echo "----------------------------------------------------------------------------------"
echo "Step 5: Verify 'Filesystem' mounted to drive - create directory 'bitcoin' on drive"
echo "----------------------------------------------------------------------------------"
    echo ""
    sudo_Root mount_Filesystem
    verify_Mount_AND_Set_Ownership
    makeBitcoinDirectory
    echo ""
    echo ""
# Moving system swap file from SD card to external drive
echo "--------------------------------------------------------------------------"
echo "Step 6: Moving location of system 'Swap File'"
echo "--------------------------------------------------------------------------"
    echo "" 
    move_Swap_File_Start
    suppress move_Swap_File_Parrent
    echo ""
    echo "System Swap File has been moved successfully"
    sleep 3.0
    echo ""
    echo ""
# Installing Uncomplicated Firewall and configuring it for Bitcoin, LND and Electrum
echo "--------------------------------------------------------------------------"
echo "Step 7: Installing and configuring Uncomplicated Firewall & fail2ban"
echo "--------------------------------------------------------------------------"
    echo ""
    ufw_Install_Start
    suppress ufw_Install_Parrent_Function_Suppressed
    ufw_Enable_Parrent_Function
    suppress ufw_Systemctrl_Parrent_Function_Suppressed
    echo ""
    echo "Uncomplicated Firewall and fail2ban have been install and configured successfully"
    sleep 3.0
    echo ""
    echo ""
# Increasing your open files limit 
echo "--------------------------------------------------------------------------"
echo "Step 8: Increasing your open files limit"
echo "--------------------------------------------------------------------------"
    echo ""
    echo "Increaing TCP connection limits"
    sudo_Root "increase_File_Limit"
    echo "Completed successfully"
    sleep 2.0
    echo ""
    echo ""    
# Increasing your open files limit 
echo "--------------------------------------------------------------------------"
echo "Step 9: Downloading, Install and verify Bitcoin Core"
echo "--------------------------------------------------------------------------"
    echo ""
    echo "Downloading and installing Bitcoin version 0.18.1 32 bit ARM"
    echo ""
    download_AND_Install_Bitcoin
    sleep 2.0
    echo ""
    echo "Bitcoin Core has been verified and installed successfully"
    echo ""
    echo ""
# Increasing your open files limit 
echo "--------------------------------------------------------------------------"
echo "Step 10: Creating Bitcoin Core Directory"
echo "--------------------------------------------------------------------------"
    echo ""
    echo "Creating Bitcoin core home directory and linking it to the external drive"
    su_Bitcoin create_Bitcoin_Core_Directory
    echo "Successfully linked Bitcoin system directory to your external drive"
    sleep 3.0
    echo ""
    echo ""
# Increasing your open files limit     
echo "--------------------------------------------------------------------------"
echo "Step 11: Configure Bitcoin Core for Mainnet and Testnet"
echo "--------------------------------------------------------------------------"
    echo ""
    echo "Configuring Bitcoin Core for mainnet and testnet, this will take about ~25 seconds..." 
    suppress download_Bitcoin_Conf_File_Complete_Parent  
    echo ""   
    echo "Please enteryour root password [A] below twice"
    echo ""
    sleep 1.0
    set_Bitcoin_Owner
    echo "Bitcoin Core was successfully configured for mainnet and testnet"
    echo ""
    echo ""
# Increasing your open files limit     
echo "--------------------------------------------------------------------------"
echo "Step 12: Set RPC Username and Password"
echo "--------------------------------------------------------------------------"
    echo ""
    echo "Configuring Bitcoin Core for mainnet and testnet, this will take about ~25 seconds..." 
    suppress download_Bitcoin_Conf_File_Complete_Parent  
    echo ""   
    echo "Please enter your RPC username and passoword for Bitcoin Mainnet [B] and Testnet [C] "
    echo ""
    sleep 1.0
    su_Bitcoin set_RPC_User_Pass
    echo ""
    echo "RPC usernam and passwords for mainnet and testnet were configured successfully"
    sleep 3.0
    echo ""
    echo ""
# Increasing your open files limit     
echo "--------------------------------------------------------------------------"
echo "Step 12: Set Bitcoind on mainnet and on testnet to system auto-start"
echo "--------------------------------------------------------------------------"
    echo ""
    echo "Configureing now"
    echo ""
    set_Bitcoind_Auto
    echo "RPC usernam and passwords for mainnet and testnet were configured successfully"
    sleep 3.0
    echo ""
    echo ""
echo "END OF SCIPT"

echo "END OF SCIPT"
