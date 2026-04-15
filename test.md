# CONFIGURATION PROXMOX

installation a partir de cle bootable  
choix hostname et Ip  
config wifi : 
sudo /usr/sbin/grub-mkconfig -o /boot/grub/grub.cfg pour mettre à jour le grub proxmox
ip link set wlp4s0 up && wpa_supplicant -B -i wlp4s0 -c /etc/wpa_supplicant.conf && dhclient wlp4s0 démarrer wifi
dhclient -r wlp4s0 && pkill wpa_supplicant && ip link set wlp4s0 down couper proprement wifi
