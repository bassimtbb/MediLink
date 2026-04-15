# MediLink — Workflow de Conception

> **Projet :** MediLink — Application de mise en relation entre patients et médecins pour la prise de rendez-vous en ligne, la gestion des créneaux et le partage sécurisé de documents médicaux.

---

## Table des matières

1. [Étape 1 — Liste brute des fonctionnalités](#étape-1--liste-brute-des-fonctionnalités)
2. [Étape 2 — Regroupement par domaine métier (DDD)](#étape-2--regroupement-par-domaine-métier-ddd)
3. [Étape 3 — Entités métier par module](#étape-3--entités-métier-par-module)
4. [Résumé de conception — Checklist en 4 questions](#résumé-de-conception--checklist-en-4-questions)

---

## Étape 1 — Liste brute des fonctionnalités

> **Méthode :** Liste exhaustive, non triée. Mode « fonctionnel pur » — *ce que* fait l'application, pas *comment* elle est construite. Trois perspectives : Patient, Médecin, et Administrateur système.

### 👤 Patient

- Créer un compte
- Se connecter / Se déconnecter
- Modifier son profil (nom, prénom, date de naissance, téléphone)
- Rechercher un médecin par spécialité, ville ou nom
- Consulter les créneaux disponibles d'un médecin
- Prendre un rendez-vous
- Annuler ou déplacer un rendez-vous
- Recevoir des rappels par notification ou email
- Consulter l'historique des rendez-vous
- Déposer des documents médicaux
- Consulter des ordonnances ou comptes rendus
- Laisser un avis sur le praticien

### 🩺 Médecin

- Créer un compte professionnel (avec numéro RPPS)
- Renseigner sa spécialité
- Définir ses horaires de consultation
- Ouvrir ou fermer des créneaux
- Consulter son agenda du jour
- Accepter ou refuser certaines demandes de rendez-vous
- Consulter le dossier administratif du patient (lors du rendez-vous)
- Déposer une ordonnance
- Déposer un compte rendu
- Suivre l'historique de ses rendez-vous
- Répondre aux avis patients

### 🛡️ Administrateur Système

- Gérer les comptes utilisateurs (patients + médecins)
- Vérifier et approuver les comptes médecins (validation RPPS)
- Suspendre ou bannir un compte
- Modérer les avis
- Superviser la plateforme (statistiques globales)
- Gérer les catégories de spécialités médicales

---

## Étape 2 — Regroupement par domaine métier (DDD)

> **Principe appliqué :** Forte cohésion au sein de chaque module (toutes les fonctionnalités partagent la même raison métier d'évoluer), faible couplage entre les modules (chaque module expose des interfaces propres et ne dépend pas des détails internes d'un autre).

Six **Contextes Bornés** (Bounded Contexts) émergent naturellement :

| # | Module / Contexte Borné | Responsabilité principale | Raison clé de la séparation |
|---|---|---|---|
| 1 | **Identity & Access** | Qui êtes-vous ? Authentification, création de comptes (patient/médecin) et validation des diplômes | La sécurité et le processus de vérification des praticiens sont critiques et isolés |
| 2 | **Medical Directory** | Qu'est-ce qui est disponible ? Recherche et profils publics des médecins (découvrabilité uniquement) | La logique de recherche évolue différemment de la prise de rendez-vous ; ce module ne gère pas les créneaux |
| 3 | **Appointment** | Quand se voit-on ? Gestion des créneaux, réservations, annulations et reports | C'est le cœur du métier avec des règles de collision et de disponibilité complexes |
| 4 | **Health Record** | Quels sont les faits médicaux ? Stockage sécurisé des ordonnances, comptes rendus et documents patients | La gestion des fichiers et la confidentialité médicale (RGPD, secret médical) exigent une infrastructure isolée |
| 5 | **Notification** | Comment informer ? Rappels automatiques, alertes de report et emails de confirmation | Ce module est asynchrone (Event-Driven) et réagit à des événements d'autres modules sans être couplé à eux |
| 6 | **Administration** | Comment va la plateforme ? Modération des avis, statistiques et gouvernance | Les outils d'analyse et de modération sont destinés aux administrateurs, pas aux utilisateurs finaux |

---

### Carte des dépendances entre modules

```
                  ┌───────────────────────┐
                  │   Identity & Access   │
                  └───────────┬───────────┘
                              │ (Token JWT / Rôle : Patient, Médecin, Admin)
          ┌───────────────────┼──────────────────┐
          ▼                   ▼                  ▼
  ┌───────────────┐   ┌───────────────┐  ┌──────────────────┐
  │   Directory   │   │  Appointment  │  │  Administration  │
  └───────┬───────┘   └───────┬───────┘  └──────────────────┘
          │                   │
          │ (ID Médecin)      │ (RdvId → DossierMédicalId)
          └─────────┬─────────┘
                    ▼
          ┌───────────────────┐
          │   Health Record   │
          └───────────────────┘
                    
          ┌──────────────────────────────────────────────────┐
          │  Événements domaine publiés (bus d'événements) : │
          │  RdvConfirmé, RdvAnnulé (Appointment)            │
          │  OrdonnanceDéposée (Health Record)               │
          │  CompteApprouvé (Identity)                       │
          │  AvisModéré (Administration)                     │
          └──────────────────┬───────────────────────────────┘
                             ▼
                   ┌───────────────────┐
                   │   Notification    │
                   └───────────────────┘
```



> **Faible couplage garanti :** Les modules de MediLink ne s'importent jamais directement. Ils communiquent via deux mécanismes :
> - Des **événements domaine** publiés sur un bus (ex. : `Appointment` publie `RdvConfirmé`, `Notification` écoute) ;
> - Des **interfaces exposées** (APIs internes) quand un module a besoin d'une donnée d'un autre. Chaque module reste maître de ses propres données.

---

## Étape 3 — Entités métier par module

> **Vocabulaire :**
> - **Entité** — Possède une identité unique (`id`), est mutable dans le temps 
> - **Objet Valeur (Value Object)** — Pas d'identité propre, immuable, défini entièrement par ses valeurs 
> - **Racine d'Agrégat** — Le "chef" d'un groupe d'entités liées ; les autres modules n'interagissent qu'avec lui via son `id`, jamais directement avec ses entités enfants

---

### Module 1 — `Identity & Access`

**Rôle :** Gérer qui peut se connecter et avec quels droits. C'est le gardien de la plateforme.

#### Racine d'Agrégat : `User`

| Attribut | Type | Pourquoi il existe |
|---|---|---|
| `id` | UUID | Identifiant unique généré par le système |
| `email` | String | Identifiant de connexion |
| `motDePasseHash` | String | Sécurité (jamais stocké en clair) |
| `role` | Enum : `PATIENT, MEDECIN, ADMIN` | Contrôle des droits d'accès |
| `statut` | Enum : `EN_ATTENTE, ACTIF, SUSPENDU` | Cycle de vie du compte |
| `createdAt` | DateTime | Traçabilité |

#### Entité : `ProfilPatient` *(appartient à User)*

| Attribut | Type | Pourquoi il existe |
|---|---|---|
| `userId` | UUID | Lien vers le `User` parent |
| `prénom` | String | Identification civile |
| `nom` | String | Identification civile |
| `dateNaissance` | LocalDate | Informations médicales de base |
| `téléphone` | String | Contact et notifications SMS |
| `adresse` | Adresse | Objet Valeur : lieu de résidence |

#### Entité : `ProfilMédecin` *(appartient à User)*

| Attribut | Type | Pourquoi il existe |
|---|---|---|
| `userId` | UUID | Lien vers le `User` parent |
| `numRPPS` | String | Numéro officiel de praticien (vérification légale) |
| `spécialitéLabel` | String | Label de spécialité (simple, sans FK vers Directory) |
| `statutVérification` | Enum : `EN_ATTENTE, APPROUVÉ, REJETÉ` | Processus de validation admin |


#### Objet Valeur : `Adresse`

| Attribut | Type |
|---|---|
| `rue` | String |
| `ville` | String |
| `codePostal` | String |
| `pays` | String |


#### Diagramme de Classes — Identity & Access

```mermaid
classDiagram
    class RoleUtilisateur {
        <<enumeration>>
        PATIENT
        MEDECIN
        ADMIN
    }

    class StatutCompte {
        <<enumeration>>
        EN_ATTENTE
        ACTIF
        SUSPENDU
    }

    class StatutVerification {
        <<enumeration>>
        EN_ATTENTE
        APPROUVE
        REJETE
    }

    class User {
        <<Racine d Agregat>>
        +UUID id
        +String email
        +String motDePasseHash
        +RoleUtilisateur role
        +StatutCompte statut
        +DateTime createdAt
        +verifierIdentifiants(pwd) Boolean
        +activer() void
        +suspendre(raison) void
    }

    class ProfilPatient {
        <<Entite>>
        +UUID userId
        +String prenom
        +String nom
        +LocalDate dateNaissance
        +String telephone
        +Adresse adresse
        +mettreAJourContact(tel) void
        +getNomComplet() String
    }

    class ProfilMedecin {
        <<Entite>>
        +UUID userId
        +String numRPPS
        +String specialiteLabel
        +StatutVerification statutVerif
        +approuverVerification() void
        +rejeterVerification(raison) void
        +estVerifie() Boolean
    }

    class Adresse {
        <<Objet Valeur>>
        +String rue
        +String ville
        +String codePostal
        +String pays
    }

    User "1" *-- "0..1" ProfilPatient : possede
    User "1" *-- "0..1" ProfilMedecin : possede
    ProfilPatient *-- Adresse : reside a
    ProfilMedecin .. StatutVerification 
    User .. StatutCompte 
    User .. RoleUtilisateur 
```

---

### Module 2 — `Medical Directory`

**Rôle :** Le catalogue public des médecins. C'est ce que voit un patient quand il cherche un praticien. Ce module gère uniquement la **découvrabilité** — il ne gère pas les créneaux (c'est `Appointment`) et n'accède pas aux dossiers médicaux.

#### Racine d'Agrégat : `ProfilPublicMédecin`

| Attribut | Type | Pourquoi il existe |
|---|---|---|
| `id` | UUID | Identifiant unique |
| `médecinId` | UUID | Référence vers `Identity` (jamais les données complètes) |
| `nomAffichage` | String | Nom visible dans les résultats de recherche |
| `spécialité` | Spécialité | Objet Valeur décrivant la discipline médicale |
| `biographie` | String | Présentation libre du praticien |
| `adresseCabinet` | Adresse | Localisation pour la recherche géographique |
| `notesMoyenne` | Decimal | Dénormalisée, mise à jour via événement `AvisPublié` depuis Administration |
| `accepteNouveauxPatients` | Boolean | Fonctionnalité : ouvrir/fermer les réservations |

#### Objet Valeur : `Spécialité`

| Attribut | Type |
|---|---|
| `code` | String  |
| `libellé` | String |
| `icôneUrl` | String |


#### Objet Valeur : `Adresse` *(réutilisé depuis Identity & Access)*

Même structure que dans `Identity & Access`. Chaque module conserve sa propre copie — pas de partage de classe entre modules (faible couplage).

#### Diagramme de Classes — Medical Directory

```mermaid
classDiagram
    class ProfilPublicMedecin {
        <<Racine d Agregat>>
        +UUID id
        +UUID medecinId
        +String nomAffichage
        +Specialite specialite
        +String biographie
        +Adresse adresseCabinet
        +Decimal notesMoyenne
        +Boolean accepteNouveauxPatients
        +modifierBio(texte) void
        +mettreAJourNote(note) void
        +basculerDisponibilite() void
    }

    class Specialite {
        <<Objet Valeur>>
        +String code
        +String libelle
        +String iconeUrl
    }

    class Adresse {
        <<Objet Valeur>>
        +String rue
        +String ville
        +String codePostal
        +String pays
    }

    ProfilPublicMedecin *-- Specialite : possede
    ProfilPublicMedecin *-- Adresse : est situe a
```

---

### Module 3 — `Appointment`

**Rôle :** Le cœur du métier. Gérer le cycle de vie complet d'un rendez-vous (de la demande à la réalisation) et les créneaux de disponibilité des médecins. C'est ici que se trouvent les règles métier les plus complexes : anti-collision de créneaux, règles d'annulation, transitions d'état.

#### Racine d'Agrégat : `RendezVous`

| Attribut | Type | Pourquoi il existe |
|---|---|---|
| `id` | UUID | Identifiant unique |
| `patientId` | UUID | Référence vers `Identity` |
| `médecinId` | UUID | Référence vers `Identity` |
| `créneauId` | UUID | Référence vers `CréneauDisponible`  |
| `motif` | String | Raison de la consultation |
| `statut` | Enum : `DEMANDÉ, CONFIRMÉ, ANNULÉ, TERMINÉ` | Cycle de vie |
| `résumé` | RésuméRdv | Snapshot immuable capturé à la confirmation |
| `createdAt` | DateTime | Traçabilité |
| `confirméAt` | DateTime (nullable) | Horodatage de confirmation |
| `annuléAt` | DateTime (nullable) | Horodatage d'annulation |
| `raisonAnnulation` | String (nullable) | Traçabilité et statistiques |


#### Entité : `CréneauDisponible`

| Attribut | Type | Pourquoi il existe |
|---|---|---|
| `id` | UUID | Identifiant unique |
| `médecinId` | UUID | Lien vers le médecin propriétaire |
| `plage` | PlageHoraire | Objet Valeur : début + fin |
| `statut` | Enum : `LIBRE, RÉSERVÉ, BLOQUÉ` | Gestion de la disponibilité |


#### Objet Valeur : `RésuméRdv`

| Attribut | Type |
|---|---|
| `nomMédecin` | String |
| `spécialité` | String |
| `dateHeure` | LocalDateTime |
| `adresseCabinet` | String |

#### Objet Valeur : `PlageHoraire`

| Attribut | Type |
|---|---|
| `début` | LocalDateTime |
| `fin` | LocalDateTime |



#### Diagramme de Classes — Appointment

```mermaid
classDiagram
    class StatutRendezVous {
        <<enumeration>>
        DEMANDE
        CONFIRME
        ANNULE
        TERMINE
    }

    class StatutCreneau {
        <<enumeration>>
        LIBRE
        RESERVE
        BLOQUE
    }

    class RendezVous {
        <<Racine d Agregat>>
        +UUID id
        +UUID patientId
        +UUID medecinId
        +UUID creneauId
        +String motif
        +StatutRendezVous statut
        +RésuméRdv resume
        +DateTime createdAt
        +DateTime confirmeAt
        +DateTime annuleAt
        +String raisonAnnulation
        +confirmer() void
        +annuler(raison) void
    }

    class CreneauDisponible {
        <<Entite>>
        +UUID id
        +UUID medecinId
        +PlageHoraire plage
        +StatutCreneau statut
        +reserver() void
        +liberer() void
        +bloquer() void
    }

    class ResumeRdv {
        <<Objet Valeur>>
        +String nomMedecin
        +String specialite
        +LocalDateTime dateHeure
        +String adresseCabinet
    }

    class PlageHoraire {
        <<Objet Valeur>>
        +LocalDateTime debut
        +LocalDateTime fin
        +estValide() Boolean
        +chevauche(autre) Boolean
    }

    RendezVous "1" *-- "1" ResumeRdv : archive
    RendezVous --> CreneauDisponible : reserve
    CreneauDisponible *-- PlageHoraire : definit
    RendezVous .. StatutRendezVous 
    CreneauDisponible .. StatutCreneau 
```

---

### Module 4 — `Health Record`

**Rôle :** Le coffre-fort médical. Stocker de façon sécurisée les documents échangés entre patient et médecin. Ce module est régi par des contraintes de confidentialité (RGPD, secret médical) — c'est pourquoi il est rigoureusement isolé de tous les autres.

#### Racine d'Agrégat : `DossierMédical`

| Attribut | Type | Pourquoi il existe |
|---|---|---|
| `id` | UUID | Identifiant unique |
| `patientId` | UUID | Propriétaire du dossier (référence vers `Identity`) |
| `documents` | List\<Document\> | Collection de tous les fichiers du patient |

#### Entité : `Document` *(appartient à `DossierMédical`)*

| Attribut | Type | Pourquoi il existe |
|---|---|---|
| `id` | UUID | Identifiant unique |
| `dossierMédicalId` | UUID | Lien vers le dossier parent |
| `type` | Enum : `ORDONNANCE, COMPTE_RENDU, ANALYSE, AUTRE` | Catégorisation |
| `téléversePar` | UUID | ID du médecin ou patient auteur |
| `urlStockage` | String | Chemin sécurisé vers le fichier (S3, Azure Blob…) |
| `métadonnées` | MétadonnéesFichier | Objet Valeur : informations techniques du fichier |
| `statut` | Enum : `EN_ATTENTE, DISPONIBLE, ARCHIVÉ` | Cycle de vie |
| `createdAt` | DateTime | Traçabilité et tri chronologique |

#### Objet Valeur : `MétadonnéesFichier`

| Attribut | Type |
|---|---|
| `nomOriginal` | String |
| `tailleMo` | Decimal |
| `formatMime` | String |
| `checksum` | String  |



#### Diagramme de Classes — Health Record

```mermaid
classDiagram
    class TypeDocument {
        <<enumeration>>
        ORDONNANCE
        COMPTE_RENDU
        ANALYSE
        AUTRE
    }

    class StatutDocument {
        <<enumeration>>
        EN_ATTENTE
        DISPONIBLE
        ARCHIVE
    }

    class DossierMedical {
        <<Racine d Agregat>>
        +UUID id
        +UUID patientId
        +List~Document~ documents
        +ajouterDocument(doc) void
        +archiverDocument(docId) void
        +getHistorique() List~Document~
    }

    class Document {
        <<Entite>>
        +UUID id
        +UUID dossierMedicalId
        +TypeDocument type
        +UUID televersePar
        +String urlStockage
        +MétadonnéesFichier metadonnees
        +StatutDocument statut
        +DateTime creeAt
        +modifierStatut(nouveau) void
    }

    class MetadonneesFichier {
        <<Objet Valeur>>
        +String nomOriginal
        +Decimal tailleMo
        +String formatMime
        +String checksum
    }

    DossierMedical "1" *-- "0..*" Document : contient
    Document *-- MetadonneesFichier : decrit par
    Document ..> TypeDocument : utilise
    Document ..> StatutDocument : utilise
```

---

### Module 5 — `Notification`

**Rôle :** Ce module ne contient aucune entité métier au sens DDD — il est **purement réactif**. Il s'abonne aux événements domaine émis par les autres modules et les convertit en communications (email, SMS, push).

#### Entité : `JournalNotification` *(piste d'audit)*

| Attribut | Type | Pourquoi il existe |
|---|---|---|
| `id` | UUID | Identifiant unique |
| `destinataireId` | UUID | Référence vers l'utilisateur cible |
| `canal` | Enum : `EMAIL, SMS, PUSH` | Canal de communication choisi |
| `type` | Enum : `RDV_CONFIRMÉ, RDV_ANNULÉ, RAPPEL_RDV, DOCUMENT_DISPONIBLE, COMPTE_APPROUVÉ` | Type d'événement ayant déclenché la notif |
| `contenu` | JSON | Données de l'événement (sérialisées) |
| `envoyéAt` | DateTime | Horodatage d'envoi |
| `statut` | Enum : `ENVOYÉ, ÉCHEC, IGNORÉ` | Résultat de la livraison |

#### Événements domaine consommés (depuis les autres modules)

| Événement | Émis par | Action |
|---|---|---|
| `RdvConfirmé` | Appointment | Notifier le patient (confirmation) + le médecin (agenda mis à jour) |
| `RdvAnnulé` | Appointment | Notifier le patient et le médecin |
| `RappelRdvProche` | Appointment (tâche planifiée) | Envoyer un rappel 24h avant au patient |
| `OrdonnanceDéposée` | Health Record | Notifier le patient qu'un document est disponible |
| `CompteApprouvé` | Identity & Access | Notifier le médecin de l'activation de son compte |
| `AvisModéré` | Administration | Notifier le patient concerné |

---

### Module 6 — `Administration`

**Rôle :** Gouvernance, modération et supervision de la plateforme. Ces fonctionnalités sont réservées aux administrateurs et évoluent au rythme de la politique interne, indépendamment des fonctionnalités produit.

#### Entité : `Avis`

| Attribut | Type | Pourquoi il existe |
|---|---|---|
| `id` | UUID | Identifiant unique |
| `patientId` | UUID | Auteur de l'avis |
| `médecinId` | UUID | Médecin évalué |
| `rendezVousId` | UUID | Lien vers le rendez-vous concerné (preuve de consultation) |
| `note` | Integer (1–5) | Note attribuée |
| `commentaire` | String | Texte libre |
| `répondreParMédecin` | String (nullable) | Réponse du praticien |
| `statut` | Enum : `VISIBLE, SIGNALÉ, SUPPRIMÉ` | Modération |
| `createdAt` | DateTime | Traçabilité |

#### Entité : `RapportModération`

| Attribut | Type | Pourquoi il existe |
|---|---|---|
| `id` | UUID | Identifiant unique |
| `signaléPar` | UUID | Référence vers l'utilisateur |
| `cibleType` | Enum : `AVIS, PROFIL` | Type de contenu signalé |
| `cibleId` | UUID | Identifiant de l'entité signalée |
| `raison` | String | Motif du signalement |
| `statut` | Enum : `EN_ATTENTE, EXAMINÉ, TRAITÉ` | Suivi admin |

---


