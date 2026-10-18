# Conception et sécurisation d’une infrastructure réseau d’entreprise

> Projet de laboratoire réalisé avec **Cisco Packet Tracer** — segmentation réseau, routage inter-VLAN, filtrage ACL et sécurisation de l’administration des équipements.

## 1. Présentation

Ce projet consiste à concevoir et configurer une petite infrastructure réseau d’entreprise organisée en trois VLAN : administration, utilisateurs et serveurs. L’objectif est de séparer logiquement les groupes, de contrôler les communications autorisées et de limiter l’accès d’administration aux équipements réseau.

Le laboratoire met en pratique plusieurs notions de réseaux et de sécurité : VLAN 802.1Q, routage inter-VLAN (« router-on-a-stick »), listes de contrôle d’accès (ACL), SSH version 2, sécurité des ports du commutateur et désactivation des interfaces inutilisées.

**Environnement :** Cisco Packet Tracer.

## 2. Objectifs

- Segmenter le réseau en VLAN selon les rôles.
- Assurer le routage inter-VLAN via un routeur Cisco.
- Limiter les communications entre utilisateurs, réseau d’administration et serveur.
- Utiliser SSH v2 pour l’administration distante de R1 et S1.
- Restreindre l’accès SSH à S1 au sous-réseau d’administration.
- Activer Port Security sur les ports d’accès et apprendre les adresses MAC en mode sticky.
- Désactiver les ports du commutateur qui ne sont pas utilisés.
- Documenter la configuration et les résultats de tests observés.

## 3. Topologie logique

```text
                       +---------------------------+
                       | R1 — Routeur              |
                       | Routage inter-VLAN        |
                       | ACL et SSH v2             |
                       +-------------+-------------+
                                     |
                         Trunk 802.1Q | Gi0/0 — Fa0/1
                                     |
                       +-------------+-------------+
                       | S1 — Commutateur           |
                       | VLAN, Port Security, SSH   |
                       +------+------+-------------+
                              |      |      |
                    Fa0/2     |      |      | Fa0/5
                    VLAN 10   |      |      | VLAN 30
                              |      |
                     +--------+  +--+---------+  +----------------+
                     | PC ADMIN|  | PC USER1   |  | SERVER         |
                     | .10.10  |  | .20.10     |  | .30.10         |
                     +---------+  +------------+  +----------------+
                                      |
                                Fa0/4 | VLAN 20
                                +-----+------+
                                | PC USER2   |
                                | .20.11     |
                                +------------+
```

Le lien entre R1 et S1 est configuré en trunk. Le routage entre les VLAN est assuré par des sous-interfaces sur R1.

## 4. Plan d’adressage

| Élément | VLAN | Adresse IP | Masque | Passerelle |
|---|---:|---|---|---|
| R1 — sous-interface G0/0.1 | 10 | `192.168.10.1` | `/24` | — |
| S1 — interface VLAN 10 (gestion) | 10 | `192.168.10.2` | `/24` | `192.168.10.1` |
| PC ADMIN | 10 | `192.168.10.10` | `/24` | `192.168.10.1` |
| R1 — sous-interface G0/0.2 | 20 | `192.168.20.1` | `/24` | — |
| PC USER1 | 20 | `192.168.20.10` | `/24` | `192.168.20.1` |
| PC USER2 | 20 | `192.168.20.11` | `/24` | `192.168.20.1` |
| R1 — sous-interface G0/0.3 | 30 | `192.168.30.1` | `/24` | — |
| SERVER | 30 | `192.168.30.10` | `/24` | `192.168.30.1` |

Masque des réseaux : `255.255.255.0`.

## 5. VLAN et ports du commutateur

| VLAN | Nom | Ports d’accès | Rôle |
|---:|---|---|---|
| 10 | ADMIN | Fa0/2 | Poste d’administration |
| 20 | USERS | Fa0/3, Fa0/4 | Postes utilisateurs |
| 30 | SERVERS | Fa0/5 | Serveur Web |
| — | Trunk | Fa0/1 | Liaison vers R1 |

