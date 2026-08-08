## Create LXC #150
```
bash -c "$(curl -fsSL https://raw.githubusercontent.com/community-scripts/ProxmoxVE/main/ct/debian.sh)"
```

## Install tailscale addon
```
bash -c "$(curl -fsSL https://raw.githubusercontent.com/community-scripts/ProxmoxVE/main/tools/addon/add-tailscale-lxc.sh)"
```

## install `ethtool` and `networkd-dispatcher`
```
apt get ethtool networkd-dispatcher
```

## Set up UDP optimizations
```
NETDEV=$(ip -o route get 8.8.8.8 | cut -f 5 -d " ")
sudo ethtool -K $NETDEV rx-udp-gro-forwarding on rx-gro-list off
```

Make sure it persists
```
printf '#!/bin/sh\n\nethtool -K %s rx-udp-gro-forwarding on rx-gro-list off \n' "$(ip -o route get 8.8.8.8 | cut -f 5 -d " ")" | sudo tee /etc/networkd-dispatcher/routable.d/50-tailscale
sudo chmod 755 /etc/networkd-dispatcher/routable.d/50-tailscale
```

## Start tailscale and advertise routes
```
tailscale up --advertise-routes=192.168.5.0/24
```

## Enable subnet routes from the admin console
- on device with the `subnet` propery, select `...`, then **Edit route settings**.
- Under **Subnet routes**, select the routes to approve, then select **Save**.

## Add split DNS to resolve local names
- In tailscape, click **DNS** on the left menu
- Click **Add a nameserver**, **Custom...**
- Add local DNS `192.168.5.1`, click **Restrict to domain**, `lan` for domain (or as it is defined in local DNS).
- Not it is possible can ping `localmachine.lan`

