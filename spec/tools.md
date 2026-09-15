# Tools — the 21 real definitions

Extracted from the reference instance's tool registry (`spec/tools.json` is the machine-readable form, in the Ollama tool format, loadable as-is). Every tool has a **tier** from `policy.md`; the model sees the JSON schema, the description and nothing else. Descriptions are in French, the language the reference user speaks to Kora — they were tuned on a 12B model and the wording is part of what works; translate them for your user, keep their shape (what it does, when to call it, what never to do).

| Tool | Tier | Arguments (required in bold) |
|---|---|---|
| **Web (auto)** | | |
| `web_search` | `auto` | **query** |
| `read_url` | `auto` | **url** |
| **PC, read-only (auto)** | | |
| `list_directory` | `auto` | path |
| `read_file` | `auto` | **path** |
| `system_info` | `auto` | — |
| `plan_cleanup` | `auto` | **directory**, older_than_days, extensions, name_contains |
| **PC, write — confined to the sandbox folder (irreversible)** | | |
| `write_file` | `irreversible` | **filename**, **content**, **reason** |
| `update_file` | `irreversible` | **filename**, **content**, **reason** |
| `append_file` | `irreversible` | **filename**, **content**, **reason** |
| `execute_cleanup` | `irreversible` | plan_id, **reason** |
| **PC actions (confirm)** | | |
| `open_url` | `confirm` | **url**, **reason** |
| `open_path` | `confirm` | **path**, **reason** |
| `launch_app` | `confirm` | **name**, **reason** |
| `control_media` | `confirm` | **action** ∈ {play_pause, next_track, previous_track, volume_up, volume_down, mute_media, unmute_media}, **reason** |
| `open_site_search` | `confirm` | **site**, **query**, **reason** |
| `play_top_result` | `confirm` | **site** ∈ {deezer, youtube}, **query**, **reason** |
| **Communication, memory, recall (auto)** | | |
| `delegate_to_provider` | `auto` | **provider** ∈ {claude, redaction, chatgpt, deepseek}, **task**, save_as, attach_path |
| `write_memory` | `auto` | section ∈ {arbitrage, patterns, fils_ouverts, anomalies}, **entry** |
| `update_memory` | `auto` | **section** ∈ {arbitrage, patterns, fils_ouverts, anomalies}, **entry_id**, **new_entry** |
| `search_past_conversations` | `auto` | **query** |
| `recall_provider_answers` | `auto` | **query** |

## Descriptions, as the model reads them

### `web_search` — `auto`

> Recherche des informations factuelles vérifiables sur le web par mots-clés (actualités, données, personnes, événements). Ne retourne que des extraits courts — utilise read_url si l'utilisateur donne une URL précise à lire en entier.

### `read_url` — `auto`

> Lit le contenu réel d'une URL précise (page web donnée par l'utilisateur). Utilise ceci, jamais web_search, quand on te donne un lien exact à consulter.

### `list_directory` — `auto`

> Liste le contenu d'un dossier sur le PC de l'utilisateur (lecture seule, aucune modification possible). Chemin absolu ou commençant par '~'. Sans argument, liste le dossier personnel (home).

### `read_file` — `auto`

> Lit le contenu texte d'un fichier précis sur le PC de l'utilisateur (lecture seule). Fichiers texte uniquement (code, config, notes...) — pas les binaires/images. Un chemin qui ressemble à des identifiants/secrets (.env, .ssh/, config.toml, credentials...) déclenche une confirmation de l'utilisateur avant lecture ; les valeurs qui ressemblent à des clés/tokens dans le contenu lu sont de toute façon masquées avant de te parvenir.

### `system_info` — `auto`

> Donne des informations système du PC de l'utilisateur : espace disque, mémoire, plateforme, temps de fonctionnement. Lecture seule, aucun argument.

### `plan_cleanup` — `auto`

> PREMIER TEMPS d'une suppression de fichiers — LECTURE SEULE, rien n'est déplacé. Calcule la liste des fichiers d'un dossier AUTORISÉ (Captures d'écran, Téléchargements, ~/Kora — la liste réelle t'est renvoyée si le dossier est refusé) qui correspondent au filtre, et renvoie un identifiant de plan avec le compte, la taille totale et des exemples. Tu ne choisis JAMAIS les fichiers un par un : le filtre sélectionne. Fichiers directement dans le dossier seulement — jamais un sous-dossier, jamais un fichier caché ou sensible. Ensuite, présente le plan à l'utilisateur puis appelle execute_cleanup(plan_id) pour l'exécuter (avec sa confirmation).

