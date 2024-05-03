# Lab Computer Remote Desktop & SSH

Remote Desktop and SSH have been setup on the lab computer.

## Remote Desktop

1. Download [AnyDesk](https://anydesk.com) on your computer.
2. Enter the desk ID. Connect to it and type in the password (check "Login Automatically" so you don't have to enter the password again for future sessions). Windows and Ubuntu have different credentials.
3. When logging in:
    - After typing in the password and pressing enter, the screen will go black or will appear to hang for about 30 seconds. Wait until you see the desktop.
    - Keep the AnyDesk window on the lab computer open, as it will end your session and AnyDesk will have to be restarted in-person.
4. At the end of your session:
    - Log out (do not sleep/suspend/shut down, see below).
    - You will be able to log back in remotely.

> [!ATTENTION]
> If you sleep/suspend/shut down, you will not be able to log back in remotely. The computer would need to be woken up in-person.
>
> All remote desktop activity is visible on the monitor connected to the computer. Someone in the room could interfere with your work.
>
> If you restart the computer:
> - By default, it will boot into Ubuntu (even if you were last working on Windows). Booting into Windows needs to be done in-person.
> - You will lose AnyDesk connection for about one minute after you initiate the restart. You need to close the session and start a new one after a minute.

## Traditional SSH

To SSH in the traditional manner, you need to be on Duke's network.
1. Join Duke's Network
    - Join Duke campus WiFi OR
    - Join [Duke's VPN](https://oit.duke.edu/service/vpn/) from anywhere else.
- `ssh drc@<lab-computer-ip>`
- Works on both Windows and Ubuntu (same IP).

## AnyDesk SSH

You can also SSH through AnyDesk's TCP tunneling feature **without being on Duke's network**. The following steps work for both Windows and Ubuntu.
1. Follow the steps [here](https://blog.anydesk.com/anydesk-tcp-tunneling/) to setup the TCP tunnel. The remote port must be 22, but the local port can be any one of your choosing.
2. Start an AnyDesk remote session. **TCP tunnel will only work after the remote session has started.**
3. `ssh -p <LOCAL_PORT> drc@localhost`

## Passwordless SSH

To avoid entering your password every time you SSH, you can generate SSH keys and copy them to the lab computer.
- Use `ssh-copy-id` on Mac/Linux to copy your SSH key to Ubuntu.
- To add your SSH key to Windows, `ssh-copy-id` will not work. You will have to manually copy your key into the file `C:\Users\drc\.ssh\authorized_keys`. Add a new line with your key. It is recommended to use `id_ed25519.pub`.