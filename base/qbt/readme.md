# qbt and wireguard running in docker

This runs wireguard and qbittorrent in k8s, with a network policy to block any other traffic. It allows for easier traffic shaping and qos so your [raspberry pi](https://downloads.raspberrypi.org/rss.xml) or [distro hopping](https://linuxtracker.org/index.php?page=torrents&category=563) images don't use all your bandwidth.

## Overview

The configuration here is extremely simple with no "smarts" added for automatic config fetching/configuration. It simply uses a static wireguard config, and static routes for it to connect to your vpn provider, self-hosted vps, or to your second homes network.



