# Remote Desktop & SSH

Remote Desktop and SSH have been setup on the lab desktop and Cthulhu's computer.

## Remote Desktop

1. Download [AnyDesk](https://anydesk.com) on your computer.
2. Enter the desk ID. Connect to it and type in the password (check "Login Automatically" so you don't have to enter the password again for future sessions). There are different credentials for each of the following:
    - Cthulhu Ubuntu
    - Lab Desktop Ubuntu
    - Lab Desktop Windows
3. When logging in:
    - If you see a black screen, click anywhere on the screen to wake it up.
    - After typing in the password and pressing enter, the screen will go black or will appear to hang for about 30 seconds. Wait until you see the desktop.
    - Keep the AnyDesk window on the computer open, as it will end your session and AnyDesk will have to be restarted in-person.
4. At the end of your session:
    - Lock screen (do not log out/sleep/shut down, see below).
    - You will be able to unlock remotely.

> [!ATTENTION]
> If you log out/sleep/shut down, you will not be able to log back in remotely. The computer would need to be woken up in-person.
>
> All remote desktop activity is visible on the monitor connected to the computer. Someone in the room could interfere with your work.
>
> If you restart the computer:
> - By default, the lab Desktop will boot into Ubuntu (even if you were last working on Windows). Booting into Windows needs to be done in-person.
> - You will lose AnyDesk connection for about one minute (lab desktop) or two minutes (Cthulhu) after you initiate the restart. You need to close the session and start a new one after a minute.

## Traditional SSH

To SSH in the traditional manner, you need to be on Duke's network.
1. Join Duke's Network
    - Join Duke campus WiFi OR
    - Join [Duke's VPN](https://oit.duke.edu/service/vpn/) from anywhere else.
2. Command
    - Lab Desktop: `ssh -p <LOCAL_PORT> drc@<lab-desktop-ip>` (same for Windows and Ubuntu)
    - Cthulhu: `ssh -p <LOCAL_PORT> robot@<cthulhu-ip>`

## AnyDesk SSH

You can also SSH through AnyDesk's TCP tunneling feature **without being on Duke's network**. The following steps work for both Windows and Ubuntu.
1. If you don't have a paid AnyDesk license, this feature is not available on newer versions of AnyDesk. You can download older versions of AnyDesk here:
    - [Windows](https://anydesk.en.uptodown.com/windows/versions)
    - [Mac](https://anydesk.en.uptodown.com/mac/versions)
2. Follow the steps [here](https://blog.anydesk.com/anydesk-tcp-tunneling/) to setup the TCP tunnel. The remote port must be 22, but the local port can be any one of your choosing.
3. Start an AnyDesk remote session. **TCP tunnel will only work after the remote session has started.**
4. Command
    - Lab Desktop: `ssh -p <LOCAL_PORT> drc@localhost` (same for Windows and Ubuntu)
    - Cthulhu: `ssh -p <LOCAL_PORT> robot@localhost`

## Passwordless SSH

To avoid entering your password every time you SSH, you can generate SSH keys and copy them to the lab computer.
- Use `ssh-copy-id` on Mac/Linux to copy your SSH key to Ubuntu on the desktop and/or Cthulhu.
- To add your SSH key to Windows on the lab Desktop, `ssh-copy-id` will not work. You will have to manually copy your key into the file `C:\Users\drc\.ssh\authorized_keys`. Add a new line with your key. It is recommended to use `id_ed25519.pub`.