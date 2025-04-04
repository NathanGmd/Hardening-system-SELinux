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
