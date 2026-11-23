# Infrastructure réseau d’entreprise — Cisco & MikroTik

**Projet de laboratoire | Administration réseau, haute disponibilité et sécurité périmétrique**

## 1. Présentation et objectifs

Ce projet simule une infrastructure réseau d’entreprise répartie sur **trois étages**, combinant des commutateurs Cisco de distribution et d’accès avec un routeur **MikroTik RouterOS** assurant la sortie vers Internet. L’architecture a été conçue pour mettre en pratique la **segmentation logique du réseau**, la **redondance des liaisons**, le **routage**, l’**attribution automatique d’adresses IP** et le **filtrage du trafic**.

**Technologies mises en œuvre :** VLAN 802.1Q, trunks, RSTP, redondance de passerelle HSRP (selon la configuration du projet), DHCP, NAT masquerade, pare-feu stateful RouterOS et administration WinBox/SSH.

## 2. Topologie et équipements

| Équipement | Fonction |
|---|---|
| **SW1DISTMAIN** | Commutateur multilayer principal de distribution |
| **SW2DISTBACKUP** | Commutateur multilayer de secours ; RSTP vérifié dans les captures |
| **SW1ACCESS / SW2ACCESS / SWACCESS3** | Accès utilisateurs aux étages 1, 2 et 3 |
| **MikroTik CHR / RouterOS 6.46.4** | Routage inter-VLAN, DHCP, NAT, filtrage et accès WAN dans les captures fournies |
| **VPCS** | Postes de test répartis dans les VLAN métiers |

**Connectivité générale :** deux commutateurs de distribution sont interconnectés aux commutateurs d’accès par des liens redondants. Le MikroTik est relié à l’infrastructure de distribution et au réseau amont. Le protocole Spanning Tree empêche la formation de boucles de couche 2.

## 3. Segmentation et plan d’adressage

### VLAN et sous-réseaux réellement configurés / visibles

| VLAN | Nom | Réseau IP visible sur MikroTik | Passerelle DHCP annoncée | Usage |
|---|---|---|---|---|
| **10** | HR | `192.168.10.0/24` | `192.168.10.1` | Ressources humaines |
| **20** | IT | `192.168.20.0/24` | `192.168.20.1` | Informatique |
| **99** | MGT-EQUIP | `192.168.99.0/24` | À confirmer | Gestion des équipements |
| **—** | WAN / amont | `172.16.100.0/24` | `172.16.100.2` (route par défaut) | Liaison vers Internet / réseau amont |

La capture RouterOS montre des interfaces `vlan10-rh`, `vlan20-it` et `vlan99-mgmt`, ainsi qu’une route par défaut via `172.16.100.2`. Des baux DHCP sont visibles, notamment `192.168.10.200` et `192.168.20.197–200`.

**Attention à l’adressage du schéma :** le dessin annote les étages comme `192.168.1.0/24`, `192.168.2.0/24` et `192.168.3.0/24`. Ces libellés ne correspondent **pas** aux sous-réseaux des VLAN 10, 20 et 99 affichés dans RouterOS. Dans cette documentation, les réseaux `192.168.10.0/24`, `192.168.20.0/24` et `192.168.99.0/24` ont été retenus comme références de configuration. Le schéma doit être mis à jour pour éviter toute confusion : un VLAN peut regrouper des équipements situés à plusieurs étages.

## 4. Commutation, trunks et haute disponibilité

