# DepthAI Camera Local Testing

The following instructions provide a step-by-step guide to setting up the hardware and software required to test the DepthAI camera locally.

## Hardware
* PoE ethernet switch, with following items plugged into it:
    * DepthAI Camera (#5 would be preferable; see step 7)
    * Tenda router (plug into port 1, 2, or 3; do NOT plug into WAN)
    * Asus laptop
    * Optional: your laptop

## Software

> [!ATTENTION]
> Perform all steps on Asus laptop (or via SSH on Asus laptop; see [Tips](#tips)).

1. Check that the camera is connected and has received an IP address from the router’s DHCP server.
    1. Open a web browser, and go to `192.168.1.1`. You should see the Tenda router’s administrative interface.
    2. Open “Parental Controls” and check the “Attached Devices” list. You should see two (or three if your laptop is connected). The laptop(s) will be labeled with proper names (the Asus laptop is called “dukerobotics-laptop”). **Record the IP address of dukerobotics-laptop.**
    3. There should also be a device labeled “Unknown” – this is the camera. **Make sure you can see the camera here before proceeding. Record the camera’s IP address.**
    4. If you don’t see the camera here:
        1. Unplug and replug the camera.
        2. Go back and forth between the “Parental Controls” tab and the “Bandwidth Control” tab until the camera appears in the list of devices.
2. Make sure you can ping the camera.
    1. Open a terminal. Run `ping <camera-ip-address>`.
    2. Make sure you get a response. If you don’t, unplug and replug the camera and try again.
3. Optional, but highly recommended: Open `robosub-ros` in VSCode.
4. In a terminal, run the following command to start the docker container:
```bash
docker run -td --privileged --net=host -e ROBOT_NAME=oogway -v /home/duke-robotics/robosub-ros:/root/dev/robosub-ros -v /dev:/dev dukerobotics/robosub-ros:onboard
```
5. Update `ROS_IP`.
    1. SSH into the container (you can do `ssh onboard`).
    2. In the container, run the following command to edit the `setup_network.bash` file:`nano /opt/ros/noetic/setup_network.bash`
    3. In the `nano` editor, change the value of `ROS_IP` to the IP address of dukerobotics-laptop.
    4. Save the changes (`Ctrl + X`, then `Y`, then `Enter`)
6. Confirm new `ROS_IP`.
    1. Open a new terminal and SSH into the container.
    2. Run `echo $ROS_IP`.
    3. Confirm that the value returned is the IP address of dukerobotics-laptop. If not, the changes to the `setup_network.bash` file weren’t saved. Perform step 5 again.
    4. Close the previous terminal (that you ran `nano` in). All new terminals will have the updated `ROS_IP`.
7. Update camera’s MAC address in our code.
    1. For camera #5, this is `44:A9:2C:36:E0:11`
    2. If you are using a different camera, you will need to find its MAC address. In a new terminal, (outside the container) run `arp -n`. Find the line corresponding to the camera’s IP address, and copy the MAC address in that line. **It is strongly suggested to record this MAC address in a permanent location, along with the camera number, as it never changes throughout the lifetime of the camera.**
    3. In the code currently on `master`, replace the MAC address on line 100 in `depthai_camera_connect.py` with the new MAC address.
    4. In the updated CV refactoring code, change this in the appropriate YAML file.
8. You can now launch a DepthAI node, and it will connect to the camera.

## Explanation
- The Tenda router runs a DHCP server that assigns IP addresses to all devices on the local network (any device plugged into the switch). We first make sure it has recognized the camera and assigned it an IP address.
- We run the Docker container using host networking – this means that the Docker container’s network isn’t isolated from the host computer’s network, allowing the container to access the camera. Host networking is only available on Linux hosts, hence the Asus laptop is required.
- By default, the onboard Docker container’s IP address is set to 192.168.1.1. Since we are using host networking, the onboard container’s IP address is the same as the host computer’s IP address. This is usually fine, as the robot computer’s IP address is always 192.168.1.1. However, in this situation, that IP address belongs to the Tenda router; the router provides the Asus laptop with a different IP address. Thus, we change `ROS_IP` to the IP address of the Asus laptop.
- The `setup_network.bash` file is sourced in `~/.ros_bashrc` inside the container; `~/.ros_bashrc` is a file we have created that is configured to be sourced every time you open a new terminal and SSH into the container. Thus, after the `ROS_IP` environment variable is changed in in `setup_network.bash`, the change is in effect for all new terminals (except the original terminal that we edited the file in and any terminals created before).
- Our code connects to the DepthAI camera by finding the IP address corresponding to its MAC address, hence we need to update the MAC address the code searches for. The MAC address is burned into the ROM of the camera’s ethernet adapter – hence, it never changes for the lifetime of the camera.

## Tips
- To copy/paste in the terminal on Linux, use `Ctrl + Shift + C` and `Ctrl + Shift + V`. To copy/paste anywhere else, use `Ctrl + C` and `Ctrl + V`.
- If you prefer to work on your laptop, you can SSH into the Asus laptop to perform these steps.
    - Connect your laptop to the ethernet switch.
    - Perform step 1 to obtain the Asus laptop’s IP address, then run `ssh duke-robotics@<ip-address>`.
    - You can also SSH via VSCode.
- On the Asus laptop, close any windows you don’t need to maximize performance.

## Troubleshooting
- When performing a ROS launch, you get “RLException: Unable to contact my own server at…”
    - This means that the ROS_IP is incorrect. Perform steps 5 and 6 again.
    - If it still doesn’t work, make sure you’ve obtained the correct IP address for dukerobotics-laptop.
    - You can verify this by running `ip addr` in a terminal (outside the container). Under `enxa0cec8b6fd6b`, the laptop’s IP address is listed next to `inet`.
- You can perform a ROS launch, but can’t connect to the camera.
    - Make sure you’re using the correct MAC address and have updated the code accordingly. Perform step 7 again.
- When you start the container, you can’t launch any of our nodes.
    - This is likely because you didn’t use the command above to start the container. Kill the running container (`docker ps` and `docker kill <container-id>`). Perform step 4 again.
