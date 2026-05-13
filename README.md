# Gestion de Parc Linux — Réseau Canopé

> Documentation technique rédigée lors d'un stage au sein de **Réseau Canopé**.

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
Réseau Canopé
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

| Document | Description |
|---|---|
| [Déployer des postes Ubuntu/Debian avec serveur local](deployer_poste_linux.md) | Configuration du serveur PXE/iPXE local pour déployer automatiquement des postes Ubuntu/Debian à distance |