Les interfaces Fa0/6 à Fa0/24 et Gi0/1 à Gi0/2 ont été vérifiées comme étant administrativement désactivées. Le SVI VLAN 10 de S1 était `up/up` avec l’adresse `192.168.10.2`.

## 6. Mesures de sécurité configurées

### 6.1 Segmentation et routage

- VLAN 10 pour l’administration.
- VLAN 20 pour les utilisateurs.
- VLAN 30 pour les serveurs.
- Routage inter-VLAN via les sous-interfaces de R1 et le trunk 802.1Q.

### 6.2 ACL sur R1

Une ACL étendue nommée `USERS_TO_ADMIN` est appliquée en entrée sur `GigabitEthernet0/0.2`, interface de passerelle du VLAN USERS.

Politique configurée :

| Ordre | Source | Destination | Service | Action |
|---:|---|---|---|---|
| 10 | `192.168.20.0/24` | `192.168.10.0/24` | Tout IP | Refuser |
| 20 | `192.168.20.0/24` | `192.168.30.10` | TCP/22 (SSH) | Refuser |
| 30 | `192.168.20.0/24` | `192.168.30.10` | TCP/80 (HTTP) | Autoriser |
| 40 | `192.168.20.0/24` | `192.168.30.10` | TCP/443 (HTTPS) | Autoriser |
| 50 | `192.168.20.0/24` | Toute destination IP | Tout IP | Refuser |
| 60 | Autres paquets atteignant cette ACL et non appariés plus tôt | — | Tout IP | Autoriser |

L’ACL est évaluée de haut en bas : la première règle correspondante s’applique. La règle 60 ne neutralise pas les refus précédents. Le refus final de la ligne 50 signifie notamment que les autres communications IP initiées depuis le VLAN USERS ne sont pas autorisées par cette ACL.

### 6.3 Administration distante SSH

- SSH version 2 est activé sur R1 et S1.
- L’administration de S1 utilise un compte local et les lignes VTY sont configurées avec `login local` et `transport input ssh`.
- Une ACL standard `SSH_ADMIN_ONLY` est appliquée en entrée sur les lignes VTY 0–4 et 5–15 de S1.
- Cette ACL autorise `192.168.10.0/24` et refuse les autres adresses sources.
- Un compteur de six correspondances a été observé sur la règle `permit` lors de la dernière vérification. Ce compteur ne constitue pas à lui seul une preuve complète de tous les scénarios de connexion.

### 6.4 Sécurité des ports (Port Security)

Sur les ports d’accès Fa0/2 à Fa0/5 :

- mode access ;
- Port Security activé ;
- maximum d’une adresse MAC sécurisée par port ;
- apprentissage sticky des adresses MAC ;
- mode de violation `restrict`.

Les sorties de vérification ont montré une adresse MAC sticky apprise sur chacun des quatre ports et zéro violation au moment de la vérification.

### 6.5 Interfaces inutilisées

Les ports Fa0/6 à Fa0/24 et Gi0/1 à Gi0/2 sont désactivés administrativement (`shutdown`). Les ports utilisés pour les postes, le serveur et le trunk restent actifs.

## 7. Tests effectués et résultats observés

Ce tableau distingue les observations faites dans Packet Tracer des tests qui restent à confirmer. Les résultats décrivent uniquement les essais rapportés pendant le laboratoire.

