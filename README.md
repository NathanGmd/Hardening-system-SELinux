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