- **802.1Q :** les liaisons montantes transportent les VLAN `10,20,99` ; VLAN natif observé : `1`.
- **SW1ACCESS :** trunks `Et0/3` et `Et1/0` en état `trunking` ; VLAN 10 et 20 actifs et transférés. VLAN 99 autorisé mais **absent de la liste des VLAN actifs** sur ces trunks dans la capture.
- **SW2DISTBACKUP :** trunks `Et1/0`, `Et1/1` et `Et1/2` ; VLAN 10, 20 et 99 autorisés et actifs. Pour les VLAN 10/20, `Et1/0` est en forwarding, tandis que `Et1/1` et `Et1/2` ne transfèrent pas ces VLAN selon les états visibles.
- **RSTP / VLAN 10 :** sur SW2DISTBACKUP, `Et1/0` est **Root / FWD** et `Et1/1`, `Et1/2` sont **Alternate / BLK**. Le root bridge indiqué a la priorité `4106` et l’adresse MAC `aabb.cc00.0100` ; SW2DISTBACKUP a la priorité `8202` et l’adresse `aabb.cc00.0200`.
- **HSRP :** redondance de passerelle déclarée dans le périmètre du projet, mais **non démontrée par les captures jointes**. À compléter avec `show standby brief` sur les deux switches multilayer si HSRP assure effectivement les passerelles des VLAN.

## 5. Services IP et sécurité MikroTik

**DHCP.** Trois serveurs DHCP sont visibles, affectés aux interfaces `vlan10-rh`, `vlan20-it` et `vlan99-mgmt`, avec une durée de bail de **8 heures**. Les réseaux DHCP 10 et 20 annoncent les passerelles `.1` et les DNS `1.1.1.1` / `8.8.8.8`.

**Routage et NAT.** Le routeur dispose de routes directement connectées vers le WAN et les trois VLAN, d’une route par défaut via `172.16.100.2` et d’une règle NAT **`srcnat` / `masquerade`** sortant sur `ether1`.

**Pare-feu RouterOS — chaîne `input` (trafic destiné au routeur) :**

1. Autorisation des connexions `established,related` ; blocage des connexions `invalid`.
2. Autorisation ICMP pour le diagnostic.
3. Règles DHCP UDP `67 → 68` sur les interfaces VLAN 10, 20 et 99.
4. Règle d’accès WinBox TCP `8291` depuis `172.16.100.1` sur `ether1`.
5. Règle de blocage du trafic entrant depuis le WAN vers le routeur (`in-interface=ether1`).

**Services d’administration :** SSH (`22`) et WinBox (`8291`) apparaissent activés ; Telnet, FTP, HTTP, HTTPS et API non chiffrée apparaissent désactivés dans la capture. La restriction WinBox à l’adresse `172.16.100.1/32` est visible dans la configuration du service. Pour une mise en production, contrôler également les règles `forward`, les politiques d’administration et l’accès SSH.

## 6. Vérifications et points à finaliser

**Éléments observables :** trunks actifs, rôles RSTP, VLAN déclarés, baux DHCP, routes RouterOS et présence d’une règle NAT masquerade. Ces captures prouvent la présence de configurations et d’états opérationnels locaux, **pas** à elles seules un test complet de bout en bout ni un basculement de haute disponibilité réussi.

**Tests recommandés pour compléter le dépôt :**

- `show vlan brief`, `show interfaces trunk`, `show spanning-tree vlan 10`, `show spanning-tree vlan 20`, `show standby brief` sur les Cisco ;
- `/ip address print`, `/ip route print`, `/ip dhcp-server lease print`, `/ip firewall filter print`, `/ip firewall nat print` sur MikroTik ;
- Tests DHCP, ping de passerelle, routage inter-VLAN, accès Internet, filtrage inter-VLAN et basculement d’un lien ;
- Vérifier l’affectation réelle du routage inter-VLAN : les interfaces VLAN et les passerelles DHCP apparaissent sur MikroTik, tandis que HSRP est annoncé sur les switches multilayer ;
- Corriger la divergence entre les adresses du schéma et celles de la configuration, puis vérifier la propagation du VLAN 99 sur SW1ACCESS.

---

**Environnement :** laboratoire d’apprentissage et de validation technique — Cisco IOS (switching) et MikroTik CHR sous RouterOS, administré via WinBox.  
**Note de sécurité :** retirer les identifiants, mots de passe, secrets, IP publiques réelles et exports de configuration sensibles avant publication sur GitHub.
