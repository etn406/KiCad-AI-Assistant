# Plan — Ajout de DeepSeek comme provider LLM

## Objectif

Permettre de sélectionner `deepseek` comme `llm_provider` dans les réglages du
plugin, afin de dialoguer avec l'API DeepSeek (`deepseek-chat`,
`deepseek-reasoner`) sans devoir saisir manuellement une URL d'endpoint custom.

DeepSeek expose une API **compatible OpenAI** (`/chat/completions`, auth
`Authorization: Bearer <key>`), donc l'essentiel de la couche transport existe
déjà. Le travail consiste surtout à **rattacher le provider à l'endpoint et au
modèle par défaut**, et à exposer l'option dans l'UI + la doc.

## État des lieux (constats)

- **Dispatch** : `LLMClient._call_llm` (`kicad_plugin/llm_client.py`) branche sur
  `ollama`, `bedrock`, `anthropic` ; tout le reste retombe sur le chemin
  OpenAI (`_call_openai` / `_stream_openai`). Un provider `deepseek` tomberait
  donc déjà dans le bon chemin — **sauf** pour l'URL de base par défaut.
- **Résolution de l'URL** : chaque méthode fait
  `base = self._settings.llm_base_url or "https://api.openai.com"` (idem
  `_stream_openai`). Il n'existe **aucune table centrale provider → base URL**.
  Sans `llm_base_url`, un provider `deepseek` frapperait OpenAI. **C'est le
  principal point à corriger.**
- **Reasoning déjà géré** : `reasoning_content` (`deepseek-reasoner`) est déjà
  parsé en streaming (`_stream_openai_request`), agrégé en non-streaming
  (`_subprocess_sse_stream`) et préservé dans l'historique
  (`msg_to_save["reasoning_content"]`). Aucune modification nécessaire.
- **UI** : `SettingsDialog._PROVIDERS` et `_DEFAULT_MODELS`
  (`kicad_plugin/ui/settings_dialog.py`) listent les providers et pré-remplissent
  le modèle par défaut.
- **Settings** : `PluginSettings.llm_provider` (défaut `"openai"`), commentaire
  d'énumération `"openai" | "anthropic" | "ollama" | "bedrock"`.
- **Vision** : DeepSeek n'accepte pas d'images → `llm_supports_vision` doit
  rester `False` pour ce provider (pas de plomberie image côté DeepSeek).

## Approche retenue

DeepSeek = provider OpenAI-compatible, avec :
- base URL par défaut : `https://api.deepseek.com` (le suffixe `/v1` est
  optionnel ; `_openai_endpoint_url` produira
  `https://api.deepseek.com/v1/chat/completions`, accepté par DeepSeek) ;
- modèle par défaut : `deepseek-chat` (le plus sûr pour le tool-calling) ;
- vision : non.

Introduire une **table centrale `_PROVIDER_DEFAULT_BASE_URL`** dans
`llm_client.py` et une méthode `_default_base_url(provider)` pour éviter de
dupliquer la logique, puis l'utiliser dans les méthodes `_call_*` / `_stream_*`.

## Changements par fichier

### 1. `kicad_plugin/settings.py`
- Mettre à jour le commentaire d'énumération de `llm_provider` pour inclure
  `"deepseek"`.
- (Optionnel) Ajouter un champ `llm_context_tokens` par défaut adapté à DeepSeek
  — voir « Décisions à confirmer ». Aucun nouveau champ requis sinon : `llm_model`
  et `llm_base_url` suffisent.

### 2. `kicad_plugin/llm_client.py`
- Ajouter une constante module :
  ```python
  _PROVIDER_DEFAULT_BASE_URL = {
      "openai": "https://api.openai.com",
      "deepseek": "https://api.deepseek.com",
      "anthropic": "https://api.anthropic.com",
      "ollama": "http://localhost:11434",
  }
  ```
- Ajouter `LLMClient._default_base_url()` qui renvoie
  `self._settings.llm_base_url` si défini, sinon la table ci-dessus (fallback
  OpenAI).
- Remplacer les littéraux dans :
  - `_call_openai` (`base = self._default_base_url()`),
  - `_stream_openai` (idem),
  - `_call_anthropic` / `_stream_anthropic` (idem, table anthropic),
  - `_call_ollama` / `_stream_ollama` (idem, table ollama).
- Mettre à jour la docstring de classe `LLMClient` (liste des providers
  supportés) pour ajouter `provider="deepseek"`.
- Aucune modification du dispatch `_call_llm` nécessaire : `deepseek` retombe
  dans la branche OpenAI (streaming via `_stream_openai`).
- Vérifier `_is_gemini_endpoint()` : détection URL-based, sans conflit avec
  `api.deepseek.com`.

