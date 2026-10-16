# supply-intake

The **supply domain's intake flow** ([ADR 0015](https://github.com/azure-intelligence-domain/ai-intelligence-domain/blob/main/docs/adr/0015-one-intake-per-domain.md)):
what the supply domain takes in from other domains. A Logic App (Consumption), owned by the
**supply-team**. It receives **OrderConfirmed** and reserves the order's stock, through the gateway, with the
WMS System API; the System API then publishes StockReserved.

| File | What |
|---|---|
| [`workflow.json`](workflow.json) | The flow: switch on the event type, map, call the System API |
| [`asyncapi.yaml`](asyncapi.yaml) | Consumer contract: what it receives (from the Order Process API's released contract) |
| [`infra/main.bicep`](infra/main.bicep) | The Logic App, its runtime identity, and its OAuth policy (the gateway only) |
| [`tests/`](tests/) | The flow against its contract |

```mermaid
sequenceDiagram
    autonumber
    participant EG as evgt-intplat-order
    participant G as apim-intplat-demo
    participant LA as supply-intake (Logic App)
    participant S as WMS System API
    EG->>G: OrderConfirmed (subscription supply-intake, filtered on the event type)
    G->>LA: POST /intake/supply-intake → run (gateway identity, OAuth policy)
    LA->>G: POST /wms/stock-reservations (runtime identity, Correlation-Id)
    G->>S: system-apis product: allow-listed caller
    S-->>LA: 201
    LA-->>G: 200
    Note over EG,S: The System API publishes StockReserved. A failure: 502, Event Grid retries, then dead-letters into the tap.
```

## Mapping

The flow maps the order domain's event onto the WMS System API's contract; nothing else.

```mermaid
flowchart LR
    ev["OrderConfirmed<br/>(order domain contract)"] -- "workflow.json" --> req["/wms/stock-reservations<br/>(WMS System API contract)"]
```

## When a delivery fails

The flow does not retry: if the System API refuses or fails, the run ends in `502 Delivery
Failed`, Event Grid retries the delivery (3 attempts), then dead-letters the event into the tap
storage (`sttapintplatdemo`, container `deadletters`, folder `supply-intake`). The Intelligence domain
sees it there; nobody has to search Logic App run histories.

## Cost

Pay per run, nothing polls: an idle flow costs nothing. A run is about five built-in actions plus
one HTTP call.

## Pipeline

`.github/workflows/deploy.yml`: contract check and flow tests, then on `main` (OIDC as
`id-intplat-supply-intake`): the Logic App, a smoke test (a call without the gateway's token gets `401`),
and a deployment event.
