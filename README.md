## Shared VPC
```
gcloud compute networks create shared-vpc --subnet-mode custom
gcloud compute networks update shared-vpc --bgp-routing-mode GLOBAL
gcloud compute networks subnets create shared-vpc-subnet1 \
--network shared-vpc --range 10.1.1.0/24 --region us-east1 \
--secondary-range shared-vpc-subnet1-services=10.0.32.0/20,shared-vpc-subnet1-pods=10.4.0.0/14
gcloud compute networks subnets create shared-vpc-subnet2 \
--network shared-vpc --range 10.2.1.0/24 --region us-east4 \
--secondary-range shared-vpc-subnet2-services=172.16.16.0/20,shared-vpc-subnet2-pods=172.20.0.0/14
gcloud compute firewall-rules create shared-vpc-allow-custom \
  --network shared-vpc \
  --allow tcp:0-65535,udp:0-65535,icmp \
  --source-ranges 10.0.0.0/8
gcloud compute firewall-rules create shared-vpc-allow-ssh-icmp \
    --network shared-vpc \
    --allow tcp:22,icmp
gcloud compute instances create shared-vpc-instance1 --zone=us-east1-b --machine-type=e2-medium \
    --network-interface=subnet=shared-vpc-subnet1,no-address \
    --shielded-secure-boot --shielded-vtpm --shielded-integrity-monitoring 

gcloud compute instances create shared-vpc-instance2 --zone=us-east4-b --machine-type=e2-medium \
    --network-interface=subnet=shared-vpc-subnet2,no-address \
    --shielded-secure-boot --shielded-vtpm --shielded-integrity-monitoring 

gcloud compute firewall-rules create shared-vpc-allow-subnets-from-on-prem \
    --network shared-vpc \
    --allow tcp,udp,icmp \
    --source-ranges 192.0.2.0/24
```

## Mock on-prem
```
gcloud compute networks create on-prem --subnet-mode custom
gcloud compute networks subnets create on-prem-subnet1 \
--network on-prem --range 192.0.2.0/24 --region us-central1
gcloud compute firewall-rules create on-prem-allow-custom \
  --network on-prem \
  --allow tcp:0-65535,udp:0-65535,icmp \
  --source-ranges 192.0.2.0/24
gcloud compute firewall-rules create on-prem-allow-ssh-icmp \
    --network on-prem \
    --allow tcp:22,icmp

gcloud compute instances create on-prem-instance1 --zone=us-central1-b --machine-type=e2-medium \
    --network-interface=subnet=on-prem-subnet1,no-address \
    --shielded-secure-boot --shielded-vtpm --shielded-integrity-monitoring

gcloud compute firewall-rules create on-prem-allow-subnets-from-shared-vpc \
    --network on-prem \
    --allow tcp,udp,icmp \
    --source-ranges 10.1.1.0/24,10.2.1.0/24
```

## Hybrid connectivity between VPC & on-prem
### HA VPN 
```
gcloud compute vpn-gateways create shared-vpc-vpn-gw1 --network shared-vpc --region "REGION"
gcloud compute vpn-gateways create on-prem-vpn-gw1 --network on-prem --region "REGION"
gcloud compute vpn-gateways describe shared-vpc-vpn-gw1 --region "REGION"
gcloud compute vpn-gateways describe on-prem-vpn-gw1 --region "Region"
gcloud compute routers create shared-vpc-router1 \
    --region "REGION" \
    --network shared-vpc \
    --asn 65001
gcloud compute routers create on-prem-router1 \
    --region "REGION" \
    --network on-prem \
    --asn 65002
```
### VPN tunnels with BGP peering
```
gcloud compute vpn-tunnels create shared-vpc-tunnel0 \
    --peer-gcp-gateway on-prem-vpn-gw1 \
    --region "REGION" \
    --ike-version 2 \
    --shared-secret [SHARED_SECRET] \
    --router shared-vpc-router1 \
    --vpn-gateway shared-vpc-vpn-gw1 \
    --interface 0
gcloud compute vpn-tunnels create shared-vpc-tunnel1 \
    --peer-gcp-gateway on-prem-vpn-gw1 \
    --region "REGION" \
    --ike-version 2 \
    --shared-secret [SHARED_SECRET] \
    --router shared-vpc-router1 \
    --vpn-gateway shared-vpc-vpn-gw1 \
    --interface 1
gcloud compute vpn-tunnels create on-prem-tunnel0 \
    --peer-gcp-gateway shared-vpc-vpn-gw1 \
    --region "REGION" \
    --ike-version 2 \
    --shared-secret [SHARED_SECRET] \
    --router on-prem-router1 \
    --vpn-gateway on-prem-vpn-gw1 \
    --interface 0
gcloud compute vpn-tunnels create on-prem-tunnel1 \
    --peer-gcp-gateway shared-vpc-vpn-gw1 \
    --region "REGION" \
    --ike-version 2 \
    --shared-secret [SHARED_SECRET] \
    --router on-prem-router1 \
    --vpn-gateway on-prem-vpn-gw1 \
    --interface 1
gcloud compute routers add-interface shared-vpc-router1 \
    --interface-name if-tunnel0-to-on-prem \
    --ip-address 169.254.0.1 \
    --mask-length 30 \
    --vpn-tunnel shared-vpc-tunnel0 \
    --region "REGION"
gcloud compute routers add-bgp-peer shared-vpc-router1 \
    --peer-name bgp-on-prem-tunnel0 \
    --interface if-tunnel0-to-on-prem \
    --peer-ip-address 169.254.0.2 \
    --peer-asn 65002 \
    --region "REGION"
gcloud compute routers add-interface shared-vpc-router1 \
    --interface-name if-tunnel1-to-on-prem \
    --ip-address 169.254.1.1 \
    --mask-length 30 \
    --vpn-tunnel shared-vpc-tunnel1 \
    --region "REGION"
gcloud compute routers add-bgp-peer shared-vpc-router1 \
    --peer-name bgp-on-prem-tunnel1 \
    --interface if-tunnel1-to-on-prem \
    --peer-ip-address 169.254.1.2 \
    --peer-asn 65002 \
    --region "REGION"
gcloud compute routers add-interface on-prem-router1 \
    --interface-name if-tunnel0-to-shared-vpc \
    --ip-address 169.254.0.2 \
    --mask-length 30 \
    --vpn-tunnel on-prem-tunnel0 \
    --region "REGION"
gcloud compute routers add-bgp-peer on-prem-router1 \
    --peer-name bgp-shared-vpc-tunnel0 \
    --interface if-tunnel0-to-shared-vpc \
    --peer-ip-address 169.254.0.1 \
    --peer-asn 65001 \
    --region "REGION"
gcloud compute routers add-interface  on-prem-router1 \
    --interface-name if-tunnel1-to-shared-vpc \
    --ip-address 169.254.1.2 \
    --mask-length 30 \
    --vpn-tunnel on-prem-tunnel1 \
    --region "REGION"
gcloud compute routers add-bgp-peer  on-prem-router1 \
    --peer-name bgp-shared-vpc-tunnel1 \
    --interface if-tunnel1-to-shared-vpc \
    --peer-ip-address 169.254.1.1 \
    --peer-asn 65001 \
    --region "REGION"
```
### Verify the hybrid connectivity 
```
gcloud compute ssh on-prem-instance1 --zone zone_name
ping -c 4 10.1.1.2
ping -c 2 10.2.1.2
```
