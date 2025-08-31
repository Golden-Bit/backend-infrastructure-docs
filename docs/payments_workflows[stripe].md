# 1) Checkout (creazione sessione) + Webhook

```mermaid
sequenceDiagram
    autonumber
    participant U as User/App
    participant A as User Auth API
    participant P as Payments API
    participant S as Stripe
    participant W as Webhook Receiver (tuo backend)
    participant DB as Database (facoltativo)

    %% Auth
    U->>A: Username/Password (OIDC/OAuth)
    A-->>U: JWT

    %% Create Checkout
    U->>P: POST /plans/me/checkout (JWT)<br/>plan|price_id, qty, customer_update, idempotency
    P->>S: Create Product/Price (se serve) [Idempotency]
    P->>S: Create Checkout Session (mode=subscription) [Idempotency]
    S-->>P: {session.id, session.url, customer_id}
    P-->>U: {url} (pagina Stripe Checkout)

    %% Payment
    U->>S: Completa pagamento (carta test)
    S-->>W: webhook checkout.session.completed<br/>(+ subscription_id, customer_id)
    Note over W: Verifica firma webhook (whsec_...)<br/>Idempotent handling

    %% Side effects
    W->>P: POST /plans/portal/session (X-API-Key o JWT)
    P->>S: Create Portal Session
    S-->>P: {portal.url}
    P-->>W: {portal.url}
    W->>DB: Append evento / Persist (opz.)
    W-->>U: (facoltativo) notifica applicazione
```

**Note chiave**

* **JWT** sul client → **Payments API**; **Payments API ↔ Stripe** con **Client ID/Secret**.
* Usa **Idempotency-Key** dedicata per *ogni* POST verso Stripe.
* Il **webhook** verifica la firma (`whsec_...`) e può generare subito un **Portal URL**.

---

# 2) Gestione abbonamento via **Billing Portal** (Stripe-hosted)

```mermaid
sequenceDiagram
    autonumber
    participant U as User/App
    participant A as User Auth API
    participant P as Payments API
    participant S as Stripe
    participant W as Webhook Receiver (Backend tuo)
    participant DB as Database/Log (opz.)

    %% Autenticazione utente
    U->>A: Login (username/password o social/OIDC)
    A-->>U: JWT

    %% Richiesta Portal
    U->>P: POST /plans/me/portal/session (JWT)<br/>return_url
    P->>S: Create Billing Portal Session
    S-->>P: {portal.url}
    P-->>U: {portal.url}

    %% Apertura Portal
    U->>S: Apre Portal URL (Stripe hosted)
    Note over S: Gestione abbonamenti<br/>Cambio piano, cancellazione,<br/>aggiornamento PM, storico fatture

    %% Eventi generati
    S-->>W: Webhook (subscription.updated,<br/>customer.updated, invoice.* ecc.)
    W->>DB: Aggiorna stato/entitlements
    W-->>U: (opzionale) Notifica App

```

**Perché conviene**

* Minimizza UI custom e compliance burden (SCA, PCI).
* Ottieni gestione piani, metodi di pagamento, fatture “out-of-the-box”.

---

# 3) Gestione abbonamento via **API** (server-side)

(*upgrade/downgrade/cancel/pause/resume* senza passare dal Portal)

```mermaid
sequenceDiagram
    autonumber
    participant U as User/App
    participant A as User Auth API
    participant P as Payments API
    participant S as Stripe
    participant DB as Database (facoltativo)

    U->>A: Login → JWT
    A-->>U: JWT

    %% Esempi di azioni
    U->>P: POST /plans/me/subscriptions/{id}/cancel (JWT)
    P->>S: subscription.cancel / modify(cancel_at_period_end)
    S-->>P: {subscription}

    U->>P: POST /plans/me/subscriptions/{id}/pause (JWT)
    P->>S: subscription.modify(pause_collection=…)
    S-->>P: {subscription}

    U->>P: POST /plans/me/subscriptions/{id}/resume (JWT)
    P->>S: subscription.resume(...)
    S-->>P: {subscription}

    P-->>U: Esito (JSON) + stato aggiornato
    P->>DB: Persist/Log (opzionale)
```

**Spiegazione rapida dei passaggi**

1. User → login presso User Auth API per ricevere il JWT. 
2. L’utente invoca /plans/me/portal/session con il JWT. 
3. Payments API crea una Portal Session presso Stripe. 
4. Stripe restituisce un portal.url, che la Payments API inoltra all’utente. 
5. L’utente apre il Portal (UI Stripe-hosted) ed esegue azioni (upgrade/downgrade, cancellazione, update metodi di pagamento). 
6. Stripe emette webhook events (subscription.updated, invoice.paid, ecc.). 
7. Il Webhook Receiver li valida e aggiorna il DB o invia notifiche all’app.