### `write_file` — `irreversible`

> Écrit un nouveau fichier texte — UNIQUEMENT dans ton dossier dédié sur le PC de l'utilisateur (~/Kora), jamais ailleurs (le nom de fichier ne doit pas contenir de chemin, juste un nom simple, ex. 'resume.md'). Refuse si un fichier du même nom existe déjà — ne remplace jamais un fichier existant, propose un autre nom ou demande à l'utilisateur de le supprimer d'abord. LIMITÉ à un texte COURT (~800 caractères max, ex. une note que l'utilisateur vient de te donner à sauvegarder telle quelle) — refusé automatiquement au-delà, PAS une suggestion : pour un résumé/une narration longue, utilise delegate_to_provider(provider="redaction", save_as="...") à la place, qui rédige ET enregistre en un seul appel sans risque de troncature. Cette action a un effet réel et déclenche systématiquement une confirmation de l'utilisateur avant d'être exécutée — fournis `reason` pour qu'il puisse décider vite.

### `update_file` — `irreversible`

> Remplace le contenu d'un fichier texte EXISTANT dans ton dossier dédié sur le PC de l'utilisateur (~/Kora) — jamais ailleurs (le nom de fichier ne doit pas contenir de chemin, juste un nom simple). Refuse si le fichier n'existe pas encore (utilise write_file pour en créer un). Réservé à un texte COURT que TU as toi-même sous la main dans cette conversation — refusé automatiquement au-delà d'environ 800 caractères, PAS une suggestion. Cette action a un effet réel et irréversible (le contenu précédent est perdu) et déclenche systématiquement une confirmation de l'utilisateur avant d'être exécutée — fournis `reason` pour qu'il puisse décider vite.

### `append_file` — `irreversible`

> Ajoute du texte à LA FIN d'un fichier EXISTANT dans ton dossier dédié (~/Kora), SANS toucher au contenu déjà présent — PRÉFÈRE CET OUTIL à update_file dès que la demande est d'AJOUTER une entrée/idée/ligne à un fichier qui existe déjà (ex. 'ajoute ça à mes notes') : tu n'as besoin de fournir QUE ce qui s'ajoute, jamais de relire le fichier ni de reproduire son contenu existant. Refuse si le fichier n'existe pas encore (utilise write_file pour en créer un). N'utilise update_file QUE si l'utilisateur demande explicitement de réorganiser/réécrire le fichier entier. Réservé à un texte COURT — refusé automatiquement au-delà d'environ 800 caractères. Cette action a un effet réel et irréversible et déclenche systématiquement une confirmation de l'utilisateur avant d'être exécutée — fournis `reason` pour qu'il puisse décider vite.

### `execute_cleanup` — `irreversible`

> SECOND TEMPS — met à la CORBEILLE (jamais effacés définitivement, récupérables) les fichiers d'un plan obtenu par plan_cleanup dans ce même échange. N'exécute que ce plan précis : un fichier modifié ou disparu depuis le plan est ignoré. Action à effet réel : déclenche systématiquement une confirmation de l'utilisateur, qui voit le résumé du plan (compte, taille, exemples) avant de décider. Ne prétends JAMAIS avoir supprimé quoi que ce soit sans le résultat réel de cet outil. Fournis `reason` en une phrase.

### `open_url` — `confirm`

> Ouvre une URL dans le navigateur par défaut de l'utilisateur (ex: lancer un site web, un lecteur en ligne). Ne fonctionne que pour http(s) — pas un fichier local (utilise open_path pour ça). Si l'utilisateur nomme un artiste/titre/recherche en plus d'un site ("lance tel artiste sur Deezer", "cherche X sur YouTube"), construis une URL DE RECHERCHE sur ce site (ex: https://www.deezer.com/search/<requête>, https://www.youtube.com/results?search_query=<requête>) plutôt que d'ouvrir seulement la page d'accueil — ça ne suffit pas à lancer la lecture (le navigateur bloque l'autoplay au premier chargement d'une page), dis-le honnêtement plutôt que de prétendre que la musique se lance toute seule. Ne connais AUCUNE playlist ou favori personnel de l'utilisateur ("ma playlist", "ma liste du matin") SAUF si son nom exact figure dans les raccourcis connus injectés dans ton prompt (le cas échéant, cf. prompt.ts::withMediaShortcuts) — utilise alors l'URL fournie telle quelle, jamais une autre. Sans raccourci correspondant et sans URL/nom concret donné par l'utilisateur, dis-le plutôt que d'inventer une recherche approximative. Cette action a un effet réel et déclenche systématiquement une confirmation de l'utilisateur avant d'être exécutée — fournis `reason` pour qu'il puisse décider vite.

