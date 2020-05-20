In this post I will walk through going from nothing to at setup for exploring an android app in a bug bounty setting. First we will install all the prereuisits and breifly explain what we intend to use them for. Next we will go into details about setting up the individual parts and how they are connected. We will look at static and dynamic analysis and try to outline a workflow/methodiligy for testing an android app.

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
- Burp go to proxy tab and select Options. Add a new listener to the emulator.

Install Burp Cert
- Go to your browser and set burp as proxy. goto http://burp
- Download certificate by pressing the button in the right upper corner to get cacert.der
- Set proxy for genymotion to 127.0.0.1:8080
- Locate genymation adb tool
- adb push cacert.der /sdcard/Download/

Certificate pinning
