# Validation notes

Commands are grouped by where they ran. These are lab observations, not a fully automated build guide.

## Windows PowerShell

```powershell
route print -4
# Confirmed a route for 10.60.2.0/24 through the Tailscale interface

tailscale ping 100.80.101.20
# Observed DERP first, then a direct connection

Test-NetConnection 10.60.2.70 -Port 22
# Observed: InterfaceAlias Tailscale, TcpTestSucceeded True

ssh -i "$env:USERPROFILE\.ssh\tslab-app" ec2-user@10.60.2.70
# Observed: interactive login (the private key stays on my laptop)
```

## Router (Session Manager shell)

```bash
sysctl net.ipv4.ip_forward
# Observed: net.ipv4.ip_forward = 1

sudo cat /etc/sysctl.d/99-tslab-forwarding.conf
# Holds net.ipv4.ip_forward = 1 so the setting is saved. A reboot was not tested.

sudo tailscale set --advertise-routes=10.60.2.0/24
# Observed: route advertised and subsequently approved in Tailscale admin console

ping -c 4 10.60.2.70
# Observed: 100% packet loss. The app security group does not allow ICMP.

timeout 5 bash -c 'echo > /dev/tcp/10.60.2.70/22' && echo 'TCP 22 reachable' || echo 'TCP 22 not confirmed'
# Observed: TCP 22 reachable
```

## Private app (SSH shell)

```bash
python3 --version
# Observed: Python 3.9.25

echo '<h1>Success! Private AWS App via Tailscale</h1><p>This server has no public IP address.</p>' > index.html
python3 -m http.server 8080 --bind 0.0.0.0
# Observed: Serving HTTP on 0.0.0.0 port 8080
```

On Windows, browse to `http://10.60.2.70:8080` while the server is running. The page loaded in this lab.

## AWS configuration checks

The AWS configuration was confirmed with read only AWS CLI queries, for example:

```bash
aws ec2 describe-instances --filters Name=vpc-id,Values=<vpc-id> \
  --query 'Reservations[].Instances[].[Tags[?Key==`Name`]|[0].Value,PrivateIpAddress,PublicIpAddress,SubnetId]' --output table

aws ec2 describe-route-tables --filters Name=vpc-id,Values=<vpc-id>

aws ec2 describe-security-groups --filters Name=vpc-id,Values=<vpc-id>
```

What these confirmed: both instances are in the lab VPC and the expected subnets, the app has no public IPv4 address, the public route table sends `0.0.0.0/0` to the internet gateway, the private route table has only the local route, and the app security group allows TCP 22 and 8080 from the router security group only.
