# App IT — Validation Onboarding/Offboarding avec aperçu mail

Application web de demo + workflow n8n pour le traitement des demandes IT
(onboarding / offboarding) avec validation de l'envoi du mail d'accomplissement.

## Fonctionnalites

- **Formulaire de demande** : type (onboarding/offboarding), nom agent, initial,
  poste, grade IT (onboarding uniquement).
- **Champ de validation** : la case « Demande deja validee » ajoute
  `validation_mail: "OK"` au payload envoye au webhook n8n.
  - `validation_mail = OK` -> le workflow enchaine directement la suite
    (aucune pause) et repond `statut: validation_ok_actions_executees_mail_envoi`.
    Il n'y a **pas de champ e-mail obligatoire** : seul `validation_mail = OK`
    porte la validation.
  - sinon -> le workflow met les actions IT executees puis attend une validation
    humaine sur une page d'**aperçu du mail** (formulaire resume du node Wait) ;
    l'app recoit `statut: actions_executees_attente_validation_mail` et le lien
    d'aperçu a ouvrir par le validateur.
- **Panneau « Endpoint webhook n8n (listening) »** : URL du webhook + jeton
  d'en-tete `X-App-Token`, modifiables dans l'interface et persistes dans le
  navigateur (localStorage). Aucun secret en dur dans le code.

## Workflow n8n

Fichier : [`workflow-n8n.json`](./workflow-n8n.json) — workflow
« App IT - Validation Onboarding/Offboarding avec apercu mail ».

Chaine principale :

1. `Webhook App IT` — `POST /webhook/app-it-validation`, authentification par
   en-tete `X-App-Token` (credential **Header Auth**).
2. `Valider payload` (Code) — normalise le payload, derive le niveau
   (`administrateur`/`technicien` -> it, `manager` -> superviseur, autre -> simple),
   lit `validation_mail` (miss en majuscules).
3. `Payload valide?` / `Reponse invalide` — HTTP 400 + liste des erreurs.
4. `Aiguiller demande` (Switch) — onboarding (3 variantes de grade) / offboarding /
   inconnu. Etapes IT (Teams, AD, acces partages, GLPI, serveur) = nodes HTTP
   **desactives** (placeholders a configurer avec les API internes).
5. `Construire apercu mail` (Code) — sujet + corps HTML + texte du mail
   d'accomplissement.
6. `Validation prealable OK?` (IF) — branche le flux selon `validation_mail` :
   - **true** (`OK`) : `Reponse app deja validee` puis `Envoyer le mail` sans pause.
   - **false** : `Reponse a l'app` (lien d'aperçu) -> `Apercu et validation envoi`
     (Wait, resume **form**, dropdown `validation_mail` : `OK`/`annuler`) ->
     `Decision envoi?` (`$json.validation_mail == "OK"`) -> `Envoyer le mail`.
7. `Envoyer le mail` — node HTTP **desactive** : remplacer l'URL par l'API mail
   interne (ou la remplacer par un node Send Email SMTP).

### Import

n8n -> Workflows -> **Import from File** -> `workflow-n8n.json`. Puis :

- creer/sélectionner le credential **Header Auth** (name: `X-App-Token`,
  value: le jeton choisi) sur le node `Webhook App IT` ;
- configurer/activer les nodes IT et `Envoyer le mail` ;
- **Publish** le workflow pour que le webhook `POST /webhook/app-it-validation`
  soit actif (ou utiliser le mode test `POST /webhook-test/...` avec « Listen for test event »).

## Lancer l'app

Aucun build : ce sont des fichiers statiques.

```bash
cd app-it-validation
python3 -m http.server 8080
# http://localhost:8080
```

Ouvrir la page, deplier « ⚙️ Endpoint webhook n8n (listening) », renseigner :

- **URL** : `http://<hote-n8n>:5678/webhook/app-it-validation`
- **Jeton** : la meme valeur que le credential `X-App-Token` dans n8n

puis **Enregistrer**, remplir le formulaire et **Envoyer la demande**.

### Exemples de payload

Validation directe (suite enchainee) :

```json
{
  "type": "onboarding",
  "agent": { "name": "Nomeny Rasolonjato", "initial": "NR", "poste": "Administrateur systeme", "grade": "administrateur" },
  "validation_mail": "OK"
}
```

Sans validation (attente d'validation humaine via l'aperçu mail) :

```json
{
  "type": "offboarding",
  "agent": { "name": "Nomeny Rasolonjato", "initial": "NR", "poste": "Comptable" }
}
```

## Securite

- Le jeton `X-App-Token` n'est **jamais** commit : il vit dans le credential n8n
  et le localStorage du navigateur de l'agent qui l'a saisi.
- Les erreurs n8n en texte brut (ex. `No authentication data defined on node!`)
  sont affichees telles quelles par l'app (pas de plantage JSON).
- En production : servir l'app en HTTPS derriere le meme reverse proxy que n8n
  (CORS prefight deja gere par n8n pour `/webhook/*`).
