# Gestion de Parc Linux

---

## Mission

Trouver une solution analogue à **Microsoft Intune** pour les postes de formation sous Linux :

| Domaine | Description |
|---|---|
| Déploiement | Installation automatisée d'OS à distance |
| Configuration | Paramétrage uniforme des postes |
| Administration | Gestion centralisée du parc |
| Supervision | Monitoring et état des machines |
| Sécurisation | Durcissement et conformité |

---

## Contexte

Les solutions documentées ici **ne sont pas adaptées à tous les usages**. Le contexte dans lequel ce travail a été réalisé est spécifique :

```
Entreprise
└── Centaines de LAN
    ├── Ville A
    ├── Ville B
    ├── Ville C
    └── ...
```

> **Contrainte principale** : éviter un serveur local dans chaque LAN — trop complexe à déployer et à maintenir à grande échelle.

La solution retenue repose sur une **architecture centralisée avec un serveur distant**, apportant :

- Flexibilité de gestion
- Maintenance simplifiée
- Déploiement uniforme sur tous les sites
- Pas d'infrastructure locale à maintenir

---

## Documentation

À lire dans l'ordre :

| Document | Description |
|---|---|
| [Déployer des postes Ubuntu/Debian avec serveur local](deployer_poste_linux.md) | Configuration du serveur PXE/iPXE pour déployer automatiquement des postes Ubuntu/Debian à distance |
| [Déployer Puppet automatiquement sur les postes](configuration_puppet8.md) | Faire en sorte que les PC clients soient enregistrées automatiquement auprès du serveur Puppet afin de reçevoir une configuration |

### Déployer des services via Puppet

| Document | Description |
|---|---|
| [Déployer un portail d'application](deployer_portailapplication_via_puppet.md) | Déployer un portail d'application permettant aux utilisateurs d'installer une liste d'applications |
| [Déployer Zabbix automatiquement pour la supervision](deployer_zabbix_via_puppet.md) | Faire en sorte que les postes soient supervisés automatiquement via Zabbix |
| [Déployer GLPI automatiquement pour l'inventaire](deployer_glpi.md) | Déployer GLPI Agent automatiquement sur les postes afin d'avoir un inventaire des postes sur le serveur GLPI | 
| [Déployer MeshCentral automatiquement](deployer_meshcentral.md) | Faire en sorte que les postes soient automatiquement supervisés auprès du serveur MeshCentral afin de pouvoir prendre la main à distance sur les postes |
