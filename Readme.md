# Workflow n8n — Création d'un nouveau projet

Ce workflow automatise la création d'un projet de bout en bout : un membre remplit un formulaire Slack, Marie valide ou modifie, et la validation crée automatiquement les espaces de travail associés (NextCloud, Notion, Slack) et enregistre la mission dans Ricobot.

Le système complet repose sur **deux applications Slack** et **quatre workflows n8n** : le workflow principal décrit ci-dessous, un workflow qui lance le formulaire, et deux workflows qui tiennent à jour les listes de clients et d'équipe. Ces compléments sont décrits en fin de document.

## Logique générale

Tout passe par un seul webhook (`/valide_form`) qui reçoit l'ensemble des interactions Slack. Un nœud Code décode le payload, puis un Switch oriente le traitement selon le type d'événement.

Le cycle de vie d'un projet se déroule en trois temps :

1. **Soumission** — Le formulaire `new_project_form` est soumis. Le projet est enregistré en base et un message récapitulatif part vers Marie avec deux boutons : *Valider* et *Modifier*.
2. **Modification (optionnelle)** — Le bouton *Modifier* ouvre une modale pré-remplie. À la soumission, la base est mise à jour et le message de validation est rafraîchi.
3. **Validation** — Le bouton *Valider* lance la création complète : dossiers NextCloud, page et sous-pages Notion, canal Slack, mission Ricobot.

## Le déclencheur et le routage

Le webhook `Attente action formulaire1` (POST `/valide_form`) est le point d'entrée unique. Le nœud `Extraction données formulaire1` décode le payload Slack et distingue deux familles : les clics de boutons (`block_actions`) et les soumissions de modale (`view_submission`), ces dernières pouvant être un nouveau projet (`new_project_form`) ou une édition (`rh_edit_form`).

Le `Switch type d'actions1` aiguille ensuite vers la bonne branche. Chaque branche commence par répondre immédiatement à Slack pour éviter un timeout.

## Les trois branches

**Nouveau projet.** Le projet est inséré dans la table `Id_project`. Si le client choisi était « Autre », il est ajouté à la table `Client`. Un message à boutons est envoyé à Marie, et le timestamp de ce message est stocké en base pour pouvoir le mettre à jour plus tard.

**Clic sur un bouton.** Un second Switch lit l'action. *Modifier* recharge les listes (clients, équipe) et ouvre une modale Slack pré-remplie via `views.open`. *Valider* récupère le projet, génère le nom de dossier, puis déclenche la création complète.

**Édition soumise.** Les données sont mises à jour en base et le message Slack d'origine est rafraîchi avec les nouvelles valeurs.

## La création complète

Quand un projet est validé, le nœud `Extraction_donnes1` calcule le nom du dossier au format `CODEPÔLE-PRJ0000-NOM_PROJET` (ou `MULTI_POLE` si plusieurs pôles), puis quatre chaînes s'exécutent en parallèle :

