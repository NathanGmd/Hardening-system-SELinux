# Hardening-system-SELinux

Dans le cadre du renforcement de la sécurité du serveur, l’authentification SSH a été configurée avec un jeu de clés Ed25519, garantissant de meilleures performances et une sécurité accrue par rapport aux clés RSA traditionnelles.

L’accès SSH est désormais restreint aux connexions par clé, avec l’authentification par mot de passe désactivée. Par mesure de sécurité supplémentaire, seul le port 22/TCP a été ouvert dans le pare-feu, limitant ainsi l’exposition du service SSH aux connexions non autorisées.

Cette configuration permet de sécuriser efficacement l’accès au serveur tout en maintenant un niveau de flexibilité optimal pour l’administration distante.
