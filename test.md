# CONFIGURATION PROXMOX

installation a partir de cle bootable  
choix hostname et Ip  
config wifi :  
apt download wpasupplicant iw libnl-3-200 libnl-genl-3-200  

ldconfig et start-stop-daemon existent mais ne sont pas dans ton PATH. Corrige ça d’abord :  
 export PATH=$PATH:/usr/local/sbin:/usr/sbin:/sbin  

dpkg -i libnl-3-200_3.7.0-0.2+b1_amd64.deb
dpkg -i libnl-genl-3-200_3.7.0-0.2+b1_amd64.deb
dpkg -i iw_5.19-1_amd64.deb
dpkg -i wpasupplicant_2%3a2.10-12+deb12u3_amd64.deb


apt download libnl-route-3-200 adduser  

dpkg -i adduser_*.deb
dpkg -i libnl-route-3-200_*.deb
dpkg -i wpasupplicant_*.deb  

monter linterface  
ip link set wlp4s0 up  

cree la config 
wpa_passphrase "NomDuRéseau" "MotDePasse" > /etc/wpa_supplicant.conf  

connecte 
wpa_supplicant -B -i wlp4s0 -c /etc/wpa_supplicant.conf  

dhclient wlp4s0  

# Titre h1
## Titre h2
### Titre h3

> Texte en citation/note

**gras**
`code inline`

```bash
bloc de code avec coloration syntaxique
```

---   ← ligne de séparation












sudo /usr/sbin/grub-mkconfig -o /boot/grub/grub.cfg pour mettre à jour le grub proxmox
ip link set wlp4s0 up && wpa_supplicant -B -i wlp4s0 -c /etc/wpa_supplicant.conf && dhclient wlp4s0 démarrer wifi
dhclient -r wlp4s0 && pkill wpa_supplicant && ip link set wlp4s0 down couper proprement wifi
