# omnilex/judilibre

Judilibre, the Cour de cassation's open data of the French judicial courts' decisions, for
[omnisense/omnilex](https://github.com/glitchr-omnisense/omnilex), through PISTE, the French
State's API portal. The source follows the API's OpenAPI description (`JUDILIBRE-public.json`,
OpenAPI 3.0.2, version 1.2.5) and calls:

| Operation | Request |
|---|---|
| `search()` with a `text` or a `number` | `GET /search` |
| `search()` without a text, `recent()` | `GET /export` |
| `decision()`, `citations()` | `GET /decision?id=` |
| `taxonomy($id, $context)` | `GET /taxonomy` |
| `changes($since, $cursor)` | `GET /transactionalhistory` |
| token | `POST https://oauth.piste.gouv.fr/api/oauth/token`, `grant_type=client_credentials`, `scope=openid` |

on `https://api.piste.gouv.fr/cassation/judilibre/v1.0` (`JudilibreSource::BASE_URI`), or
`https://sandbox-api.piste.gouv.fr/cassation/judilibre/v1.0` and
`https://sandbox-oauth.piste.gouv.fr/api/oauth/token` with `sandbox: true`.

Documentation read on 2026-10-04: the OpenAPI description in the Cour de cassation's repository
(<https://github.com/cour-de-cassation/judilibre-search/blob/master/public/JUDILIBRE-public.json>),
the repository's README (<https://github.com/cour-de-cassation/judilibre-search>), data.gouv.fr's
record of the service (<https://www.data.gouv.fr/dataservices/api-judilibre/>).

## Installation

```sh
composer require omnilex/judilibre
```

Requires PHP 8.2, `omnisense/omnilex` (1.x) and `symfony/http-client` (6.4, 7 or 8). In a Symfony
application with `OmnilexBundle`, the factory is registered as soon as the package is installed,
with the application's HTTP client and the token store over `cache.app`.

## Credentials

An account on <https://piste.gouv.fr> (free), an application there with the Judilibre API
subscribed and its terms of use accepted, and either:

- the application's **OAuth client id and secret** (`client_id`, `client_secret`): a token is
  asked with the client-credentials flow and sent as `Authorization: Bearer` ([the core's
  authentication](https://github.com/glitchr-omnisense/omnilex/blob/1.x/docs/authentication.md));
- or its **API key** (`key_id`), sent as the `KeyId` header, the security scheme the OpenAPI
  description declares.

The sandbox has its own application and credentials. Keep them in `.env.local` or the secrets
vault.

## Options

| Option | Type | Default | Required | |
|---|---|---|---|---|
| `client_id`, `client_secret` | string | `null` | these two, or `key_id` | OAuth through PISTE |
| `key_id` | string | `null` | | the API key; when set, the OAuth pair is not used |
| `sandbox` | bool | `false` | no | PISTE's sandbox, with the sandbox application's credentials |
| `recent` | `update` or `creation` | `update` | no | the `date_type` `recent()` reads: `update` (created or updated in the base) or `creation`; another value is an `InvalidArgumentException` |
| `resolve_references` | bool | `false` | no | `resolve_references=true` on every call: labels in place of keys in the answers |
| `throttle` | float | `0.5` | no | seconds between two calls (the quotas are per application, on the portal) |
| `base_uri` | string | production's or the sandbox's | no | |
| `token_uri` | string | PISTE's or the sandbox's | no | |
| `user_agent` | string | `omnilex/1.x (+https://github.com/glitchr-omnisense/omnilex)` | no | |

Neither `key_id` nor both OAuth options: an `InvalidConfigException` when the source is built
(`Registry::usable()` leaves it out). Without an HTTP client, the factory makes one with a
30-second timeout.

> [!WARNING]
> Known defect: `sandbox` and `resolve_references` are read with PHP's `(bool)`: the string
> `'false'` is true. And a `recent` other than its two values is an `InvalidArgumentException`,
> not an `InvalidConfigException`, which `Registry::usable()` does not skip.

## Capabilities

`capabilities()`: holds decisions; reads no other identifier than its own; criteria `text`,
`number`, `kind`, `jurisdictions`, `from`, `to`, `subjects`, `types`; jurisdictions `FR`; 50 per
page; pseudonymised.

| Operation | Supported |
|---|---|
| `search()` | yes |
| `decision()` | yes, by Judilibre's own identifier (24 hexadecimal characters) |
| `citations()` | yes, from the decision |
| `recent()` | yes |
| `text()`, `article()` | no (the source does not implement them) |

Beyond the contract: `taxonomy(string $id, ?string $context = null): array<string, string>` and
`changes(DateTimeInterface $since, ?string $cursor = null, int $limit = 500): array`.

## Behaviour

**Search with a text or a number.** `GET /search` with:

| `Query` | Parameter |
|---|---|
| `text` | `query`, `operator=and` (every word); `"..."`: the expression, `operator=exact` |
| `number` | `query`, `operator=exact`; a pourvoi normalised (`n° C 17-18.194` → `17-18.194`); not along with a `text` (`NotSupportedException`) |
| `jurisdictions` | a kind of court (`cc`, `ca`, `tj`, `tcom`, `cph`) → `jurisdiction`; anything else is a seat (`ca_paris`, `tj33063`) → `location` |
| `types` | `type`: `arret`, `qpc`, `ordonnance`, `saisie`... (`taxonomy('type')`) |
| `subjects` | `theme`: the Cour de cassation's themes (`taxonomy('theme')`) |
| `from`, `to` | `date_start`, `date_end`: the decision's date |
| `sort` | `relevance` → `sort=scorepub` (relevance and publication level); `newest`, `oldest` → `sort=date`, `order=desc` or `asc` |
| `limit`, `cursor` | `page_size` (1 to 50), `page` (the cursor, from 0); `Results::$next` is the next page when the API gives a `next_page`; `total` |

`kind` other than `DECISION` is refused; `title` and `at` are not criteria of the source.

**Search without a text.** `/search` answers nothing to an empty query: the filters go to `GET
/export` with `date_type=creation` when a date is set, `order` by the sort, `batch_size` (1 to
1000), `batch` (the cursor), `abridged=true`.

**What is new.** `recent($since)`: `GET /export` with `date_start` = that day, `date_type` = the
`recent` option, `order=desc`, the query's `jurisdictions`, `types`, `subjects`; a `text` or a
`number` is refused (search for them instead).

**A reference.** `id`, `title` (`<court>, <chamber>, <d/m/Y>, n° <number>`), the ECLI and
numéros de pourvoi, the date, `type`, `number`, the court, a `summary` (the summary, else the
first three highlights), `url` `null`.

**A decision.** `GET /decision?id=`; an ECLI, a pourvoi or any other identifier is a
`NotSupportedException` (search with `Query::$number` first, then read the result):

```php
$found = $judilibre->search(new Query(number: '17-18.194'));
$decision = $judilibre->decision($found->items[0]->id);
```

The `Decision` holds every case number (`number`, `numbers`), the ECLI and the pourvois (only the
Cour de cassation's numbers are pourvois; the courts below number otherwise), the court (`Cour de
cassation` or `Cour d'appel` for their keys, the key as `code`; the chamber, formation and seat),
the date, `type`, `solution` (`solution_alt` when the solution is `other`), `publication` (the
levels joined by commas), `summary`, `subjects` (the themes), the full text as `content` (`html`
`null`), `pseudonymised: true`, `updatedOn` (`update_date`), `language: fr`, `url` `null`; and in
`raw` what the models do not carry - the `zones` of the text (introduction, exposé du litige,
moyens, motivations, dispositif, annexes: character offsets), `nac`, `files`, `timeline`,
`titlesAndSummaries` - the text itself removed.

**Links.** `citations()` reads the decision: each `visa` (a text applied, `APPLIES`, its Légifrance
address parsed into an identifier when it has one), `contested` (`CONTESTS`), `forward`
(`FOLLOWED_BY`), each `rapprochement` (`RELATED`, its description as the note); sliced to `$limit`.

**Taxonomy.** `taxonomy('type')`, `taxonomy('theme')`, `taxonomy('jurisdiction')`,
`taxonomy('chamber', 'cc')`...: the keys a list takes, with their labels - what
`Query::$jurisdictions`, `$types` and `$subjects` take.

**Changes.** `changes($since)` reads `/transactionalhistory` from that moment (UTC, `page_size` 10
to 500): `transactions` (each `id`, `action`, `date`), `next` (a query string the API hands for
the next page, valid for a minute) and `total`. A decision withdrawn must be dropped by whoever
kept a copy:

```php
$changes = $judilibre->changes($lastSync);
foreach ($changes['transactions'] as $transaction) {
    match ($transaction['action']) {
        'deleted' => $store->delete($transaction['id']),          // withdrawn: the copy must go
        default => $store->save($judilibre->decision($transaction['id'])),
    };
}
// $changes['next']: the next page, $judilibre->changes($lastSync, $changes['next'])
```

**Pseudonymisation.** The decisions are pseudonymised by the Cour de cassation before they are
published. They are given as they are: nothing here tries to put a name back on a person, by
cross-reference or otherwise, and an application that shows them must not either (the re-use
conditions forbid it).

## Example

```php
<?php

require __DIR__.'/vendor/autoload.php';

use Omnilex\Judilibre\JudilibreSourceFactory;
use Omnilex\Model\Query;
use Symfony\Component\HttpClient\HttpClient;

$judilibre = (new JudilibreSourceFactory(HttpClient::create()))->create([
    'client_id' => getenv('PISTE_CLIENT_ID'),
    'client_secret' => getenv('PISTE_CLIENT_SECRET'),
]);

$results = $judilibre->search(new Query(text: 'expropriation', jurisdictions: ['cc'], from: '2020-01-01', sort: Query::NEWEST, limit: 10));
foreach ($results as $reference) {
    echo $reference->title, "\n";
}
$decision = $judilibre->decision($results->items[0]->id);
echo $decision->ecli(), ' ', $decision->solution, "\n";
foreach ($decision->citations as $citation) {
    echo '  ', $citation->relation->value, ' ', $citation->target->title, "\n";
}
print_r($judilibre->taxonomy('publication'));
```

Not run: no PISTE credentials. In Symfony:

```yaml
omnilex:
    sources:
        judilibre:
            factory: judilibre
            options:
                client_id: '%env(default::PISTE_CLIENT_ID)%'
                client_secret: '%env(default::PISTE_CLIENT_SECRET)%'
                sandbox: '%env(bool:default::PISTE_SANDBOX)%'
```

From the core's harness, with the credentials in `docker/.env`:
`docker compose run --rm omnilex search judilibre expropriation --jurisdiction cc --from 2020-01-01`.

## Errors

| Case | Result |
|---|---|
| an unknown decision (404) | `null` (`citations()`: `[]`) |
| an ECLI, a pourvoi, any identifier but Judilibre's | `NotSupportedException` |
| `kind` other than `DECISION`, `title`, `at`, a number with a text, a text in `recent()` | `NotSupportedException` |
| credentials refused (the token server, a 401 twice, a 403, a 400 with `WWW-Authenticate`) | `AuthenticationException` |
| a 429, a quota spent | `RateLimitedException` |
| a 5xx, a 423, a timeout | `UnavailableException` |
| another 4xx | `ProviderException` |

## Verified, and not

Tested offline: `Tests/JudilibreSourceTest.php` (14 tests), on `MockHttpClient`, with the examples
of the OpenAPI description as fixtures (a search for "expropriation", the decision 17-18.194 of 20
December 2018, an export batch, the publication taxonomy, the transactional history);
`decision-links.json` is that decision with links composed after the documented schemas. The
search signed through PISTE, the next page, a case number and an exact expression, the filters by
date without a text, the criteria refused, a decision whole and pseudonymised, its links, its own
identifiers only, what is new, the taxonomy, the changes, the sandbox and the API key, refused or
slowed never "not found".

Not verified: **no call was made to Judilibre** - no PISTE credentials. In particular:

- the OpenAPI description declares the `KeyId` header as the API's security scheme; PISTE's
  gateway answers an unsigned call with `WWW-Authenticate: Bearer` (seen on 2026-10-04). Both ways
  of signing are offered; neither was tried with real credentials;
- the action names of `/transactionalhistory` other than `created` (the example's);
- `visa`, `contested`, `forward` and `rapprochements` are read after their documented schemas
  (`textLink`, `decisionLink`): the description's example decision has none;
- with `resolve_references`, the API puts labels where the keys were: `Court::$code` is then
  `null`;
- what `date_type=creation` filters on for a search without a text;
- the quotas "will be on the PISTE portal": not public. The description limits `/export` to 10
  per batch and 1 000 in all for an unauthenticated connection, 1 000 per batch otherwise.

`GET /scan`, `GET /stats` and `GET /healthcheck` are documented and not mapped.

## Responsibility

An independent, unofficial client, not affiliated with the Cour de cassation or the PISTE
platform. Whoever runs it complies with the terms of PISTE and of the Judilibre API
([piste.gouv.fr/cgu](https://piste.gouv.fr/cgu)) and with the re-use conditions of the decisions -
never re-identifying a person: see the README's Responsibility section.
