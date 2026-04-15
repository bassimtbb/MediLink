# MediLink - Workflow de Conception 

> **Projet :** MediLink — (TO DO)

---

## Table des matières

1. [Étape 1 — Liste brute des fonctionnalités](#étape-1--liste-brute-des-fonctionnalités)
2. [Étape 2 — Regroupement par domaine métier (DDD)](#étape-2--regroupement-par-domaine-métier-ddd)
3. [Étape 3 — Entités métier par module](#étape-3--entités-métier-par-module)


---

## Étape 1 — Liste brute des fonctionnalités

> **Méthode :** Liste exhaustive, non triée. Mode « fonctionnel pur » — *ce que* fait l'application, pas *comment* elle est construite. Trois perspectives : Patient, Médecin , et Administrateur système.

### 👤 Patient

- Créer un compte
- Se connecter
- Modifier son profil 
- Rechercher un médecin par spécialité, ville ou nom
- Consulter les créneaux disponibles
- Prendre rendez-vous
- Annuler ou déplacer un rendez-vous
- Recevoir des rappels par notification ou email
- Consulter l’historique des rendez-vous
- Déposer des documents médicaux 
- Consulter des ordonnances ou comptes rendus 
- Laisser un avis sur le praticien. 


### 🏪 Médecin

- Créer un compte professionnel
- Renseigner sa spécialité 
- Définir ses horaires de consultation
- Ouvrir ou fermer des créneaux
- Consulter son agenda 
- Accepter ou refuser certaines demandes 
- Consulter le dossier administratif du patient
- Déposer une ordonnance
- Déposer un compte rendu
- Suivre l’historique des rendez-vous. 

### 🛡️ Perspective Administrateur Système

- Gérer les comptes utilisateurs, 
- Vérifier les comptes médecins, 
- Modérer les avis, 
- Superviser la plateforme, 
- Gérer les catégories de spécialités médicales, 
- Consulter des statistiques globales. 

---

## Étape 2 — Regroupement par domaine métier (DDD)

> **Principe appliqué :** Forte cohésion au sein de chaque module (toutes les fonctionnalités partagent la même raison métier d'évoluer), faible couplage entre les modules (chaque module expose des interfaces propres et ne dépend pas des détails internes d'un autre).

Six **Contextes Bornés** (Bounded Contexts) émergent naturellement :

| # | Module / Contexte Borné | Responsabilité principale | Raison clé de la séparation |
|---|---|---|---|
| 1 | **Identity & Access** | Qui êtes-vous ? Authentification, création de comptes (patient/médecin) et validation des diplômes | La sécurité et le processus de vérification des praticiens sont critiques et isolés |
| 2 | **Medical Directory** | Qu'est-ce qui est disponible ? Recherche, spécialités et profils publics des médecins | La logique de recherche (filtres, villes) évolue différemment de la prise de rendez-vous |
| 3 | **Appointment** | Quand se voit-on ? Gestion des créneaux, réservations, annulations et reports | C'est le cœur du métier avec des règles de collision et de disponibilité complexes |
| 4 | **Health Record** | Quels sont les faits ? Stockage sécurisé des ordonnances, comptes rendus et documents patients | La gestion des fichiers (PDF) et la confidentialité médicale exigent une infrastructure spécifique |
| 5 | **Notification** | Comment informer ? Rappels automatiques, alertes de report et emails de confirmation | Ce module est souvent asynchrone (Event-Driven) pour ne pas bloquer le reste de l'app |
| 6 | **Administration** | Comment va la plateforme ? Modération des avis et statistiques de fréquentation | Les outils d'analyse et de modération sont destinés aux administrateurs, pas aux utilisateurs |

### Carte des dépendances entre modules

```
                  ┌───────────────────────┐
                  │   Identity & Access   │
                  └───────────┬───────────┘
                              │ (Token JWT / Rôle : Patient, Médecin, Admin)
          ┌───────────────────┼───────────────────┐
          ▼                   ▼                   ▼
  ┌───────────────┐   ┌───────────────┐   ┌──────────────┐
  │   Directory   │   │  Appointment  │   │  Governance  │
  └───────┬───────┘   └───────┬───────┘   └──────────────┘
          │                   │
          │ (ID Praticien)    │ (Événements : RdvCréé, RdvAnnulé)
          └─────────┬─────────┘
                    ▼
          ┌───────────────────┐
          │   Health Record   │
          └─────────┬─────────┘
                    │
                    │ (Événements domaine : RdvConfirmé,
                    │  OrdonnanceDéposée, RappelProche...)
                    ▼
          ┌───────────────────┐
          │   Notification    │
          └───────────────────┘
```

> **Faible couplage garanti :(TO DO) .
---

## Étape 3 — Entités métier par module

> **Vocabulaire :**
> - **Entité** — Possède une identité unique, est mutable dans le temps
> - **Objet Valeur (Value Object)** — Pas d'identité, immuable, défini entièrement par ses valeurs
> - **Racine d'Agrégat** — Point d'entrée unique d'un groupe d'entités liées
(TO DO)

*Document produit dans le cadre du projet MediLink*  

