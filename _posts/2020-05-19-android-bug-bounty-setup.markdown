Prerequisites:
- Install Genymotion (https://www.genymotion.com/fun-zone/)
- Setup android device with api version 9.0 and bridge network
- Install Ghidra (https://ghidra-sre.org/)
- Visual studio code decompiler plugin (https://marketplace.visualstudio.com/items?itemName=tintinweb.vscode-decompiler)
- Install Burp suite

Prepare proxy
- Get your local ip address (in my case it was 192.168.1.30, found with iwconfig) 
- Set your local machine as the proxy for the emulator
  - Settings -> Network & Internet -> Wi-Fi -> "cog wheel" -> Pen in the top -> advanced opetions -> Proxy -> Manual
- Insert proxy host with ip and set the port to 8080

Install Burp Cert
- Go to your browser and set burp as proxy. goto http://burp
- Download certificate by pressing the button in the right upper corner


Certificate pinning
