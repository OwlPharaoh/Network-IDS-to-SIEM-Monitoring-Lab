## IPs and Hostnames
192.168.2.23    Wazuh Manager on Ubuntu Server  
192.168.2.24    Wazuh Agent on Windows Server 2025 (Agent ID 001)
192.168.2.19    Wazuh Agent on  Kali Linux (Agent ID 002)

## Confirm VM Connectivity
- Utilize ping, traceroute commands to confirm connectivity of VMs.

## Steps to Install Wazuh Manager on Ubuntu Server
- Installed UTM Hypervisor on Mac
- Downloaded and virtualized Ubuntu Server
- Changed network adapter setting to bridged adapter, so both Mac and windows host can communicate, as both are connected to the same router.
- Run sudo apt-get full-upgrade -y to get latest updates on Ubuntu Server
- Following Wazuh's official documentation, use the installation assistant by typing 'curl -sO https://packages.wazuh.com/4.12/wazuh-install.sh && sudo bash ./wazuh-install.sh -a' (NB: You'll need root permission). For more information, you can visit: https://documentation.wazuh.com/current/quickstart.html

- Install the GNOME desktop on the Ubuntu server using: 
    sudo apt tasksel dialog
    sudo tasksel [which opens a text-based dialog where you can select the best desktop eviroment for your server. (I used GNOME for this lab)], then 
    sudo reboot.

- Using firefox, visit https://<wazuh-manager_IP_address>:443, enter the credentials provided after the installation of Wazuh Manager in 11

## Steps to Install Wazuh Agent on Kali Linux
- From the Wazuh web interface on the Wazuh Manager, select deploy agent and provide the requested information: IP address of the server where Wazuh Manager is hosted and the IP address and name of the VM where the agent will be installed.
- A script will be provided , which will be run using root permission in the terminal.
- Wazuh agent installs sucessfully, then you need to run the following commands: 
    systemctl daemon-reload
    systemctl enable wazuh-agent
    systemctl start wazuh-agent
- Wazuh agent is successfully configured on Kali Linux VM
- Verify on Wazuh dashboard on Ubuntu Server, the new agent should be listed as Active.


## Steps to Install Wazuh Agent on Windows Server 2025
- From the Wazuh web interface on the Wazuh Manager, select deploy agent and provide the requested information: IP address of the server where Wazuh Manager is hosted and the IP address and name of the VM where the agent will be installed.
- A script will be provided , which will be run using admin rights in PowerShell.
- Wazuh agent installs sucessfully, then you need to run the following commands: 
   NET START Wazuh
- Wazuh agent is successfully configured on Windows Server 2025 VM
- Verify on Wazuh dashboard on Ubuntu Server, the new agent should be listed as Active.


## Steps to Install Sysmon on Windows Server 2025
- Download Sysmon fron the Microsoft site: 'https://learn.microsoft.com/en-us/sysinternals/downloads/sysmon'
- Download a solid Sysmon config like SwiftOnSecurity's or Olaf Hartong's modular config. Copy xml file to same folder as 
- Using Powershell, navigate to the Sysmon download folder, then run:
    .\Sysmon64.exe -accepteula -i sysmonconfig.xml

- Ensure logs are being collected: look for Event IDs: 1(Process creation), 3 (Network Connection.)
- You can verify by examining Event Viewer >>>> Application and Security Logs >>> Windows >>> Sysmon >>> Operational

- If for some reason that fails, on the Wazuh Manager, check this file: '/etc/ossec/ossec.conf'
- Ensure it contains:
    <wodle name="eventchannel">
        <enabled>yes</enabled>
        <channel>Microsoft-Windows-Sysmon/Operational</channel>
        <query>Event/System[EventID=1 or EventID=3 orEventID=11]</query>
    </wodle>
    Then restat Wazuh Manager using:
        sudo systemctl restart wazuh-manager



