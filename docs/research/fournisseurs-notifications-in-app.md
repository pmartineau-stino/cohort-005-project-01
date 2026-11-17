# Fournisseurs de notifications in-app pour Cadence

- **Date** : 2026-11-17. Pages de tarifs et registre npm consultés ce jour.
- **Question** : quels services tiers de notifications in-app (Knock, Novu, MagicBell, Courier, SuprSend, Engagespot) conviennent à Cadence, et lequel recommander ?
- **Contexte** : React Router 8.3 en framework mode (SSR, loaders/actions), React 19.2, SQLite + Drizzle, vitest. Les utilisateurs sont des lignes en base avec des ids entiers. Il n'y a pas de vraie auth : le panneau DevUI change d'utilisateur via le cookie de session. Seul le canal in-app est concerné. L'auto-hébergement n'est pas requis.
- **Décision** : ce document ne fait que recommander. Le choix final revient à l'humain dans le ticket #5 « Choisir le fournisseur de notifications ».

## Critères

1. **Offre gratuite** suffisante : limites exactes et changements récents.
2. **SDK serveur Node** : nom du paquet, version, forme de l'appel de déclenchement.
3. **Composant React prêt à l'emploi et/ou mode headless** : compatibilité React 19 (peerDependencies) et SSR/React Router.
4. **Plusieurs utilisateurs simulés en local** : comment un utilisateur est identifié, s'il faut le pré-créer, et s'il existe un environnement de dev.
5. **Authentification du flux côté client** : clé publique + id, HMAC ou JWT signé ; ce qui est exigé en dev et en prod.
6. **États lu / vu / archivé** et mécanisme temps réel.
7. **Testabilité sans appeler le vrai service.**

Les versions npm viennent de `npm view <pkg> version peerDependencies peerDependenciesMeta`, exécuté le 2026-11-17. Pour les directives `"use client"`, j'ai téléchargé les archives avec `npm pack` et cherché la chaîne dans `dist/`.

---

## Knock

**1. Offre gratuite.** Plan Free sur https://knock.app/pricing, consulté le 2026-11-17 :

- jusqu'à 10 000 messages/mois ; les messages en échec ou rejetés ne comptent pas ;
- profils stockés illimités ;
- membres d'équipe illimités et environnements illimités ;
- rétention du flux illimitée (configurable), logs conservés 30 jours ;
- la marque Knock reste affichée ; la retirer demande le plan Starter à 250 $/mois.

Aucune limite de MAU n'est indiquée. La seule limite par utilisateur concerne les « guides » : 500 destinataires/mois, ce qui sort de notre périmètre.

**2. SDK serveur.** Paquet `@knocklabs/node`, version 1.41.1, publiée le 2026-11-10. Exemple tiré du README du paquet :

```ts
import Knock from '@knocklabs/node';
const client = new Knock({ apiKey: process.env.KNOCK_API_KEY });
await client.workflows.trigger('new-comment', { recipients: ['42'], data: { lessonId: 7 } });
```

On peut identifier un destinataire en ligne en passant un objet `{ id, name }` au lieu d'une chaîne : il est alors « guaranteed to be identified before the workflow is triggered » (https://docs.knock.app/send-notifications/triggering-workflows/api). Le workflow et son canal in-app se configurent dans le dashboard.

