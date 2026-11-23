# LLD – Assignment 35 (SRX port mirroring og IDS)

Enhederne svarer til [HLD'en](../HLD.pdf).

## R1 / SRX-C1 – routede interfaces

| Interface | Adresse | Zone | Host-inbound | Tilsluttet | Bemærkning |
|---|---|---|---|---|---|
| ge-0/0/1.0 | 192.168.3.1/24 | USERLAN (trust) | all | patch → ge-0/0/10 | DHCP-server |
| ge-0/0/2.0 | 192.168.4.1/24 | DMZ | ping | patch → ge-0/0/6 | Filter `my-port-filter` input + output |
| ge-0/0/3.0 | 10.56.16.85/22 | untrust | ping | LabLan / R3 | Source NAT + destination NAT |
| ge-0/0/4.0 | 2.2.2.2/24 | MONITOR | ping | PC4 | Mirror-udgang |

## R1 / SRX-C1 – L2-porte

| Interface | VLAN | ID | Tilsluttet |
|---|---|---|---|
| ge-0/0/6 | mySwitch1Vlan (DMZ) | 4 | patch fra ge-0/0/2 |
| ge-0/0/7 | mySwitch1Vlan (DMZ) | 4 | PC3 / WebServer |
| ge-0/0/10 | mySwitch2Vlan (USERLAN) | 3 | patch fra ge-0/0/1 |
| ge-0/0/11 | mySwitch2Vlan (USERLAN) | 3 | PC1 |
| ge-0/0/12 | mySwitch2Vlan (USERLAN) | 3 | ubrugt |

Alle porte er untagged access-porte. VLANs har intet l3-interface. Routingen sker på ge-0/0/1 og ge-0/0/2 via patch-kablerne.

## PC'er

| Enhed | Sidder i | Zone | Adresse | Gateway |
|---|---|---|---|---|
| PC1 | ge-0/0/11 | USERLAN | DHCP | 192.168.3.1 |
| PC3 / WebServer | ge-0/0/7 | DMZ | 192.168.4.10/24 | 192.168.4.1 |
| PC4 / Monitor | ge-0/0/4 | MONITOR | 2.2.2.1/24 | ingen |
| PC1 under SYN flood | LabLan | – | DHCP på LabLan | 10.56.16.1 |

## Port mirroring

| Del | Værdi |
|---|---|
| Spejlet interface | ge-0/0/2.0 (DMZ-gateway) |
| Retning | input og output (begge veje) |
| Filter | `my-port-filter`, term `myMirrorTerm`: fra 0.0.0.0/0 → `count mirror-count`, `port-mirror`, `accept` |
| Sampling | `input rate 1` = hver pakke |
| Family | inet (kun IPv4) |
| Udgang | ge-0/0/4.0, next-hop 2.2.2.1 (PC4) |

Filteret slutter med `accept`, så den originale trafik går videre som før. Uden `accept` ville filteret droppe al DMZ-trafik.

## Zoner og policies

| Fra | Til | Policy | Match | Handling |
|---|---|---|---|---|
| USERLAN | untrust | permit-internet | any / any / any | permit |
| USERLAN | DMZ | permit-web | any / any / any | permit |
| untrust | DMZ | permit-web-dnat | any / webserver / junos-http | permit |
| alle andre | | default | | deny |

MONITOR har ingen policies. PC4 kan derfor ikke nå andre zoner, og ingen kan nå PC4.

## NAT

| Type | Regelsæt | Fra → til | Match | Oversættelse |
|---|---|---|---|---|
| Source | soho | USERLAN, DMZ → untrust | 0.0.0.0/0 | interface 10.56.16.85 |
| Destination | from-lablan | untrust | 10.56.16.85:80 | pool webserver-pool 192.168.4.10 |

Destination NAT sker før trafikken når ge-0/0/2. Spejlet trafik fra en DNAT-forbindelse har derfor destination 192.168.4.10 og LabLan-afsenderens rigtige adresse som kilde.

## Routing

| Rute | Next-hop |
|---|---|
| 0.0.0.0/0 | 10.56.16.1 (R3) |
| 2.2.2.0/24, 192.168.3.0/24, 192.168.4.0/24, 10.56.16.0/22 | direkte forbundet |
