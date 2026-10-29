# Pluia MCP server

Remote MCP server for French zoning rules (PLU / PLUi). Give it an address or a cadastral parcel in France: it returns the zone, the planning document in force, and the key rules of the zone, each value cited with the page of the official PDF regulation.

- Server URL: `https://pluia.fr/mcp` (Streamable HTTP)
- Service: https://pluia.fr (published by AZERIA IT)
- Page for AI assistants: https://pluia.fr/ia
- Agent guide (plain Markdown): https://pluia.fr/SKILL.md
- MCP registry name: `io.github.AZERIA-IT/pluia`

This repository contains documentation and client configuration examples only. The Pluia service itself is not open source and its code is not published here. The documentation and examples are under the MIT license (see [LICENSE](LICENSE)).

## What it does

Pluia reads the planning regulation (plan local d'urbanisme) that applies to a parcel and gives back the rules with their proof: the page number and a link to the official PDF at that page. A rule is only labelled "verified" when its value was re-read on the cited page; other values carry their own status (official text to read, conditional, undetermined, and so on) so an assistant can say what is certain and what is not.

Typical uses:

- find the parcel behind an address or a cadastral reference;
- get the zone of a parcel and the planning document in force;
- list servitudes, prescriptions and heritage constraints that add to the zone rules;
- read key rules (footprint, height, setbacks, open ground, parking) with their pages;
- check a small project (extension, pool, garage, shed...) rule by rule, from measures you declare;
- quote the text of a page of the regulation.

The tool names, descriptions and messages are in French. Argument names are in English.

## Connect

Two ways to authenticate, both against `https://pluia.fr/mcp`:

1. **OAuth 2.1 with your Pluia account** (no key to copy). The client opens pluia.fr, you sign in, you see the application name and the access requested, then you choose Authorize or Deny. Authorization code flow with PKCE (S256). Dynamic client registration and client metadata documents are supported. Discovery documents:
   - https://pluia.fr/.well-known/oauth-protected-resource
   - https://pluia.fr/.well-known/oauth-authorization-server

   Scopes: `read`, `analyze`, `studio`, `report`, `ask`. You can withdraw an access at any time in your Pluia account (Compte, Applications connectées).
2. **API key**, created in your Pluia account (Compte, Clés d'API), sent on every request as `Authorization: Bearer <key>`. Read it from an environment variable; never commit it.

Both require a Pluia account with a confirmed e-mail address. A free plan exists, with no payment card; quotas depend on your plan. See https://pluia.fr/pricing for what each plan includes.

The server is stateless and answers with JSON (`POST` only, no streaming, no session). It serves MCP revision `2026-07-28` and falls back to `2025-11-25`.

## Tools

`tools/list` only shows the tools your access allows. Tools marked "uses quota" consume one unit of your plan's quota per call. These tools are annotated as not read-only, so most clients ask for your confirmation before calling them. `consulter_quotas` shows what is left.

| Tool | What it does | Main arguments | Scope | Quota |
|---|---|---|---|---|
| `trouver_parcelle` | Finds candidate parcels (IDU, commune, section, number, area, address) | one of: `address`, `idu`, `insee`+`section`+`number` (+`prefix`), `lat`+`lon`; `limit` | read | no |
| `zonage_parcelle` | Zone at the parcel, zones it touches, planning document in force, indexing state | `idu` | read | no |
| `servitudes_parcelle` | Servitudes, prescriptions and heritage constraints (including the need to consult the Architecte des Bâtiments de France), with sources | `idu` | read | no |
| `etat_document` | Whether Pluia can read the regulation of a commune, a document or a dossier, and whether it is up to date | exactly one of `insee`, `document_id`, `dossier_id` | read | no |
| `lister_dossiers` | Your dossiers (parcel analyses), newest first | `idu` (filter), `limit`, `offset` | read | no |
| `analyser_parcelle` | Creates the dossier of a parcel: reads its regulation and extracts the key rules | `idu`, `zone` | analyze | 1 dossier |
| `regles_parcelle` | Key rules of a dossier, each with value, status, condition and proof (page, excerpt, PDF link) | `dossier_id` or `idu`, `include_topics` | read | no |
| `citer_reglement` | Text of one page of the official regulation cited by one of your dossiers, with its PDF link | `document_id`, `page` | read | no |
| `verifier_projet` | Rule-by-rule verdict for a project, from the measures you declare | `dossier_id`, `project_type`, `measures`, `answers` | studio | no |
| `poser_question` | Written answer to a question about the regulation of a dossier, with cited passages | `dossier_id`, `question`, `conversation_id`, `zone` | ask | 1 question |
| `generer_rapport` | PDF report of the dossier (same document as in the Pluia app) | `dossier_id`, `include_pdf` | report | 1 export |
| `consulter_quotas` | Your plan, the scopes of your access, remaining quotas | none | read | no |

`verifier_projet` project types: `extension`, `veranda`, `piscine`, `garage`, `carport`, `annexe`, `pool_house`, `abri_jardin`, `abri`.

Notes:

- `analyser_parcelle` and `generer_rapport` are idempotent for the same arguments on the same UTC day: retrying after a `processing` answer does not charge twice.
- Reading the regulation of a commune for the first time takes about 1 to 3 minutes. The tool waits up to 45 seconds, then answers `processing` or `indexing` with a `retry_after` delay; calling again with the same arguments resumes the same operation.
- The server also exposes resources (`pluia://dossiers/{id}/regles`, `pluia://documents/{id}/pages/{page}`) and one prompt, `verifier_faisabilite`.

## Install by client

Ready-to-copy files are in [`examples/`](examples). Each client's syntax was checked against its documentation (linked below).

| Client | File or steps | Documentation |
|---|---|---|
| Claude (claude.ai and desktop app) | [`examples/claude-custom-connector.md`](examples/claude-custom-connector.md) (custom connector, OAuth). Pluia is also listed in Claude's connector directory (see https://pluia.fr/ia). | https://support.claude.com/en/articles/11175166-get-started-with-custom-connectors-using-remote-mcp |
| Claude Code | `claude mcp add --transport http pluia https://pluia.fr/mcp` then `/mcp` to sign in (OAuth); or with a key: [`examples/claude-code.mcp.json`](examples/claude-code.mcp.json) | https://code.claude.com/docs/en/mcp |
| Cursor | [`examples/cursor.mcp.json`](examples/cursor.mcp.json) (`~/.cursor/mcp.json`) | https://cursor.com/docs/context/mcp |
| VS Code | [`examples/vscode.mcp.json`](examples/vscode.mcp.json) (`.vscode/mcp.json`) | https://code.visualstudio.com/docs/copilot/customization/mcp-servers |
| Codex CLI | [`examples/codex.config.toml`](examples/codex.config.toml) (`~/.codex/config.toml`) | https://developers.openai.com/codex/mcp |

Not covered here: ChatGPT. The ChatGPT listing of Pluia is under review (see https://pluia.fr/ia); no configuration is documented in this repository until it is published.

Timeouts: `poser_question` can take up to about 3 minutes. If your client has a short default tool timeout, raise it (the Claude Code and Codex examples set one).

With a key in a file that is committed to a repository, anyone with read access gets the key. Prefer OAuth, or an environment variable as in the examples.

## Example questions

Once connected, ask in plain language, with an address or a parcel:

- "What is the PLU zone at 10 rue de la Paix, 75002 Paris?"
- "What is the maximum height on this parcel? Give me the page of the regulation."
- "Can I build a 32 m² pool 3 m from the boundary on this parcel?"
- "Is there a servitude or an Architecte des Bâtiments de France review on this plot?"

The assistant chooses the tools. The usual order is `trouver_parcelle`, `zonage_parcelle`, `lister_dossiers` (to reuse an existing dossier), `analyser_parcelle` if needed (uses quota), then `regles_parcelle`, `citer_reglement` and `verifier_projet`.

## Data sources

What the service reads, as documented for these tools:

- Géoportail de l'urbanisme (GPU): zoning, planning documents and their PDF regulations, prescriptions.
- Géoplateforme (IGN) address search, with the Base adresse nationale as fallback: address to parcel.
- API Carto cadastre: parcels and their geometry.

Servitudes and heritage constraints are returned with their source in each answer.

## Limits

- France only. Only planning documents published on the Géoportail de l'urbanisme are read. For a parcel under the national rule (RNU), a communal map (carte communale) or with no published document, no zone rule is given.
- Pluia has to read a regulation before it can quote it. A commune not read yet is reported as such and triggers the reading (minutes); an outdated or unreadable regulation never produces a rule or a "yes".
- Only values re-read on the cited page are marked verified. Others are returned with their status and the page link, to be read.
- When a source does not answer (for example the Géoportail), the answer says so; it is never an absence of rule.
- A parcel can touch several zones; the choice of zone is left to the user. A `nearest_parcel` match is the closest parcel to a point and is uncertain.
- A project verdict relies on the measures you declare, not on a survey. It is indicative.
- Pluia is not a certificat d'urbanisme, not an authorization and not legal advice. It does not replace the urban planning service of the mairie, a lawyer or an urban planner. Confirm important rules with the mairie before filing.
- Coverage by commune and the share of rules that are verified are not stated here; the service reports them per commune and per dossier.
- Only documents of your own dossiers can be quoted with `citer_reglement`.

## Registry

Listed in the MCP registry as `io.github.AZERIA-IT/pluia`: https://registry.modelcontextprotocol.io/v0/servers/io.github.AZERIA-IT%2Fpluia/versions/latest. The [`server.json`](server.json) in this repository is the registry entry with the repository field.

## Contact

contact@pluia.fr · https://pluia.fr

---

# En français

## Ce que fait Pluia

Serveur MCP distant pour les règles d'urbanisme du PLU (ou PLUi) d'une adresse ou d'une parcelle en France. Chaque règle est citée avec la page du règlement PDF officiel et son lien. Seule une valeur relue à la page citée est marquée « Vérifié » ; les autres portent leur statut (texte officiel à lire, sous conditions, non déterminé...).

- URL du serveur : `https://pluia.fr/mcp` (HTTP streamable)
- Service : https://pluia.fr, édité par AZERIA IT
- Page dédiée aux assistants : https://pluia.fr/ia
- Ce dépôt ne contient que de la documentation et des exemples de configuration (licence MIT). Le service Pluia n'est pas open source.

## Connexion

- **OAuth 2.1 avec votre compte Pluia** : le client ouvre pluia.fr, vous vous connectez, vous voyez l'application et les accès demandés, puis vous autorisez ou refusez. Retrait possible à tout moment dans Compte → Applications connectées.
- **Clé d'API** créée dans Compte → Clés d'API, envoyée en `Authorization: Bearer <clé>`. Lisez-la depuis une variable d'environnement, ne la versionnez jamais.
- Un compte Pluia à l'adresse e-mail confirmée est nécessaire. Une offre gratuite existe, sans carte bancaire ; les quotas dépendent de l'offre : https://pluia.fr/pricing

## Outils

12 outils : `trouver_parcelle`, `zonage_parcelle`, `servitudes_parcelle`, `etat_document`, `lister_dossiers`, `analyser_parcelle`, `regles_parcelle`, `citer_reglement`, `verifier_projet`, `poser_question`, `generer_rapport`, `consulter_quotas`. Détail, arguments et portées dans le tableau anglais ci-dessus. Trois outils consomment une unité de quota par appel : `analyser_parcelle` (1 dossier), `poser_question` (1 question), `generer_rapport` (1 export PDF).

## Installation

Voir le tableau « Install by client » et le dossier [`examples/`](examples) : Claude (connecteur personnalisé), Claude Code, Cursor, VS Code, Codex CLI. ChatGPT : non documenté ici, l'inscription est en cours de validation selon https://pluia.fr/ia.

## Exemples de questions

- « Quelle est la zone du PLU au 10 rue de la Paix, 75002 Paris ? »
- « Quelle hauteur maximale sur cette parcelle ? Donne la page du règlement. »
- « Puis-je construire une piscine de 32 m² à 3 m de la limite ? »
- « Y a-t-il une servitude ou un avis de l'Architecte des Bâtiments de France sur ce terrain ? »

## Sources de données

Géoportail de l'urbanisme (zonage, documents d'urbanisme et règlements PDF, prescriptions), Géoplateforme de l'IGN avec la Base adresse nationale en repli (adresse vers parcelle), API Carto cadastre (parcelles).

## Limites

- France uniquement, documents publiés sur le Géoportail de l'urbanisme seulement. RNU, carte communale ou document absent : aucune règle de zone.
- Un règlement non encore lu par Pluia est signalé comme tel, et sa lecture prend quelques minutes. Un règlement périmé ou illisible ne donne jamais de règle ni de « oui ».
- Si une source ne répond pas, la réponse le dit : ce n'est jamais une absence de règle.
- Le verdict d'un projet repose sur vos mesures déclarées ; il est indicatif.
- Pluia n'est pas un certificat d'urbanisme, ni une autorisation, ni un conseil juridique. Confirmez les règles importantes auprès du service urbanisme de la mairie avant un dépôt.

## Contact

contact@pluia.fr · https://pluia.fr