| Test | Résultat observé | Interprétation |
|---|---|---|
| PC ADMIN → `192.168.10.1` (ping) | 4 réponses sur 4, 0 % de perte | Connectivité vers la passerelle ADMIN confirmée |
| PC ADMIN → R1 via SSH | Session ouverte et invite `R1#` obtenue | SSH vers R1 fonctionnel |
| PC ADMIN → S1 via SSH | Une session a atteint l’invite `S1#` | SSH et authentification vers S1 fonctionnels lors de cet essai |
| PC USER1 → `192.168.10.10` (ping) | Réponses `Destination host unreachable` émises par `192.168.20.1` | Cohérent avec le refus de l’ACL vers le VLAN ADMIN |
| PC USER1 → `192.168.10.30` (ping) | Réponses `Destination host unreachable` émises par `192.168.20.1` | Adresse testée ; le résultat montre l’échec de connectivité, pas un test vers le PC ADMIN configuré (`192.168.10.10`) |
| PC USER1 → serveur Web | Le navigateur s’ouvre et une page du serveur Packet Tracer est affichée | Accès Web observé ; HTTP et HTTPS séparément doivent être consignés si testés indépendamment |
| PC USER1 → S1 via SSH | `Connection timed out; remote host not responding` | Échec de connexion observé, cohérent avec la politique ; confirmer la cause par un test contrôlé si nécessaire |
| Port Security sur Fa0/2–Fa0/5 | Une MAC sticky par port ; 0 violation affichée | Apprentissage MAC et état des ports vérifiés |
| Ports inutilisés de S1 | Interfaces administrativement down | Désactivation vérifiée |

### Tests complémentaires recommandés

- Tester séparément `http://192.168.30.10` et `https://192.168.30.10` depuis PC USER1 et noter chaque résultat.
- Vérifier que l’ACL `USERS_TO_ADMIN` est appliquée en entrée sur `GigabitEthernet0/0.2`.
- Après une tentative SSH depuis USER1, relever `show access-lists` sur S1 et confirmer que la règle `deny any` est bien responsable du refus.
- Tester les autres communications interdites/permises selon la matrice ACL, sans modifier les règles avant d’avoir enregistré les résultats actuels.

## 8. Commandes utiles de vérification

Sur R1 :

```text
show ip interface brief
show access-lists
show ip interface GigabitEthernet0/0.2
show ip ssh
show running-config
```

Sur S1 :

```text
show vlan brief
show ip interface brief
show access-lists
show port-security
show port-security address
show ip ssh
show running-config
```

## 9. Structure suggérée du dépôt GitHub

```text
reseau-entreprise-securise/
├── README.md
├── topology/
│   └── topology.png
├── packet-tracer/
│   └── reseau-entreprise-securise.pkt
├── configs/
│   ├── R1.txt
│   └── S1.txt
└── screenshots/
    ├── vlan-brief.png
    ├── acl-r1.png
    ├── ssh-admin-s1.png
    ├── port-security.png
    └── tests-connectivite.png
```

Ajouter uniquement les fichiers réellement créés. Remplacer les noms génériques par les noms exacts des fichiers du laboratoire.

## 10. Limites et bonnes pratiques

- Ce laboratoire a été réalisé dans Cisco Packet Tracer ; il ne remplace pas une validation sur équipements physiques ou dans un environnement de production.
- Les résultats de test doivent être accompagnés de captures d’écran ou de sorties de commandes lorsqu’ils sont présentés dans un portfolio.
- Ne jamais publier de mots de passe, secrets, clés privées ou fichiers contenant des identifiants réutilisables. Remplacer les identifiants de laboratoire avant de partager les configurations.
- Ne pas présenter des mécanismes non implémentés et testés (par exemple pare-feu dédié, IDS/IPS ou SIEM) comme faisant partie de la version actuelle.

## 11. Compétences mises en pratique

- Commutation et segmentation VLAN.
- Trunk 802.1Q et routage inter-VLAN.
- ACL IPv4 et filtrage des flux.
- Administration distante SSH v2.
- Contrôle d’accès aux lignes VTY.
- Port Security et apprentissage d’adresses MAC.
- Durcissement de base d’un commutateur.
- Vérification de configuration et documentation de tests.

---

**Statut :** laboratoire en cours de documentation. Les éléments configurés sont décrits ci-dessus ; les tests complémentaires indiqués restent à effectuer ou à consigner séparément.

**Auteur :** Yacoub Ahmed  
**Date :** 18-10-2026
