# DepthAI Camera Local Testing

The following instructions provide a step-by-step guide to setting up the hardware and software required to test the DepthAI camera locally. A router is not required if the Linux computer's wired network profile is setup correctly (see step 2 under [Software](#software)).

## Hardware
* PoE ethernet switch, with following items plugged into it:
    * DepthAI Camera (#5 would be preferable; see step 7)
    * Linux computer
    * Optional: Tenda router (plug into port 1, 2, or 3; do NOT plug into WAN)
    * Optional: Your computer

## Software

> [!ATTENTION]
> Perform all steps on Linux computer (or via SSH on Linux computer; see [Tips](#tips)).

1. Setup wired network profile on Linux computer for static IP. *If you are using a router, skip this step.*
    1. Open Settings > Network.
    2. Click the plus sign next to "Wired" to add a new wired connection.
    3. Under Identity, name the connection "DepthAI Camera" or something similar.
    4. Under IPv4, set the method to "Manual".
    5. Under IPv4 > Addresses, add the following:
        * Address: `169.254.1.10`
        * Netmask: `255.255.0.0`
        * Leave Gateway blank.
    6. Click "Add".
    7. Click on the new profile to select it. A checkmark should appear next to its name, meaning it is now being used.
    8. **After this is setup once, for every future setup, you do NOT need to recreate the profile. You only need to ensure that the profile you created is selected in network settings.**
    8. For the rest of the steps use the following IP addresses:
        * Linux computer: `169.254.1.10`
        * Camera: `169.254.1.222`
2. Ensure wired network profile on Linux computer is set to obtain IP from DHCP server. *If you are not using a router, skip this step.*
    1. Open Settings > Network.
    2. Click on the wired connection profile with a checkmark next to it.
    3. Under IPv4, set the method to "Automatic (DHCP)".
    4. Click "Apply".
    5. Ensure that the profile with the checkmark next to it is the one you just edited.
3. Check that the camera and Linux computer are connected and have received an IP address from the router’s DHCP server. *If you are not using a router, skip this step.*
    1. Open a web browser, and go to `192.168.1.1`. You should see the Tenda router’s administrative interface.
    2. Open “Parental Controls” and check the “Attached Devices” list. You should see two (or three if your laptop is connected). The computer(s) will be labeled with their hostnames (run `hostname` in the terminal or command prompt of a computer to find its hostname). **Record the IP address of the Linux computer.**
    3. There should also be a device labeled “Unknown” – this is the camera. **Make sure you can see the camera here before proceeding. Record the camera’s IP address.**
    4. If you don’t see the camera here:
        1. Unplug and replug the camera.
        2. Go back and forth between the “Parental Controls” tab and the “Bandwidth Control” tab until the camera appears in the list of devices.
4. Make sure you can ping the camera.
    1. Open a terminal. Run `ping <camera-ip-address>`.
    2. Make sure you get a response. If you don’t, unplug and replug the camera and try again.
5. Optional, but highly recommended: Open `robosub-ros` in VSCode.
6. In a terminal, run the following command to start the docker container. If `robosub-ros` is not located in the user's home directory, replace `~/robosub-ros` in the command with the correct path.
```bash
docker run -td --privileged --net=host -e ROBOT_NAME=oogway -v ~/robosub-ros:/root/dev/robosub-ros -v /dev:/dev dukerobotics/robosub-ros:onboard
```
7. Update `ROS_IP`.
    1. SSH into the container (you can do `ssh onboard`).
    2. In the container, run the following command to edit the `setup_network.bash` file:`nano /opt/ros/noetic/setup_network.bash`
    3. In the `nano` editor, change the value of `ROS_IP` to the IP address of the Linux computer.
    4. Save the changes (`Ctrl + X`, then `Y`, then `Enter`)
8. Confirm `ROS_IP` was updated.
    1. Open a new terminal and SSH into the container.
    2. Run `echo $ROS_IP`.
    3. Confirm that the value returned is the IP address of dukerobotics-laptop. If not, the changes to the `setup_network.bash` file weren’t saved. Perform step 7 again.
    4. Close the previous terminal (that you ran `nano` in). All new terminals will have the updated `ROS_IP`.
9. Update camera’s MAC address in our code. *If you are not using a router, skip this step.*
    1. Known MAC addresses:
        - Camera #4: `44:A9:2C:3C:3D:85`
        - Camera #5: `44:A9:2C:36:E0:11`
    2. If you are using a different camera, you will need to find its MAC address. In a new terminal, (outside the container) run `arp -n`. Find the line corresponding to the camera’s IP address, and copy the MAC address in that line. **It is strongly suggested to record this MAC address in a permanent location, along with the camera number, as it never changes throughout the lifetime of the camera.**
    3. In the code currently on `master`, replace the MAC address on line 100 in `depthai_camera_connect.py` with the new MAC address.
    4. In the updated CV refactoring code, change this in the appropriate YAML file.
10. You can now launch a DepthAI node, and it will connect to the camera.

## Explanation
- The router runs a DHCP server that assigns IP addresses to all devices on the local network (any device plugged into the switch). We first make sure it has recognized the camera and assigned it an IP address.
- If there is no DHCP server on the network (which is the case when there you are not using a router), the camera automatically assigns itself the IP address `169.254.1.222`. We ensure that the Linux computer has an IP address in the same subnet `169.254.1.10` and the netmask is set to `255.255.0.0` to allow communication between the two devices.
- We run the Docker container using host networking – this means that the Docker container’s network isn’t isolated from the host computer’s network. This is required to allow the container to access the camera, as indicated in the [DepthAI documentation](https://docs.luxonis.com/projects/api/en/latest/install/#docker). Host networking is only available on Linux hosts, hence a Linux computer is required.
- By default, the onboard Docker container’s IP address is set to `192.168.1.1`. Since we are using host networking, the onboard container’s IP address is the same as the host computer’s IP address. This is usually fine, as the robot computer’s IP address is always 192.168.1.1. However, in this situation, that IP address belongs to the router – the router provides the Linux computer with a different IP address – or the computer is using an IP address beginning with `169.254`. Thus, we change `ROS_IP` to the IP address of the Linux computer.
- The `setup_network.bash` file is sourced in `~/.ros_bashrc` inside the container; `~/.ros_bashrc` is a file we have created that is configured to be sourced every time you open a new terminal and SSH into the container. Thus, after the `ROS_IP` environment variable is changed in in `setup_network.bash`, the change is in effect for all new terminals (except the original terminal that we edited the file in and any terminals created before).
- Our code connects to the DepthAI camera by finding the IP address corresponding to its MAC address, hence we need to update the MAC address the code searches for. The MAC address is burned into the ROM of the camera’s ethernet adapter – hence, it never changes for the lifetime of the camera. This is only necessary if you are using a router; if you are not using a router, the code already attempts to connect to the static IP address.

## Tips
- To copy/paste in the terminal on Linux, use `Ctrl + Shift + C` and `Ctrl + Shift + V`. To copy/paste anywhere else, use `Ctrl + C` and `Ctrl + V`.
- If you prefer to work on your laptop, you can SSH into the Linux computer to perform these steps. It may be useful to SSH via VSCode.
- If you are using the Asus laptop, close any windows you don’t need to maximize performance.
- If you are not using a router, comment out all parts of `depthai_camera_connect.py` that attempt custom autodiscovery and DepthAI autodiscovery; keep only the part that connects to the camera using the static IP address. This will speed up the connection process.

## Troubleshooting
- When performing a ROS launch, you get “RLException: Unable to contact my own server at…”
    - This means that the ROS_IP is incorrect. Perform steps 5 and 6 again.
    - If it still doesn’t work, make sure you’ve obtained the correct IP address for dukerobotics-laptop.
    - You can verify this by running `ip addr` in a terminal (outside the container). Under `enxa0cec8b6fd6b`, the laptop’s IP address is listed next to `inet`.
- You can perform a ROS launch, but can’t connect to the camera.
    - Make sure you’re using the correct MAC address and have updated the code accordingly. Perform step 7 again.
- When you start the container, you can’t launch any of our nodes.
    - This is likely because you didn’t use the command above to start the container. Kill the running container (`docker ps` and `docker kill <container-id>`). Perform step 4 again.
