# Hardening-system-SELinux
### 3.2 Sécurisation de l’administration du serveur

Dans le cadre du renforcement de la sécurité du serveur, l’authentification SSH a été configurée avec un jeu de clés Ed25519, garantissant de meilleures performances et une sécurité accrue par rapport aux clés RSA traditionnelles.

L’accès SSH est désormais restreint aux connexions par clé, avec l’authentification par mot de passe désactivée. Par mesure de sécurité supplémentaire, seul le port 2222/TCP a été ouvert dans le pare-feu, limitant ainsi l’exposition du service SSH aux connexions non autorisées.

Cette configuration permet de sécuriser efficacement l’accès au serveur tout en maintenant un niveau de flexibilité optimal pour l’administration distante.\
\
Audit ssh par lynis du serveur :
```
[+] SSH Support
------------------------------------
  - Checking running SSH daemon                               [ FOUND ]
    - Searching SSH configuration                             [ FOUND ]
    - OpenSSH option: AllowTcpForwarding                      [ OK ]
    - OpenSSH option: ClientAliveCountMax                     [ OK ]
    - OpenSSH option: ClientAliveInterval                     [ OK ]
    - OpenSSH option: FingerprintHash                         [ OK ]
    - OpenSSH option: GatewayPorts                            [ OK ]
    - OpenSSH option: IgnoreRhosts                            [ OK ]
    - OpenSSH option: LoginGraceTime                          [ OK ]
    - OpenSSH option: LogLevel                                [ OK ]
    - OpenSSH option: MaxAuthTries                            [ OK ]
    - OpenSSH option: MaxSessions                             [ OK ]
    - OpenSSH option: PermitRootLogin                         [ OK ]
    - OpenSSH option: PermitUserEnvironment                   [ OK ]
    - OpenSSH option: PermitTunnel                            [ OK ]
    - OpenSSH option: Port                                    [ OK ]
    - OpenSSH option: PrintLastLog                            [ OK ]
    - OpenSSH option: StrictModes                             [ OK ]
    - OpenSSH option: TCPKeepAlive                            [ OK ]
    - OpenSSH option: UseDNS                                  [ OK ]
    - OpenSSH option: X11Forwarding                           [ SUGGESTION ]
    - OpenSSH option: AllowAgentForwarding                    [ OK ]
    - OpenSSH option: AllowUsers                              [ NOT FOUND ]
    - OpenSSH option: AllowGroups                             [ NOT FOUND ]
```
Configuration du firewall :
```
[ngermond@localhost ssh]$ sudo firewall-cmd --list-all
public (active)
  target: default
  icmp-block-inversion: no
  interfaces: enp0s3 enp0s8
  sources:
  services:
  ports: 80/tcp 2222/tcp
  protocols:
  forward: yes
  masquerade: no
  forward-ports:
  source-ports:
  icmp-blocks:
  rich rules:
```

80  ouvert pour le httpd plus tard.

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