### `open_path` — `confirm`

> Ouvre un fichier ou dossier local avec l'application par défaut du système de l'utilisateur (ex: ouvrir un document, un dossier dans l'explorateur de fichiers). Ne crée, ne modifie et ne supprime rien — mais sur certains fichiers (un exécutable, un .desktop), "ouvrir avec l'application par défaut" peut vouloir dire LANCER ce fichier, pas juste l'afficher. Cette action a un effet réel et déclenche systématiquement une confirmation de l'utilisateur avant d'être exécutée — fournis `reason` pour qu'il puisse décider vite.

### `launch_app` — `confirm`

> Lance une application installée sur le PC de l'utilisateur, désignée par son nom (ex: "Rhythmbox", "VLC", "GIMP") — PAS un site web (utilise open_url) ni un fichier/dossier (utilise open_path). Ne fonctionne QUE pour les noms déjà connus de l'utilisateur (table app-shortcuts.json) — si l'appli demandée n'y figure pas, dis-le honnêtement à l'utilisateur plutôt que de deviner un chemin ou une commande. N'invente JAMAIS de chemin de fichier pour lancer une application. Cette action a un effet réel et déclenche systématiquement une confirmation de l'utilisateur avant d'être exécutée — fournis `reason` pour qu'il puisse décider vite.

### `control_media` — `confirm`

> Contrôle la lecture audio/vidéo en cours sur le PC de l'utilisateur (lecteur actif, quel qu'il soit — Spotify, VLC, un onglet de navigateur...). Actions et déclencheurs typiques : play_pause = "pause"/"reprends"/"relance" ; next_track = "piste suivante"/"chanson suivante"/"change de musique"/"change de son" ; previous_track = "piste précédente"/"reviens en arrière" ; volume_up = "monte le son"/"plus fort" ; volume_down = "baisse le son"/"moins fort" ; mute_media = "coupe le son"/"coupe la musique"/"mute" ; unmute_media = "remets le son"/"remets la musique"/"démute". mute_media/unmute_media agissent UNIQUEMENT sur la musique/vidéo/radio détectée en cours de lecture — jamais sur le son du PC dans son ensemble, ta propre voix reste toujours audible après l'un ou l'autre. Ne choisit ni piste ni niveau de volume précis. Cette action a un effet réel et déclenche systématiquement une confirmation de l'utilisateur avant d'être exécutée — fournis `reason` pour qu'il puisse décider vite.

### `open_site_search` — `confirm`

> Ouvre une recherche dans le navigateur par défaut de l'utilisateur sur un site connu (ex: "images" pour une recherche visuelle, "youtube" pour une vidéo, "deezer" pour une musique, "web" pour une recherche générale) — jamais pour une information que TU dois lire/synthétiser toi-même (utilise web_search/read_url pour ça, cf. section ACCÈS WEB). Utilise ce site précisément pour ce que l'utilisateur veut VOIR à l'écran (images, vidéo à lancer, page de résultats à parcourir), pas pour construire ta propre réponse. Fournis uniquement des MOTS de recherche dans `query` — jamais une URL, le site se charge de la construire. Si le site demandé n'est pas dans la liste connue, dis-le honnêtement plutôt que d'utiliser open_url avec une URL devinée. Cette action a un effet réel et déclenche systématiquement une confirmation de l'utilisateur avant d'être exécutée — fournis `reason` pour qu'il puisse décider vite.

### `play_top_result` — `confirm`