- **NextCloud** : création du dossier principal puis des sous-dossiers Documents_Admins, Livrables et Facturations.
- **Notion** : une page projet est créée avec une couverture choisie par IA (un agent `llama-3.1-8b-instant` renvoie un emoji et une requête Unsplash, l'image étant ensuite récupérée via Unsplash). Cette page sert de parent à cinq sous-pages (contexte, calendrier, suivi financier, documentation, notes de réunion) et à une base de suivi. Un message de confirmation avec les liens Notion et NextCloud est envoyé à Marie.
- **Slack** : un canal d'équipe est créé, puis les membres y sont invités. Une annonce est aussi postée dans le canal des nouveaux projets.
- **Ricobot** : la mission est créée dans Strapi avec le statut « à valider ».

## Intégrations utilisées

Slack (messages, modales, canal, invitations), NextCloud (dossiers), Notion (pages et base de suivi), Unsplash (image de couverture), un modèle de langage `llama-3.1-8b-instant` (choix de l'emoji et de la requête image), et Ricobot/Strapi (enregistrement de la mission). Le workflow s'appuie aussi sur les Data Tables natives de n8n pour sa base de données.

## Bases de données (Data Tables)

- **Id_project** — table maîtresse des projets (nom, client, pôles, description, montant, contacts, date de début, équipe, ts du message Slack).
- **Client** — liste des clients, complétée automatiquement quand on saisit « Autre ».
- **Equipe** — correspondance entre nom et identifiant utilisateur Notion.
- **pole_person_table** — correspondance entre nom et identifiant utilisateur Slack.

## Identifiants d'interaction

Le chemin webhook est `valide_form`. Les modales utilisent les `callback_id` `new_project_form` et `rh_edit_form`. Les boutons utilisent les `action_id` `validate_project` et `edit_project`. Les pôles sont mappés vers des codes courts (dev, jur, cons, com, egov, bizdev, rp), avec `multi` lorsque plusieurs pôles sont sélectionnés.

## À configurer avant la mise en production

Plusieurs nœuds HTTP contiennent des placeholders à remplacer par de vraies clés : `SLACK_BOT_KEY`, `API_NOTION_KEY`, `API_UNSPLACH_KEY`, `API_KEY_RICO` et `SLACK_N8N_BOT`. Pensez aussi à vérifier les identifiants en dur (utilisateur Slack de Marie, canal des nouveaux projets, base Notion).

## Points d'attention

- Les dossiers NextCloud sont créés sous un chemin `/tests/`, à retirer en production.
- Le nœud `AI Agent` n'a pas d'entrée connectée dans l'export : sans alimentation, la couverture Unsplash ne recevra pas les données du projet.
- La branche d'édition est reliée au flux de création, ce qui peut relancer toute la création lors d'une simple modification — à vérifier si c'est voulu.

---

## Les applications Slack

Le système utilise deux applications Slack qui se répartissent les rôles :

- **Slackbot_commande** — la porte d'entrée. Elle fournit la commande Slack qui démarre tout le processus (elle appelle le webhook `/num_commande`) et reçoit l'interactivité, c'est-à-dire les clics de boutons et les soumissions de formulaire envoyés au webhook `/valide_form`.
- **n8n_commande** — l'application qui agit. C'est elle qui ouvre les formulaires (`views.open`), envoie les messages, crée le canal de projet et invite les membres. C'est le bot derrière la credential `n8n_bot_num`.

En résumé : *Slackbot_commande* déclenche et écoute, *n8n_commande* exécute. La répartition exacte (quel jeton dans quel nœud) doit correspondre aux réglages de vos deux apps côté Slack.

## Les workflows compagnons

Trois workflows entourent le workflow principal : un qui le lance, et deux qui préparent ses données.

### Execute_commande_numéricité — le point de départ

C'est lui qui ouvre le formulaire de création.

- Déclenché par la commande Slack via le webhook `/num_commande`.
- Il charge en parallèle la liste des clients (table `Client`) et celle des équipes (table `Equipe`), puis les met au format des menus déroulants Slack. L'option « Autre » est ajoutée à la liste des clients pour permettre d'en saisir un nouveau.
- Il ouvre alors la modale « Nouveau Projet » (`new_project_form`), pré-remplie avec ces listes.

À la soumission, la modale envoie les données au webhook `/valide_form` : c'est ce workflow qui démarre toute la chaîne décrite plus haut.

### Récupération de client notion — synchronisation des clients

Garde la liste des clients à jour pour le formulaire.

- Déclenché automatiquement par un planificateur.
- Il lit la base Notion « Master Data (Project) » et récupère les options du champ « Client ».
- Pour chaque client, il vérifie s'il existe déjà dans la table `Client`. S'il est nouveau, il l'ajoute ; sinon il passe au suivant.

### Récupération équipe — synchronisation de l'équipe

Garde la liste des membres à jour pour le formulaire et pour les correspondances Notion/Slack.

- Déclenché par un planificateur (tous les mois).
- Il lit toutes les pages de la base Notion et extrait les membres uniques du champ « Equipe ».
- Pour chaque membre, il vérifie s'il existe déjà dans la table `Equipe`. S'il est nouveau, il l'ajoute avec son nom et son identifiant Notion (`id_notion`) ; sinon il passe au suivant.

Ces deux derniers workflows tournent en arrière-plan : ils évitent d'avoir à mettre à jour les listes à la main, et garantissent que le formulaire propose toujours des clients et des membres à jour.