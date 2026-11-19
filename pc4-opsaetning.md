# PC4 – opsætning af monitor-PC (Wireshark + Snort)

PC4 er Thomas' ThinkPad med Arch Linux. Det kablede interface hedder `enp0s31f6`. Kommandoerne er til Linux. På Windows kan Wireshark også bruges, men Snort er sat op til Linux her.

## 1. Kabel og adresse

Sæt kablet fra ge-0/0/4 på SRX-C1 i PC4. Tjek at der er link:

```
ip -br link show enp0s31f6
```

Der skal stå `UP` og ikke `NO-CARRIER`.

Giv PC4 adressen 2.2.2.1/24 uden gateway. SRX'en sender kopierne til 2.2.2.1, så PC4 skal svare på ARP for den adresse. Med NetworkManager:

```
nmcli con add type ethernet ifname enp0s31f6 con-name monitor ipv4.method manual ipv4.addresses 2.2.2.1/24 ipv4.never-default yes ipv6.method disabled connection.autoconnect-priority 100
nmcli con up monitor
```

Uden gateway bliver Wi-Fi ved med at være standardrute, så PC4 kan stadig nå internettet.

`never-default` og `autoconnect-priority 100` er med, fordi PC4 havde andre kablede profiler fra tidligere opgaver. Uden dem valgte NetworkManager en gammel profil med en gateway der ikke findes, og så forsvandt internetforbindelsen over Wi-Fi.

Tjek forbindelsen:

```
ping -c 3 2.2.2.2
```

## 2. Wireshark

Brugeren skal være i gruppen `wireshark`, så kan der optages uden sudo. Start Wireshark og vælg `enp0s31f6`, eller optag fra terminalen:

```
dumpcap -i enp0s31f6 -a duration:30 -w dmz-mirror.pcapng
```

Brug displayfilteret `ip.addr == 192.168.4.10` for at se trafikken til og fra webserveren.

## 3. Snort 3

Snort er ikke i Arch' officielle repo. Vi bruger det officielle Docker-image fra Cisco Talos, så der ikke skal bygges noget:

```
docker pull ciscotalos/snort3
```

Kør Snort på mirror-porten med vores regler (fra mappen med `snort/local.rules`):

```
docker run --rm -it --user root --net=host --cap-add=NET_RAW --cap-add=NET_ADMIN \
  -v "$PWD/snort:/rules:ro" --entrypoint /home/snorty/snort3/bin/snort ciscotalos/snort3 \
  -c /home/snorty/snort3/etc/snort/snort.lua -R /rules/local.rules \
  -i enp0s31f6 -A alert_fast -k none
```

- `--entrypoint` er nødvendig, fordi imagets standard-entrypoint starter en bash og ignorerer kommandoen.
- `--user root` og `--cap-add` giver lov til at lytte på netkortet. Imaget kører ellers som brugeren `snorty`.
- `--net=host` giver containeren adgang til `enp0s31f6`.
- `-R` indlæser vores regler oven i standardkonfigurationen (`snort.lua` ligger i `/home/snorty/snort3/etc/snort/` i imaget).
- `-A alert_fast` skriver én linje pr. alarm i terminalen.
- `-k none` ignorerer checksumfejl. Kopierne kan have forkerte checksums, og så ville Snort ellers springe dem over.

Stop med Ctrl-C. Snort skriver en optælling af pakker og alarmer når den stopper.

Testet 2026-11-17 på `wlan0`: konfigurationen validerer med 0 advarsler, og regel 1000036 gav alarm på en SYN til 192.168.4.10.

Alternativ uden Docker: `yay -S snort` (AUR, bygger Snort 3 fra kilde, tager lang tid). Kommandoen er den samme uden `docker run`-delen.

## 4. Reglen

```
alert tcp any any -> 192.168.4.10 80 ( msg:"ASS35 TCP SYN flood mod WebServer"; flags:S; detection_filter:track by_dst, count 100, seconds 1; classtype:attempted-dos; sid:1000035; rev:1; )
```

`flags:S` matcher pakker hvor kun SYN er sat. `detection_filter` gør at reglen først giver alarm når der er kommet mere end 100 af dem til samme destination inden for samme sekund. Snort tæller i hele sekunder på uret, så de første alarmer kan komme lidt senere end efter 100 pakker (se README, "Resultat"). Normal browsing laver nogle få SYN pr. sekund og udløser den ikke.

Regel 1000036 giver alarm på hver ny forbindelse og bruges kun til at vise at Snort ser trafikken. Den kommenteres ud med `#` før SYN flood-testen, ellers fylder den terminalen.

## 5. Angriberen (PC1 flyttet til LabLan)

PC1 er Sebastians laptop med Windows. hping3 findes ikke til Windows, så vi brugte `nping`, som følger med Nmap.

Flyt PC1s kabel fra ge-0/0/11 til en LabLan-port, så den får en 10.56.x.x-adresse med DHCP. Under testen fik den 10.56.18.11.

Hvis VMware er installeret, så slå VMnet8 fra først. På Sebastians laptop havde den 10.56.16.1/24, og så sendte Windows trafikken til 10.56.16.85 ud på det virtuelle net i stedet for netkortet. Kør i PowerShell som administrator:

```
Disable-NetAdapter -Name "VMware Network Adapter VMnet8"
```

Start angrebet, stadig i PowerShell som administrator:

```
nping -e eth0 -H --tcp -p 80 --flags syn --rate 1000 -c 10000 10.56.16.85
```

Det sender 10.000 SYN-pakker med 1000 i sekundet, altså ca. 10 sekunder. `-e eth0` er det kablede netkort i nping's navngivning (`nping --iflist` viser listen). LabLan er skolens fælles net. Angrebet går mod vores egen DNAT-adresse og rammer kun vores webserver.

Slå VMnet8 til igen bagefter:

```
Enable-NetAdapter -Name "VMware Network Adapter VMnet8"
```

På Linux kan det samme gøres med hping3: `sudo hping3 -S -p 80 -i u1000 -c 10000 10.56.16.85`. Det er ikke testet her.
