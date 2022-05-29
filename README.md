# Integrating Pi-hole on Home Network

Pi-hole is an open-source network level ad-blocking and Internet tracker blocking application, with DNS forwarding and DHCP features. I spent time over the weekend to integrate it on my home Wifi network. This post is an artefact of the steps I took and hopefully someone else may find it useful too :)

You can deploy the Pi-hole tool on a virtual machine or you can spin it up within a docker container. I have taken the latter approach as it involves less overhead if you already have Docker set up.

## Steps:
1. I have a WSL instance running on my PC, and that is where I also have my docker set up. Use vim to create and open a docker-compose file. 

    `vim docker-compose.yml`
2. The contents of the file should be as follows:
    ```yaml
    version: "3"
    
    services:
      pihole:
        container_name: pihole
        image: pihole/pihole:latest
        ports:
          - "53:53/tcp"
          - "53:53/udp"
          - "67:67/udp"
          - "80:80/tcp"
        environment:
          TZ: 'Australia/Melbourne'
        volumes:
          - './etc-pihole/:/etc/pihole/'
          - './etc-dnsmasq.d/:/etc/dnsmasq.d/'
        cap_add:
          - NET_ADMIN
        restart: unless-stopped
    ```

  - The file specifies the pihole:latest Docker image, which will be pulled from the Docker Hub public repository. 
  - There are 4 port mappings between the host environment and the container's environment, allowing the host to communicate with the containerised application.
  - There is one environment variable passed that will be used to define the time zone.
  - There are two named volumes that the container will make use of, they contain all the necessary Pi-hole application data.
  - If you are going to use the Pi-hole as a DHCP server as well (optional), you are required to add NET_ADMIN as a container capability.
  - Lastly, you can define a restart policy for the container when it exits or when the Docker daemon restarts. In this case, I have set it to not restart when the container is stopped.