Contexte du service apache :
```
[ngermond@localhost /]$ ps -eZ | grep httpd
system_u:system_r:httpd_t:s0        710 ?        00:00:00 httpd
system_u:system_r:httpd_t:s0        734 ?        00:00:00 httpd
system_u:system_r:httpd_t:s0        735 ?        00:00:01 httpd
system_u:system_r:httpd_t:s0        736 ?        00:00:01 httpd
system_u:system_r:httpd_t:s0        737 ?        00:00:01 httpd
```
A ce moment là lorsque l'on tente de se connecter au serveur web, celui-ci est accessible.\
\
Contexte des fichier apache /var/www/html :
```
[ngermond@localhost www]$ ls -Z html
unconfined_u:object_r:httpd_sys_content_t:s0 assets
unconfined_u:object_r:httpd_sys_content_t:s0 index.html
```
Contexte des fichier créer dans srv_1 /srv/srv/srv_1 :
```
[ngermond@localhost srv]$ ls -Z srv_1/
unconfined_u:object_r:var_t:s0 assets
unconfined_u:object_r:var_t:s0 index.html
```
On constate que les étiquettes change de httpd_sys_content_t vers var_t
On en conclus que selon l'endroit où les fichiers sont créés, le système leurs attribues des étiquette correspondantes. Reste à comprendre comment celui-ci gère l'attribution de ces fameuses étiquettes.
```
[core:error] [pid 744:tid 873] (13)Permission denied: [client 10.1.1.1:60325] AH00035: access to /index.html denied (filesystem path '/srv/srv/srv_1/index.html') because search permissions are missing on a component of the path
```
Comme il est possible de constater sur la ligne ci dessus, lorsque apache tente d'acceder a "index.html" dans le repertoire, srv_1, on vois qu'il reçois une permission denied.\
\
```
sudo dnf install setroubleshoot
sudo sealert -a /var/log/audit/audit.log
```
Sealert permet d'avoir une vision claire des logs, ainsi qu'avoir des suggestion de modifications afin de corriger les erreurs. Ici sealert me conseille le ligne suivante :
```
SELinux is preventing /usr/sbin/httpd from open access on the file /srv/srv/srv_1/index.html.

*****  Plugin catchall_labels (83.8 confidence) suggests   *******************

If you want to allow httpd to have open access on the index.html file
Then you need to change the label on /srv/srv/srv_1/index.html
Do
# semanage fcontext -a -t FILE_TYPE '/srv/srv/srv_1/index.html'
where FILE_TYPE is one of the following: [...]
Then execute:
restorecon -v '/srv/srv/srv_1/index.html'
```
Ce qui nous indique comment changer le contexte de mon fichier index.html de la manière suivante :
```
[ngermond@localhost srv_1]$ sudo semanage fcontext -a -t httpd_sys_content_t '/srv/srv/srv_1/index.html'
[ngermond@localhost srv_1]$ sudo restorecon -v '/srv/srv/srv_1/index.html'
Relabeled /srv/srv/srv_1/index.html from unconfined_u:object_r:var_t:s0 to unconfined_u:object_r:httpd_sys_content_t:s0
[ngermond@localhost srv_1]$ sudo setenforce 1
```
Et si l'on tente à nouveau de se connecter au serveur web, ce coup ci la page index.html est bien chargée !\
On aurai pu également utiliser 'chcon' pour appliquer un contexte temporaire.\
A noter que le "restorecon" ci dessus" aurait effacé les modification temporaire que l'on aurai pu apporter avec "chcon".
\
Il est également possible d'utiliser sealert d'une manière différente en appliquant directement des modifications en se référant aux logs notamment avec les fonctions "ausearch" et "audit2allow"

### 3.5 Durcissement de la configuration de SELinux
Afin de durcir ma configuration de SElinux je m'appuie sur le CIS Rocky linux 9 Benchmark V2, notamment la partie faisant un focus sur SElinux :

![image](https://github.com/user-attachments/assets/4befd887-1a3d-4d50-bb50-f351f3e2d8fd)

On peux voir ci dessous, toutes les vérifiactions effecutées afin de durcir la configuration au mieux. J'ai uniquement laissé troubleshout disponible sur la machine afin de continuer de m'en servir.

```
[ngermond@localhost ~]$ sudo grubby --info=ALL | grep -Po '(selinux|enforcing)=0\b'
[ngermond@localhost ~]$ sudo grep -E '^\s*SELINUXTYPE=(targeted|mls)\b' /etc/selinux/config
SELINUXTYPE=targeted
[ngermond@localhost ~]$ sudo sestatus | grep Loaded
Loaded policy name:             targeted
[ngermond@localhost ~]$ sudo ps -eZ | grep unconfined_service_t
[ngermond@localhost ~]$ sudo rpm -q mcstrans
package mcstrans is not installed
[ngermond@localhost ~]$ sudo rpm -q setroubleshoot
setroubleshoot-3.3.32-1.el9.x86_64
```
Finalement, toujours grâce à un audit lynis, l'Hardening index de ma machine trouvera un score de 74 :
```
  Lynis security scan details:

  Hardening index : 74 [##############      ]
  Tests performed : 268
  Plugins enabled : 2
```

