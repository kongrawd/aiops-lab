# Evidence — what this runbook rests on, and what it does not

Every step traces to published documentation. A claim resting on inference is reason to leave it out rather than to label it, and this revision removes several that an earlier draft labelled instead.

`metadata.verification` is `documented`: the procedure has not been run against a domain with a known insight.

## Basis

- **The `Insight` type reference** for every field the procedure reads and for the values each can hold: `Priority` as `CRITICAL | HIGH | MEDIUM | LOW`, `Status` as `ACTIVE | RESOLVED | DISMISSED`, `Type` as `EVENT | RECOMMENDATION`, `InsightId` as a fixed-length 36-character hyphenated identifier, and `DisplayName`, `IsExperimental`, `CreationTime` and `UpdateTime`. **This is the page an earlier draft failed to read**, and the reason it routed on the wrong field.
- **The `ListInsights` reference** for the request shape — a required entity of a type and a value, `MaxResults` valid from 1 to 500, `SortOrder` of `ASC` or `DESC` — and for the response carrying `Insights[]` and `NextToken`. Both operations are documented as reads that use `POST`.
- **The `DescribeInsightDetails` reference** for the request and for the response being a flat `Fields[]` of `Name`, `Type` and `Value`, with no per-insight enumeration of the names. That is why the procedure iterates it and takes figures from the listing instead.
- **The Cluster Insights page** for the three surfaces and their floors, and for the regional-availability statement.
- **The Insights Catalog** as the authority for the figure each insight is generated against. **This runbook deliberately carries none of those figures** — they live in the catalog and move there, so the procedure sends the reader to read them rather than reproducing them.
- **The CAT indices reference** for `expand_wildcards` being a parameter of that API, and the index-name-syntax reference for `all` including hidden indexes. The shard arithmetic `pri × (1 + rep)` uses columns documented on the same page.
- **Code analysis of the published Security Analytics plugin** for every plugin index family being created with `index.hidden`, which is what makes the dot-prefix classification in step 2 meaningful.
- **Engagement observation** for the failure this runbook prevents: on a live domain running detectors, a shard-count finding was attributed to the operator's own sharding, a re-sharding recommendation was made on that basis, and a later read showed the large majority of shards belonging to hidden plugin indexes. The operator disputed the attribution and was right. **Not vendor-documented**, and no figures from it are reproduced here.

## Removed in this revision rather than labelled

An earlier draft shipped with defects that an adversarial pass against the published references found. They are recorded so they are not re-introduced.

- **Routing on `Type`.** `Type` holds `EVENT` or `RECOMMENDATION` and can never hold an insight family name, so the entry condition could never match and the does-not-apply clause was always true. The family name is in `DisplayName`.
- **A warning that `Priority` and `Status` have no documented value list.** They are both enumerated on the `Insight` type page. The warning was the inverse of the documentation, and it was the most confidently worded sentence in the draft.
- **A two-listing shard diff using `_cat/shards?expand_wildcards=all`.** `expand_wildcards` is not a documented parameter of that API, and an unrecognised parameter is rejected. The figure derived from it was described as the number the runbook existed to produce. Step 2 now uses a single `_cat/indices` listing, where the parameter is documented, and classifies by index name.
- **A pre-flight warning that a pattern without `expand_wildcards=all` returns nothing.** The engine matches a dot-prefixed hidden index against a dot-prefixed wildcard pattern regardless, so the warning did not apply to the patterns in use and the positive control built on it did not prove what it claimed. Step 2's control is now the presence of dot-prefixed rows, which is directly observable.
- **Four attribution branches that depended on whether the insight's own count includes hidden indexes.** That question is undocumented and the draft's own evidence file said not to assert it. The procedure now classifies and reports both populations, which is correct whatever the insight counted — the question is no longer load-bearing and has been removed rather than carried as an open item.
- **`GET /_index_template` as the discriminator for an index-settings insight.** The catalog's condition concerns a setting present on an index; a template read cannot establish it in either direction.
- **A claim that the main-response version override also affects the `_cat/nodes` version column.** The setting is consumed only on the main response.
- **Two symptoms the draft's own evidence file described as reasoned rather than observed**, sitting in a table whose other rows were observed, with nothing marking the difference.
- **Illustrative percentage splits** inside the procedure. Invented figures read identically to measured ones.

## Not established

- **Whether the exact IAM action names match the operation names.** The pre-flight says to confirm them against the service-authorization reference rather than asserting them.
- **Whether an insight can be suppressed.** `DISMISSED` exists as a status, so the state exists, but no documented operation sets it. Nothing in the procedure depends on it.
- **Which regions the feature covers.** The catalog states a list and that it is expanding, so the procedure points at the catalog rather than reproducing the list.

## To raise the level

Run the procedure against a domain with an open shard-count insight. Record the literal `DisplayName`, `Priority` and `Status` values and the `Fields[].Name` set, which closes the two documentation gaps above, and compare the insight's stated figure against the two population sums — which would settle whether the insight counts hidden indexes, the question this revision removed rather than answered. Then set `verification: replicated` and record the version and region tested.