### 3. `kicad_plugin/ui/settings_dialog.py`
- Ajouter `"deepseek"` à `_PROVIDERS` (ordre : `openai`, `deepseek`,
  `anthropic`, `ollama`, `bedrock` — ou en fin de liste ; à trancher).
- Ajouter `"deepseek": "deepseek-chat"` à `_DEFAULT_MODELS` (logique de
  pré-remplissage existante inchangée).
- Le champ « Model supports vision » reste un checkbox manuel ; pas de
  changement forcé (mais cf. doc : à laisser décoché pour DeepSeek).

### 4. Documentation
- `README.md` : table des réglages → ajouter `deepseek` à `llm_provider` ;
  section provider dédiée (modèles, endpoint, absence de vision).
- `README_CN.md` : idem (version chinoise).
- `docs/configuration.md` : la table providers ne documente aujourd'hui que
  Bedrock ; ajouter une sous-section « DeepSeek » (base URL, modèles,
  tool-calling).
- `docs/plugin/plugin_ux.md` : mettre à jour la liste
  « LLM provider selector (OpenAI / Anthropic / Custom endpoint) ».
- `docs/plugin/architecture.md` : mentionner DeepSeek comme provider
  OpenAI-compatible.

### 5. Tests (`tests/unit/plugin/test_llm_client.py`, `test_settings.py`)
- `test_settings.py` : un test « accepte deepseek comme valeur de
  `llm_provider` » (round-trip save/load).
- `test_llm_client.py` (classe `TestOpenAICompatibleRequests`) :
  - `test_deepseek_default_base_url` → avec `llm_provider="deepseek"` et
    `llm_base_url=""`, `_call_openai` poste sur
    `https://api.deepseek.com/v1/chat/completions` avec header
    `Authorization: Bearer`.
  - `test_deepseek_custom_base_url_overrides` → `llm_base_url` non vide
    gagne toujours.
  - `test_deepseek_streaming_base_url` → même URL côté `_stream_openai`.
  - `test_deepseek_reasoning_preserved` → réutiliser le cas
    `reasoning_content` existant (déjà couvert, mais valider avec
    `llm_provider="deepseek"`).
- (Optionnel UI) test que `_DEFAULT_MODELS["deepseek"] == "deepseek-chat"`.

## Ordre d'exécution suggéré

1. `settings.py` (commentaire + éventuel défaut de contexte).
2. `llm_client.py` (table + `_default_base_url` + refactor des `_call_*`).
3. Tests cœur (`test_llm_client.py`, `test_settings.py`) — valider le transport.
4. `settings_dialog.py` (UI).
5. Documentation (README ×2, docs/).
6. `make test` / `run_tests.py` complet.

## Décisions à confirmer

> **Statut : implémenté.** Décisions retenues : modèle par défaut `deepseek-chat` ;
> `deepseek` ajouté en **fin** de la liste UI ; `llm_context_tokens` laissé global
> (pas de défaut par provider) ; aucune plomberie image (vision non supportée).

- **Modèle par défaut** : `deepseek-chat` (tool-calling fiable) vs
  `deepseek-reasoner` (raisonnement, tool-calling supporté seulement depuis
  DeepSeek-V3.2 et parfois capricieux). Recommandation : `deepseek-chat`.
- **Placement dans la liste UI** : juste après `openai` (regroupement
  OpenAI-compatible) ou en fin de liste.
- **Contexte par défaut** : `llm_context_tokens` global = `128000`. DeepSeek
  `deepseek-chat` accepte 128K (V3.2) mais 64K sur certaines versions ; faut-il
  un défaut par provider via l'UI, ou laisser l'utilisateur ajuster ? (peut être
  hors périmètre de ce changement).
- **Vision** : confirmer qu'on ne fait aucune plomberie image pour DeepSeek
  (décocher/avertir) — c'est la position recommandée.

## Risques / points d'attention

- **Endpoint** : `https://api.deepseek.com/v1/chat/completions` est accepté par
  DeepSeek, mais si un doute subsiste, `_openai_endpoint_url` gère déjà une base
  se terminant par `/openai` ; on peut étendre la résolution si nécessaire.
- **Modèles raisonneurs** : `deepseek-reasoner` peut renvoyer `reasoning_content`
  sans être exposé au client ; déjà stocké et renvoyé — attention à ne pas le
  retransmettre au provider dans un tour ultérieur si l'API le rejette (à
  vérifier ; le client le conserve déjà dans l'historique).
- **Pas de régression** : le refactor de `_default_base_url` doit conserver les
  valeurs par défaut actuelles (OpenAI/in fine pour tout provider inconnu) — les
  tests existants `TestOpenAICompatibleRequests` servent de garde-fou.