3. Now you can call the docker-compose file to pull the Pi-hole image and spin up the container.
    ```bash

      ali@DESKTOP-ET53LGC:~/github_dir/docker/pihole$ docker-compose --file docker-compose.yml up -d
      Creating network "pihole_default" with the default driver
      Pulling pihole (pihole/pihole:latest)...
      latest: Pulling from pihole/pihole
      69692152171a: Pull complete
      5075d85cb7c8: Pull complete
      3f2bd4d593c2: Pull complete
      ed00944e14aa: Pull complete
      51e85fe341a8: Pull complete
      32133e10e039: Pull complete
      3aea3aedb753: Pull complete
      65779c9f12d7: Pull complete
      8653933a4db7: Pull complete
      ca2bae9b9329: Pull complete
      9d42bda4ea2b: Pull complete
      3752d32eac3c: Pull complete
      fe38bd3dfa5c: Pull complete
      Digest: sha256:b51628bfa49b71ce4af4831b34e276693a6d647b82037151d8eb0d34da504432
      Status: Downloaded newer image for pihole/pihole:latest
      Creating pihole ... done
      ali@DESKTOP-ET53LGC:~/github_dir/docker/pihole$`
    ```
 
 4. Now you can verify that the Pi-hole container is running and stable.
    ```bash
      ali@DESKTOP-ET53LGC:~/github_dir/docker/pihole$ docker-compose ps
       Name    Command            State                                      Ports
      --------------------------------------------------------------------------------------------------------
      pihole   /s6-init   Up (health: starting)   0.0.0.0:53->53/tcp,:::53->53/tcp,
                                                  0.0.0.0:53->53/udp,:::53->53/udp,
                                                  0.0.0.0:67->67/udp,:::67->67/udp,
                                                  0.0.0.0:80->80/tcp,:::80->80/tcp
      ali@DESKTOP-ET53LGC:~/github_dir/docker/pihole$
      ali@DESKTOP-ET53LGC:~/github_dir/docker/pihole$ docker-compose ps
       Name    Command       State                                       Ports
      --------------------------------------------------------------------------------------------------------
      pihole   /s6-init   Up (healthy)   0.0.0.0:53->53/tcp,:::53->53/tcp, 0.0.0.0:53->53/udp,:::53->53/udp,
                                         0.0.0.0:67->67/udp,:::67->67/udp, 0.0.0.0:80->80/tcp,:::80->80/tcp
      ali@DESKTOP-ET53LGC:~/github_dir/docker/pihole$

    ```

  6. Since the Pi-hole container is up and healthy, we might be able to launch the GUI. But what's its IP address? Since Pi-hole is running on my PC, its IP address is the same as my PC's IP address in the Wifi interface.
    ![alt text](https://github.com/ali-qasimi/pihole/blob/32d54d6c670aaa0555c2bfd8206e936bffd22804/Screenshot%202021-08-09%20105049.png "Wifi Settings")
     <br>Looking at the settings above, we can see that the DNS server address is set to the home Wifi router's IP address at 10.0.0.138. It is automatically set by the router as it is operating as the DHCP server. We can confirm this by searching for that IP address on the browser.
    ![alt text](https://github.com/ali-qasimi/pihole/blob/5be1cc35ee548ee2a70f964fdc4795506720d776/Screenshot%202021-08-09%20111025.png "DNS Settings on the modem")
     <br> On the home router's web interface we can see that the DNS server is indeed set to itself. So all the end devices connected to the Wifi would be sending DNS queries to the modem instead of the Pi-hole server. And like some ISPs, my router supplied by my ISP has the option to edit the DNS server greyed out!
     <br><br>There are a few ways forward here: 
     <br>(a) Buy a third party home router that allows configuring the DNS server. 
     <br>(b) Keep using the same home router and instead configure the end devices to point at the Pi-hole DNS IP address instead.
     <br>For my case, I have gone for option (b).
  
  7. Since the Pi-hole server's IP address is provisioned by the DHCP server, there might be a limited lease period for this IP address or after a reboot the DHCP server might decide to provision us with a different IP address from its available pool. To prevent this, we can reserve the existing 10.0.0.200 IP address so it is always used by the Pi-hole server. Luckily, this option is available on my existing home router.
    ![alt text](https://github.com/ali-qasimi/pihole/blob/bf53ada10b08b14fd6a9bdd22124615c6cd87030/Screenshot%202021-08-09%20113851.png "Reserve IP Address")
  8. I can modify my Windows end device's DNS settings under Control Panel -> Network and Internet -> Network Connections, and open the Wifi Network Properties from right click menu. Then, the DNS settings can be accessed from the TCP/IPv4 Properties.
    ![alt text](https://github.com/ali-qasimi/pihole/blob/7192f18ec80e69e57ac02365b779fb54d82eae08/Screenshot%202021-08-09%20114857.png "DNS Settings")
    <br> In case the Pihole server goes down, we should ideally set a secondary DNS server IP as well. This could be the home router address, or a Public DNS Server like 1.1.1.1 (CloudFlare) or 8.8.8.8 (Google).
  10. Finally, querying for the 10.0.0.200 address on the browser:
    ![alt text](https://github.com/ali-qasimi/pihole/blob/4be238a7e8367ae682a0553c8ebcdff5cbb3b857/Screenshot%202021-08-08%20160515.png "First Look at the GUI")
    We can see that the GUI has launched successfully, which is great, and we can see some overall stats and graphs. To access all the features though, we should login. The default username is "admin", and a random password is generated during the installation phase of Pi-hole. However, we can set a new password of our choice by entering the docker environment and running the following commands:<br>
  ```bash
        ali@DESKTOP-ET53LGC:~/github_dir$ docker container ls
        CONTAINER ID   IMAGE                  COMMAND      CREATED        STATUS                             PORTS                                                                                                                                        NAMES
        dd095d2cf7f1   pihole/pihole:latest   "/s6-init"   23 hours ago   Up 19 seconds (health: starting)   0.0.0.0:53->53/udp, :::53->53/udp, 0.0.0.0:53->53/tcp, :::53->53/tcp, 0.0.0.0:80->80/tcp, 0.0.0.0:67->67/udp, :::80->80/tcp, :::67->67/udp   pihole
        ali@DESKTOP-ET53LGC:~/github_dir$
        ali@DESKTOP-ET53LGC:~/github_dir$
        ali@DESKTOP-ET53LGC:~/github_dir$
        ali@DESKTOP-ET53LGC:~/github_dir$
        ali@DESKTOP-ET53LGC:~/github_dir$ docker container exec -it dd095d2cf7f1  sh
        #
        #
        # bash
        root@dd095d2cf7f1:/#
        root@dd095d2cf7f1:/# pihole -a -p
        Enter New Password (Blank for no password):
        Confirm Password:
          [✓] New password set
        root@dd095d2cf7f1:/#
        root@dd095d2cf7f1:/#

  ```
  
  11. Logging in, we can now access all the features:
  
  ![alt text](https://github.com/ali-qasimi/pihole/blob/7192f18ec80e69e57ac02365b779fb54d82eae08/Screenshot%202021-08-08%20155730.png "Full Dashboard")
  
  
  ## That's it! 
    