**Quando usarlo**

* Hai una UI personalizzata e vuoi un controllo granulare (UX su misura, automazioni, policy).

---

# 4) Rinnovi, Fatture e Pagamenti ricorrenti

```mermaid
sequenceDiagram
    autonumber
    participant S as Stripe
    participant W as Webhook Receiver
    participant P as Payments API
    participant U as User/App
    participant DB as Database (facoltativo)

    S-->>W: invoice.upcoming (pre-promemoria)
    W->>DB: aggiorna stato/notify (opzionale)

    S-->>W: invoice.created → invoice.finalized
    W->>DB: registra bozza/finale (opzionale)

    S-->>W: invoice.payment_succeeded / invoice.paid
    W->>DB: marca come pagata / estende entitlement (opzionale)
    W-->>U: (facoltativo) notifica “rinnovo avvenuto”

    S-->>W: invoice.payment_failed
    W-->>U: (facoltativo) notifica “problema pagamento”
    W->>P: (opzionale) azioni automatizzate (grace period, downgrade programmato)
```

**Tip**

* Gestisci **state machine** lato tuo DB (“active”, “past\_due”, “canceled”, …) sincronizzata con webhook Stripe.

---

# 5) Aggiornamento / Aggiunta **Metodo di Pagamento**

```mermaid
sequenceDiagram
    autonumber
    participant U as User/App
    participant S as Stripe
    participant P as Payments API

    %% Step 1: User crea PM client-side
    U->>S: Crea PaymentMethod via Elements/SDK
    S-->>U: pm_xxx (tokenizzato)

    %% Step 2: User invia pm_xxx al backend
    U->>P: POST /plans/me/payment-methods/attach (JWT, pm_xxx)

    %% Step 3: Payments API interagisce con Stripe
    P->>S: Attach PaymentMethod al Customer
    S-->>P: Conferma attach

    %% Step 4: (opzionale) Set default PM
    P->>S: Subscription.modify (set default PM)
    S-->>P: OK

    %% Step 5: Esito
    P-->>U: Risposta {status, details}

```
**Spiegazione**

1. User/App usa Stripe Elements/SDK per raccogliere i dati carta e genera un PaymentMethod tokenizzato (pm_xxx). 
2. L’utente invia il token pm_xxx al backend tramite la rotta POST /plans/me/payment-methods/attach, autenticandosi con JWT. 
3. La Payments API chiama Stripe per associare il PaymentMethod al Customer corrispondente.
4. (Opzionale) la Payments API imposta il PM come default per una Subscription. 
5. Infine, la Payments API risponde all’utente con l’esito.

**Alternative**

* Far fare tutto dal **Portal** (nessuna UI custom) → vedi Diagramma 2.

---

# 6) Creazione dinamica del **piano** (Product/Price) + Checkout

(*quando non usi price preconfigurati*)

```mermaid
sequenceDiagram
    autonumber
    participant U as User/App (Configurator)
    participant P as Payments API
    participant S as Stripe

    U->>P: POST /plans/me/checkout (JWT)<br/>plan: {name,currency,unit_amount,recurring,...}
    P->>S: product.create (se non esiste) [Idempotency]
    P->>S: price.create (ricorrente) [Idempotency]
    P->>S: checkout.session.create (mode=subscription) [Idempotency]
    S-->>P: {session.url}
    P-->>U: {session.url}
```

**Best practice**

* Genera **Idempotency-Key** per ciascuna chiamata (`…:product.create`, `…:price.create`, `…:checkout.create`).
* Se abiliti `automatic_tax`, assicurati **customer address** valido **o** `customer_update.address='auto'`.

---

## Suggerimenti operativi (riassunto)

* **Scegli Portal** quando vuoi “**move fast**”: meno codice, UX standard, update PM/fatture/upgrade/cancel già pronti.
* **Scegli API pure** quando ti serve **massimo controllo** e UI 100% personalizzata.
* Mantieni **webhook** idempotenti e con **verifica firma**.
* Conserva nel tuo DB lo **stato derivato** (entitlements/licenze), non duplicare i dati fiscali di Stripe.
* In **Test mode**, documenta bene la differenza tra *customer* e *user* nella tua app; usa un mapping stabile (es. claim `sub` del JWT ⇄ `internal_customer_ref` su Stripe).

Se vuoi, posso generarti i **file separati** (uno per diagramma) o comporre un **README** completo con i frammenti Mermaid già divisi e pronti per il rendering.
