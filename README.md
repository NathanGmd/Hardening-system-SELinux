# Hardening-system-SELinux
### 3.2 Sécurisation de l’administration du serveur

Dans le cadre du renforcement de la sécurité du serveur, l’authentification SSH a été configurée avec un jeu de clés Ed25519, garantissant de meilleures performances et une sécurité accrue par rapport aux clés RSA traditionnelles.

L’accès SSH est désormais restreint aux connexions par clé, avec l’authentification par mot de passe désactivée. Par mesure de sécurité supplémentaire, seul le port 22/TCP a été ouvert dans le pare-feu, limitant ainsi l’exposition du service SSH aux connexions non autorisées.

Cette configuration permet de sécuriser efficacement l’accès au serveur tout en maintenant un niveau de flexibilité optimal pour l’administration distante.

```
public (active)
  target: default
  icmp-block-inversion: no
  interfaces: enp0s3 enp0s8
  sources:
  services:
  ports: 80/tcp 443/tcp 22/tcp
  protocols:
  forward: yes
  masquerade: no
  forward-ports:
  source-ports:
  icmp-blocks:
  rich rules:
```

80 et 443 ouvert pour le httpd plus tard.

### 3.3 Installation d’un serveur Web

```
[ngermond@localhost ~]$ sestatus
SELinux status:                 enabled
SELinuxfs mount:                /sys/fs/selinux
SELinux root directory:         /etc/selinux
Loaded policy name:             targeted
Current mode:                   enforcing
Mode from config file:          enforcing
Policy MLS status:              enabled
Policy deny_unknown status:     allowed
Memory protection checking:     actual (secure)
Max kernel policy version:      33
```

SELinux dispose de trois modes :

Enforcing : SELinux applique toutes les règles de sécurité. Les actions non autorisées sont bloquées et enregistrées dans les logs.

Permissive : SELinux ne bloque pas les actions, mais enregistre les violations dans les logs.

Disabled : SELinux est désactivé et aucune règle de sécurité n'est appliquée.

Actuellement, SELinux est en mode "Enforcing", ce qui signifie que toutes les règles de sécurité sont activement appliquées.

Si un profil SELinux est mal configuré pour un binaire en mode "Enforcing", SELinux bloquera l'exécution du binaire et enregistrera l'incident dans les logs. Cela peut empêcher le bon fonctionnement du service lié à ce binaire.

### 3.4 Modification d’un profil Selinux

```
[ngermond@localhost /]$ ps -eZ | grep httpd
system_u:system_r:httpd_t:s0        710 ?        00:00:00 httpd
system_u:system_r:httpd_t:s0        734 ?        00:00:00 httpd
system_u:system_r:httpd_t:s0        735 ?        00:00:01 httpd
system_u:system_r:httpd_t:s0        736 ?        00:00:01 httpd
system_u:system_r:httpd_t:s0        737 ?        00:00:01 httpd
```