> Cherche un titre/une vidéo sur "deezer" ou "youtube" et ouvre DIRECTEMENT le premier résultat trouvé dans le navigateur par défaut de l'utilisateur — pas une page de résultats à parcourir (utilise open_site_search pour ça), l'objectif ici est que la lecture démarre. Sur YouTube la vidéo démarre généralement automatiquement à l'ouverture. Sur Deezer, l'ouverture de la page du titre NE GARANTIT PAS le démarrage automatique de la lecture (dépend d'une session Deezer déjà connectée dans le navigateur de l'utilisateur) — dis-le honnêtement si tu n'as aucune preuve que la musique a effectivement démarré, ne prétends jamais l'inverse. Fournis uniquement des mots de recherche dans `query`, jamais une URL. Cette action a un effet réel et déclenche systématiquement une confirmation de l'utilisateur avant d'être exécutée — fournis `reason` pour qu'il puisse décider vite.

### `delegate_to_provider` — `auto`

> Délègue une tâche plutôt que d'y répondre toi-même. provider="claude" : architecture, raisonnement théorique approfondi, requête ambiguë ou critique, analyser une image/un PDF (cf. attach_path), ou si tu doutes de ta propre capacité — le résultat est la vraie réponse de Claude, à réécrire ensuite avec ta propre voix, jamais un copier-coller brut (Claude ne sait pas qu'il te parle à TOI). provider="redaction" : résumé ou narration longue — le texte retourné EST déjà le livrable final, transmets-le proche du verbatim (pas de réécriture du corps). provider="chatgpt" : l'utilisateur demande ChatGPT nommément, ou veut explicitement un second avis à comparer à celui de Claude sur la même question. provider="deepseek" : question de programmation/debug technique où l'utilisateur demande DeepSeek nommément, ou dont raisonner sur du code est le vrai cœur de la tâche. Pour chatgpt/deepseek comme pour claude, réécris toujours le résultat avec ta propre voix — jamais un copier-coller brut. `save_as` (optionnel, uniquement avec provider="redaction") : nom de fichier si l'utilisateur veut que ce texte soit enregistré — l'application écrit le fichier directement à partir du texte rédigé, n'appelle JAMAIS write_file en plus pour ce même texte ensuite (le retaper toi-même risque de le tronquer/déformer).

### `write_memory` — `auto`

> Écris une information dans TA mémoire durable sur l'utilisateur — UNIQUEMENT si l'utilisateur te le demande littéralement ("écris ça dans ta mémoire") ou fait une déclaration forte et explicite ("souviens-toi de ça", "intègre bien ce point", "c'est essentiel"). N'appelle JAMAIS cet outil de ta propre initiative pour une simple remarque intéressante — un mécanisme séparé, automatique, reste le filet de sécurité pour ça.

### `update_memory` — `auto`

> Corrige une entrée EXISTANTE de ta mémoire durable sur l'utilisateur — quand une nouvelle information contredit ou périme ce que tu avais noté avant, jamais pour une simple nuance. UNIQUEMENT sur demande littérale ("corrige ta mémoire...") ou déclaration forte explicite, même règle que write_memory. `entry_id` doit être copié EXACTEMENT depuis le format "[id:...]" visible dans ta mémoire condensée — jamais inventé ; si tu ne vois pas l'id de l'entrée à corriger, utilise write_memory pour une nouvelle entrée à la place.

### `search_past_conversations` — `auto`

> Cherche dans tes AUTRES conversations avec l'utilisateur — jamais celle-ci, qui est déjà entièrement dans les messages ci-dessus. Utilise cet outil quand l'utilisateur fait référence à un échange antérieur ("comme on disait l'autre jour", "tu te souviens de...", "on avait déjà parlé de...") ET que tu ne trouves pas ce contexte dans les messages ci-dessus — vérifie-les TOUJOURS en premier. Ne confonds jamais avec write_memory/read_url : ceci retrouve une CONVERSATION passée, pas un fait mémorisé ni une page web. Renvoie les fils les plus pertinents avec un extrait — pas le fil entier.

### `recall_provider_answers` — `auto`

> Retrouve le TEXTE INTÉGRAL d'un avis qu'un autre modèle (Claude, ChatGPT, DeepSeek) t'a donné lors d'un échange précédent. Tu ne conserves de ces avis que ce que tu en as toi-même recopié dans ta réponse : si l'utilisateur te demande de recroiser, comparer ou compléter un avis que tu as reçu il y a plusieurs tours, utilise cet outil au lieu de lui redemander ce qui a été dit. Les avis du tour en cours et des tout derniers tours sont déjà dans les messages ci-dessus — vérifie-les TOUJOURS en premier. Ne confond pas avec search_past_conversations, qui retrouve un échange avec FLORENT, pas la réponse d'un autre modèle.