**3. React.** Composants dans `@knocklabs/react` 0.15.1 (`KnockProvider`, `KnockFeedProvider`, `NotificationIconButton`, `NotificationFeedPopover`, plus une CSS à importer). Les hooks headless sont dans `@knocklabs/react-core` 0.17.1 : `useNotifications`, `useNotificationStore`, `useKnockClient` (https://docs.knock.app/in-app-ui/react/headless/feed).

- Compatibilité React 19 confirmée :
  - peer `react`/`react-dom` : `^17 || ^18 || ^19` ;
  - `next` et `@tanstack/react-router` sont des peers marqués `optional: true` dans `peerDependenciesMeta`, donc aucune dépendance à Next.
- Le `dist/esm/index.mjs` publié contient une directive `"use client"`.
- La doc ne dit rien du SSR : non vérifié. Par prudence, monter le composant côté client uniquement.

**4. Utilisateurs simulés.**

- L'id est une chaîne arbitraire (par exemple `String(user.id)`), passée via `user={{ id }}` côté client (https://docs.knock.app/in-app-ui/react/feed).
- Pour changer d'utilisateur, il suffit de changer l'id passé au provider (implémentation non testée).
- Environnements : Production toujours ; Development dès qu'on active les environnements, avec d'autres possibles. Chacun a ses propres clés et données (utilisateurs, messages, logs) (https://docs.knock.app/concepts/environments).

**5. Authentification du flux.**

- Par défaut, la clé publique + l'id utilisateur suffisent : l'« Enhanced security mode » est désactivé.
- En prod, il faut l'activer, car sinon « a request with your public key can act as any user ». Le client doit alors envoyer un JWT RS256 signé par le backend avec `signUserToken`, exporté par `@knocklabs/node` (claim `sub`, expiration par défaut 3600 s).
- On peut garder le mode désactivé en Development seulement.
- Source : https://docs.knock.app/in-app-ui/security-and-authentication

**6. États et temps réel.**

- États : seen, read, interacted, archived, avec des compteurs unread/unseen pour les badges.
- Temps réel : « receives notifications over a websocket » (https://docs.knock.app/in-app-ui/feeds/overview).
- En headless, c'est à l'application de marquer les éléments comme « seen » (`markAsSeen`, `markAllAsSeen`).

**7. Testabilité.**

- Pas de mode mock ni d'émulateur documenté. L'environnement Development sert de bac à sable.
- L'appel serveur se résume à une méthode (`workflows.trigger`), facile à mettre derrière une interface `NotificationSender` et à remplacer par un faux dans vitest.

## Novu

**1. Offre gratuite.** Plan Free sur https://novu.co/pricing, consulté le 2026-11-17 :

- 10 000 workflow runs/mois et au plus 5 workflows ;
- 2 environnements (Dev + Prod) ;
- 3 membres d'équipe ;
- abonnés illimités ;
- 60 événements/s ;
- rétention du flux d'activité de 24 h ;
- tous les canaux, dont In-app.

La suppression de la marque Novu n'est listée qu'à partir du plan Pro (dès 30 $/mois). La marque reste donc probablement sur le Free (déduction, non vérifié).

**2. SDK serveur.** Paquet `@novu/api` 3.20.0, publié le 2026-11-16. L'ancien `@novu/node` (2.6.6) est déprécié : « March 20, 2025 marks the end of support for @novu/node » (avertissement npm). Exemple d'appel :

```ts
import { Novu } from '@novu/api';
const novu = new Novu({ secretKey: process.env.NOVU_SECRET_KEY });
await novu.trigger({ workflowId: 'new-comment', to: { subscriberId: '42' }, payload: { lessonId: 7 } });
```

Source : https://docs.novu.co/platform/sdks/server/typescript

D'après les guides officiels, un abonné inexistant est créé au déclenchement, et un abonné existant est mis à jour avec les champs de `to` (https://docs.novu.co/platform/concepts/subscribers).

**3. React.** `@novu/react` 3.20.0 fournit `<Inbox />` et des hooks headless : `useNotifications`, `useCounts`, `useNovu`.

- Compatibilité React 19 confirmée : peer `react`/`react-dom` `^18 || ^19`, avec `react-dom` optionnel. Un paquet séparé `@novu/nextjs` existe ; `@novu/react` ne dépend pas de Next.
- L'exemple officiel utilise `useNavigate` de React Router via la prop `routerPush` (https://docs.novu.co/platform/inbox/setup-inbox).
- Pas de directive `"use client"` dans `@novu/react` ; seul l'exemple Next l'ajoute. La doc ne dit rien du SSR hors Next : non vérifié.
- Un « keyless mode » existe : `<Inbox />` sans props, données temporaires pendant 24 h, pratique pour une démo.

**4. Utilisateurs simulés.**

- `subscriberId` est une chaîne arbitraire.
- Development et Production ont chacun leur `applicationIdentifier` et leur clé secrète. Leurs abonnés et leurs flux d'activité sont isolés (https://docs.novu.co/platform/concepts/environments).
- Les environnements personnalisés sont réservés aux plans Team et Enterprise.

**5. Authentification du flux.**

- `applicationIdentifier` (public) + `subscriberId` suffisent tant que le chiffrement HMAC est désactivé.
- Une fois l'option HMAC activée dans le fournisseur In-App, il faut passer `subscriberHash` = HMAC-SHA256(secretKey, subscriberId) en hex, sinon « the Inbox will not load ».
- C'est un réglage par environnement : on peut l'activer seulement en Production.
- Source : https://docs.novu.co/platform/inbox/prepare-for-production

**6. États et temps réel.**

- États : read/unread, archive (https://docs.novu.co/platform/inbox/headless-mode).
- seen : marqué automatiquement après environ 1 s de visibilité, de façon irréversible.
- snooze : disponible à partir du SDK 3.3.0 (https://docs.novu.co/platform/inbox/features/snooze).
- Temps réel par websocket : les hooks se mettent à jour seuls, et on peut écouter l'événement `notifications.notification_received`.

**7. Testabilité.**

- Pas de mode mock documenté. L'environnement Development sert de bac à sable.
- Novu est open source (https://github.com/novuhq/novu), donc on pourrait en théorie lancer une instance locale pour des tests de bout en bout. Ce n'est pas requis et je ne l'ai pas évalué ici.
- L'appel serveur se résume à une méthode (`trigger`), facile à mettre derrière une interface.

## MagicBell

**1. Offre gratuite.** Plan Builder à 0 $ sur https://www.magicbell.com/pricing, consulté le 2026-11-17 :

- **1 000 deliveries/mois**, nettement moins que les autres fournisseurs ;
- « Unlimited Users » (non défini), membres d'équipe illimités ;
- 1 projet.

Plan suivant : Startup à 249 $/mois. La page ne dit rien des restrictions de l'inbox ni de la marque.

**2. SDK serveur.** Le paquet actuel est `magicbell-js` 1.8.0, avec `import { Client } from 'magicbell-js/project-client'`. On envoie avec `client.broadcasts.createBroadcast({ recipients: [{ externalId: '42' }], ... })`, authentifié par un jeton projet (https://www.magicbell.com/docs/libraries/magicbell-js). Ancien SDK : `magicbell` 4.5.1.

**3. React.** Paquet actuel : `@magicbell/react` 1.14.0, avec `ContextProvider` (prop `token`), `Inbox` et `FloatingInbox` (https://www.magicbell.com/docs/libraries/magicbell-react).

- Il ne déclare **aucune peerDependency** sur `react` ; la compatibilité React 19 n'est pas déclarée, donc non vérifiée.
- Plusieurs modules de `dist/esm` (`floating-inbox.js`, etc.) ont une directive `"use client"`.
- Pas de hooks headless documentés pour ce paquet.
- Les anciens `@magicbell/magicbell-react` 11.5.1 et `@magicbell/react-headless` 5.5.1 (peer `react >= 18.3.1`) existent toujours. Leur statut n'est pas clair : la doc actuelle ne les mentionne plus.

**4. Utilisateurs simulés.** L'utilisateur est identifié par `user_external_id` ou `user_email` dans le JWT. La doc ne dit pas s'il est créé automatiquement : non vérifié. Les environnements ne sont pas documentés, et le plan gratuit n'a qu'un projet.

**5. Authentification du flux.** Un **User JWT est obligatoire**, côté composant comme côté README (« You'll need a User JWT to use the SDK »). Il est signé en HS256 avec la Secret Key du projet et contient `user_external_id`/`user_email` et `api_key` (https://www.magicbell.com/docs/api/authentication/user). Le JWT est donc requis aussi en dev.

**6. États et temps réel.**

- read/unread, archive et suppression (https://www.magicbell.com/docs/channels/in_app). « Seen » n'est pas mentionné.
- Connexion persistante, sans précision sur le transport. Le paquet dépend de `reconnecting-websocket`, donc probablement des websockets.

**7. Testabilité.** Pas de mode mock documenté. Un seul projet sur le plan gratuit, donc pas d'environnement de dev séparé.

## Courier

**1. Offre gratuite.** Plan Developer sur https://www.courier.com/pricing, consulté le 2026-11-17 :

- 10 000 sends/mois ;
- les envois depuis les environnements de test comptent comme ceux de production ;
- pas de frais par siège.

Plan suivant : Business à 0,005 $ par send, « including the first 10,000 ». Autrement dit, passer à Business fait payer dès le premier envoi. Les environnements personnalisés ne figurent que dans Enterprise. La page ne dit rien des restrictions de l'Inbox ni de la marque.

**2. SDK serveur.** Paquet `@trycourier/courier` 10.0.0, publié le 2026-10-29 : une version majeure récente, réécrite (Node 20+ requis). Exemple tiré du README :

```ts
import Courier from '@trycourier/courier';
const client = new Courier(); // lit COURIER_API_KEY
await client.send.message({ message: { to: { user_id: '42' }, content: { title: '…', body: '…' } } });
```

Le README montre `to: { email }`. L'envoi vers `user_id` et le routage vers l'Inbox sont déduits de la doc Inbox, mais je n'ai pas vu d'exemple exact : non vérifié. Les profils se créent avec `client.profiles.create(id, …)`.

**3. React.** `@trycourier/courier-react` 9.3.5, avec `<CourierInbox />` et `useCourier()` pour une UI personnalisée.

- Peer `react`/`react-dom` `>=18.0.0`, donc React 19 est accepté.
- La doc précise que l'Inbox est « client-only » et qu'il faut `'use client'` en Next 13+. Le paquet publié ne contient pas cette directive.
- Pour React 17, il existe le paquet séparé `@trycourier/courier-react-17`.
- Source : https://www.courier.com/docs/sdk-libraries/courier-react-web

**4. Utilisateurs simulés.**

- Connexion avec `courier.shared.signIn({ userId, jwt })`. Changer d'utilisateur demande de rappeler `signIn` avec un nouveau JWT.
- Les clés sont par environnement (test et production), mais je n'ai pas pu ouvrir la page de doc sur les environnements (404) : non vérifié.

**5. Authentification du flux.**

- Un **JWT est obligatoire, y compris en dev**. Le backend l'obtient via `POST /auth/issue-token` (ou `client.auth.issueToken`) avec le scope `user_id:<id> inbox:read:messages inbox:write:events`.
- Sans `expires_in`, le jeton n'expire jamais ; les SDK ne le renouvellent pas.
- Source : https://www.courier.com/docs/platform/inbox/authentication

**6. États et temps réel.** États read, opened, archived (https://www.courier.com/docs/platform/inbox/inbox-overview). Le websocket n'est ouvert qu'après un appel explicite à `inbox.listenForUpdates()`.

**7. Testabilité.** Il existe un environnement de test, mais ses envois sont décomptés du quota. Pas de mode mock documenté.

## SuprSend

**1. Offre gratuite.** Plan Free sur https://www.suprsend.com/pricing, consulté le 2026-11-17 :

- 10 000 notifications/mois ;
- membres d'équipe illimités ;
- « Multiple Environments » ;
- logs conservés 30 jours ;
- inbox embarquée, inbox headless et toasts inclus, mais le flux in-app affiche la marque SuprSend (« with SuprSend branding »).

Aucune limite de MAU n'est indiquée. Plan suivant : Essentials à 100 $/mois.

**2. SDK serveur.** Paquet `@suprsend/node-sdk` 1.15.1, publié le 2026-05-19. Exemple tiré du README :

```js
const { Suprsend, WorkflowTriggerRequest } = require('@suprsend/node-sdk');
const supr = new Suprsend(WORKSPACE_KEY, WORKSPACE_SECRET);
await supr.workflows.trigger(new WorkflowTriggerRequest({ workflow: 'new-comment', recipients: [{ distinct_id: '42' }], data: {} }));
```

Le README précise que cet appel « doesn't require user creation before hand » si le destinataire est décrit en ligne.

**3. React.** `@suprsend/react` 1.6.0 propose `SuprSendProvider`, `Inbox`, `NotificationFeed` et `Preferences`. Les hooks sont dans `@suprsend/react-core` 2.5.0.

- Peer `react >= 16.8.0` : React 19 n'est pas exclu, mais pas explicitement testé.
- L'ancien `@suprsend/react-inbox` n'est plus maintenu (https://docs.suprsend.com/docs/inbox-react).
- Pas de directive `"use client"` dans `react-core`. Rien sur le SSR dans la doc : non vérifié.

**4. Utilisateurs simulés.**

- `distinctId` est arbitraire : « When a value is passed, the SDK creates and authenticates the user », et `null` déconnecte (README de `@suprsend/react`).
- Trois espaces de travail (Sandbox, Staging, Production), chacun avec ses clés et ses données isolées.
- Un « Test Mode » existe en Staging (https://docs.suprsend.com/docs/workspaces).

**5. Authentification du flux.**

- Clé publique + `distinctId` suffisent par défaut.
- En « enhanced security mode », il faut fournir `userToken`, un JWT ES256 signé avec la Signing Key ; ce mode est recommandé en production (https://docs.suprsend.com/docs/client-authentication).

**6. États et temps réel.**

- Temps réel par socket (`socketHost`, `disconnectWhenHidden`) (https://github.com/suprsend/suprsend-react-sdk/blob/main/docs/in-app-feed.md).
- Je n'ai pas trouvé de doc officielle détaillant seen/read/archive : non vérifié.

**7. Testabilité.** Staging avec Test Mode, mais pas de mock.

## Engagespot (candidat écarté)

- **Prix non vérifiés.** Je n'ai pas pu consulter la page de tarifs officielle : `engagespot.co/pricing` et `www.engagespot.co/pricing` renvoient 404, et `engagespot.com` une erreur TLS. Seuls des agrégateurs tiers mentionnent un plan « Launch » gratuit à 10 000 déclenchements/mois, ce qui n'est pas une source primaire.
- **SDK peu maintenus** d'après npm :
  - `@engagespot/node` 1.7.0 : dernière publication le 2024-11-06 ;
  - `@engagespot/react-component` 3.0.1 (peer `react >=16.8.3`) : dernière publication le 2025-05-09.
- La documentation (https://docs.engagespot.co/) n'a pas pu être lue (page vide).
- Écarté faute de sources primaires vérifiables.

**Autres exclusions.**

- Pusher Beams fait du push, pas de flux in-app.
- OneSignal et Firebase Cloud Messaging sont orientés push ; je ne les ai pas étudiés.

---

## Tableau comparatif

| Critère | Knock | Novu | MagicBell | Courier | SuprSend |
|---|---|---|---|---|---|
| Gratuit / mois | 10 000 messages | 10 000 runs, 5 workflows | **1 000 deliveries** | 10 000 sends | 10 000 notifs |
| Sièges / environnements (free) | illimités / illimités | 3 / Dev + Prod | illimités / 1 projet | non précisé / test + prod (non vérifié) | illimités / Sandbox, Staging, Prod |
| Marque sur le free | oui | probable | non précisé | non précisé | oui |
| SDK Node | `@knocklabs/node` 1.41.1 | `@novu/api` 3.20.0 | `magicbell-js` 1.8.0 | `@trycourier/courier` 10.0.0 | `@suprsend/node-sdk` 1.15.1 |
| React UI + headless | oui + oui | oui + oui | oui + non documenté | oui + `useCourier` | oui + `react-core` |
| Peer React 19 | `^19` explicite | `^19` explicite | non déclaré | `>=18` | `>=16.8` |
| `"use client"` publié | oui | non | oui | non (doc : client-only) | non |
| Id utilisateur | chaîne libre, identification en ligne | chaîne libre, créé au trigger | external_id dans le JWT | userId + JWT | distinctId, créé par le SDK |
| Auth en dev | clé publique + id | clé publique + id | **JWT obligatoire** | **JWT obligatoire** | clé publique + id |
| Auth en prod | JWT RS256 (enhanced security) | HMAC `subscriberHash` | JWT HS256 | JWT (issue-token) | JWT ES256 (enhanced security) |
| États | seen / read / interacted / archived | seen / read / archived / snooze | read / archived | read / opened / archived | non vérifié |
| Temps réel | websocket | websocket | connexion persistante | websocket (opt-in) | socket |
| Test / mock | env Dev, pas de mock | env Dev, OSS auto-hébergeable | non | env test décompté du quota | Staging + Test Mode |

## Recommandation

**Recommandation : Knock.** C'est l'offre qui couvre le mieux les sept critères au moindre coût d'intégration :

- 10 000 messages/mois, sièges et environnements illimités sur le plan gratuit ;
- compatibilité React 19 déclarée, sans dépendance à Next (les peers `next` et `@tanstack/react-router` sont optionnels), et une directive `"use client"` déjà présente ;
- en dev, l'authentification se limite à la clé publique + `String(user.id)`, ce qui colle au changement d'utilisateur du panneau DevUI. Passer au JWT signé avec `signUserToken` en prod est un ajout local, sans refonte ;
- états seen/read/archived complets, websocket, et hooks headless si l'UI fournie ne convient pas ;
- un déclenchement serveur en une seule méthode, facile à cacher derrière une interface et à remplacer par un faux dans vitest.

**Second choix : Novu.** Offre équivalente sur le volume (10 000 runs), React 19 explicite, et un exemple officiel avec React Router (`routerPush`). Il est open source, donc auto-hébergeable si l'on veut un jour des tests de bout en bout sans le cloud. L'authentification de prod (HMAC) est encore plus simple. Ses limites sur le plan gratuit : 3 sièges, 5 workflows, 24 h de rétention des logs, et pas de `"use client"`, sans conséquence hors Next.

**À éviter pour ce projet :**

- **MagicBell** : 1 000 envois/mois, JWT obligatoire même en dev, aucune peer React déclarée.
- **Courier** : JWT obligatoire même en dev, sends de test décomptés, SDK Node 10.0.0 tout juste réécrit.
- **Engagespot** : prix invérifiables, SDK inactifs.
- **SuprSend** reste une option viable, mais moins bien documentée sur les états et sans compatibilité React 19 explicite.

**Points d'attention, quel que soit le choix :**

1. Rendre le composant de notifications côté client uniquement : aucun fournisseur ne documente le SSR hors Next.
2. Les ids entiers sont convertis en chaînes : utiliser `String(user.id)` partout.
3. Le choix final appartient à l'humain (ticket #5 « Choisir le fournisseur de notifications »).

## Non vérifié

- Comportement SSR réel de chaque composant React dans React Router 8.3 (accès à `window` au chargement du module) : non testé.
- Knock : création implicite d'un destinataire passé seulement par son id, sans identification en ligne.
- Novu : affichage de la marque sur le plan Free (déduction tirée de la page de tarifs).
- MagicBell : création automatique des utilisateurs, statut des anciens paquets React.
- Courier : page des environnements (404), envoi vers `user_id` montré seulement dans la doc Inbox.
- SuprSend : actions read/seen/archive.
- Engagespot : tarifs et documentation.

## Sources

Toutes consultées le 2026-11-17.

- Knock :
  - tarifs https://knock.app/pricing
  - sécurité https://docs.knock.app/in-app-ui/security-and-authentication
  - flux React https://docs.knock.app/in-app-ui/react/feed
  - headless https://docs.knock.app/in-app-ui/react/headless/feed
  - vue d'ensemble des flux https://docs.knock.app/in-app-ui/feeds/overview
  - environnements https://docs.knock.app/concepts/environments
  - trigger API https://docs.knock.app/send-notifications/triggering-workflows/api
  - npm `@knocklabs/node`, `@knocklabs/react`, `@knocklabs/react-core`
- Novu :
  - tarifs https://novu.co/pricing
  - Inbox https://docs.novu.co/platform/inbox/setup-inbox
  - HMAC https://docs.novu.co/platform/inbox/prepare-for-production
  - SDK serveur https://docs.novu.co/platform/sdks/server/typescript
  - environnements https://docs.novu.co/platform/concepts/environments
  - headless https://docs.novu.co/platform/inbox/headless-mode
  - snooze https://docs.novu.co/platform/inbox/features/snooze
  - abonnés https://docs.novu.co/platform/concepts/subscribers
  - npm `@novu/api`, `@novu/react`, `@novu/node` (déprécié)
- MagicBell :
  - tarifs https://www.magicbell.com/pricing
  - JWT utilisateur https://www.magicbell.com/docs/api/authentication/user
  - magicbell-js https://www.magicbell.com/docs/libraries/magicbell-js
  - React https://www.magicbell.com/docs/libraries/magicbell-react
  - canal in-app https://www.magicbell.com/docs/channels/in_app
  - npm `@magicbell/react`, `magicbell-js`, `@magicbell/magicbell-react`, `@magicbell/react-headless`
- Courier :
  - tarifs https://www.courier.com/pricing
  - React https://www.courier.com/docs/sdk-libraries/courier-react-web
  - auth Inbox https://www.courier.com/docs/platform/inbox/authentication
  - Inbox https://www.courier.com/docs/platform/inbox/inbox-overview
  - npm `@trycourier/courier` (README), `@trycourier/courier-react`
- SuprSend :
  - tarifs https://www.suprsend.com/pricing
  - auth client https://docs.suprsend.com/docs/client-authentication
  - espaces de travail https://docs.suprsend.com/docs/workspaces
  - Inbox React https://docs.suprsend.com/docs/inbox-react
  - flux in-app https://github.com/suprsend/suprsend-react-sdk/blob/main/docs/in-app-feed.md
  - npm `@suprsend/node-sdk` (README), `@suprsend/react` (README), `@suprsend/react-core`
- Engagespot :
  - https://docs.engagespot.co/ (illisible)
  - npm `@engagespot/node`, `@engagespot/react-component`
