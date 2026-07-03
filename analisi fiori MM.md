siamo in un contesto SAP BTP. Questo progetto descrive una necessità da risolvere tramite un'architettura Sap Fiori Freestyle su BTP, sviluppata tramite BAS, che comunichi tramite destination-cloud connector all'ambiente S4 on-premise contenente logiche di back end e persistenza. Dovrebbero essere previste all'atto pratico 2 app Fiori Freestyle da sviluppare, + una app Fiori Standard da esporre, ed eventualmente estendere, qualora necessario. Analizza il requisito, ed elabora una roadmap relativa allo sviluppo della parte Fiori Freestyle dividendo in task, considerato che lo sviluppatore in questione è un esperto in materia, con accesso a strumenti di IA atti alla programmazione.

Ho ricostruito il perimetro come due app custom UI5/Fiori su BTP più pubblicazione della standard F4839, con backend S/4 on-prem via Destination + Cloud Connector. Mi concentro sulla roadmap Freestyle e segnalo le dipendenze backend che bloccano o abilitano i task UI.

Ragionato per 26s

**Contesto tecnico ricostruito**

Siamo su **SAP BTP** con app Fiori Freestyle sviluppate in **BAS**, deployate su HTML5 Apps Repository / Launchpad o Work Zone, integrate con **S/4HANA on-premise** tramite **Destination + Cloud Connector**.  
Il requisito prevede gestione budget per CdC/esercizio, controllo budget in creazione reservation tramite BADI backend, workflow approvativo UO/DG e uso della Fiori standard **F4839** per creare le reservation.

**Soluzione principale**

Prevederei **2 app Fiori Freestyle custom**:

- **App Gestione Budget**
  - Gestione budget DG per esercizio e centro di costo.
  - Distribuzione budget da DG verso UO.
  - Visualizzazione importo assegnato, distribuito, consumato e residuo.
  - Campi audit read-only.
- **App Approvazione Reservation**
  - Lista reservation da approvare per utente corrente.
  - Dettaglio reservation, materiali, quantità, importi stimati/consumati.
  - Azioni approva/rifiuta I livello e II livello.
  - Inserimento motivo rifiuto.
  - Aggiornamento stato su tabella custom backend.
  - Chiamata backend che a sua volta usa BAPI_RESERVATION_CHANGE, BAPI_TRANSACTION_COMMIT.

La Fiori standard **F4839** resta l'app di creazione reservation. La logica di controllo budget, blocco sforamento e gestione flag XWAOK/MOVEMENT deve rimanere backend, non UI.

**Roadmap Fiori Freestyle**

**Fase 0 - Analisi tecnica e contratti backend**

**Obiettivo:** bloccare il contratto OData prima dello sviluppo UI.

**Task**

| **Task**                                        | **Output**                                                              |
| ----------------------------------------------- | ----------------------------------------------------------------------- |
| Analisi funzionale requisito budget/reservation | matrice funzionalità UI/backend                                         |
| Definizione EntitySet OData per budget          | contratto BudgetDGSet, BudgetUOSet                                      |
| Definizione EntitySet OData per approvazioni    | contratto ReservationApprovalSet, ReservationItemSet                    |
| Definizione Function Import / Action backend    | ApproveReservation, RejectReservation, DistributeBudget, CopyBudgetYear |
| Definizione autorizzazioni                      | ruoli budget admin, approvatore UO, approvatore DG                      |
| Definizione error handling standard             | messaggi Gateway normalizzati per UI5                                   |

**Contratto OData consigliato**

Per App Budget:

GET /sap/opu/odata/sap/ZGW_BUDGET_SRV/BudgetDGSet?\$filter=Gjahr eq '2026'  
GET /sap/opu/odata/sap/ZGW_BUDGET_SRV/BudgetUOSet?\$filter=Gjahr eq '2026' and KostlDg eq '...'  
POST /sap/opu/odata/sap/ZGW_BUDGET_SRV/BudgetDGSet  
PUT /sap/opu/odata/sap/ZGW_BUDGET_SRV/BudgetDGSet(Gjahr='2026',KostlDg='...')  
POST /sap/opu/odata/sap/ZGW_BUDGET_SRV/DistributeBudget

Per App Approvazione:

GET /sap/opu/odata/sap/ZGW_RES_APPROVAL_SRV/ReservationApprovalSet?\$filter=Status eq 'PENDING'  
GET /sap/opu/odata/sap/ZGW_RES_APPROVAL_SRV/ReservationApprovalSet('0001234567')?\$expand=Items  
POST /sap/opu/odata/sap/ZGW_RES_APPROVAL_SRV/ApproveReservation  
POST /sap/opu/odata/sap/ZGW_RES_APPROVAL_SRV/RejectReservation

**Fase 1 - Setup progetto BTP/BAS**

**Obiettivo:** creare struttura enterprise pulita per le due app.

**Task**

| **Task**                                         | **Output**                               |
| ------------------------------------------------ | ---------------------------------------- |
| Creazione progetto MTA o due moduli UI5 separati | mta.yaml, app router, html5 repo         |
| Configurazione destination BTP verso S/4         | destination HTTP verso Cloud Connector   |
| Configurazione xs-app.json                       | routing /sap/opu/odata verso destination |
| Setup modelli OData V2 nel manifest.json         | model principale backend                 |
| Setup modelli JSON locali                        | viewModel, filterModel, appStateModel    |
| Setup formatter centralizzati                    | model/formatter.js                       |
| Setup error handler comune                       | util/ErrorHandler.js                     |
| Setup routing UI5                                | list/detail/object pages                 |

**Struttura consigliata**

/app-budget  
/webapp  
/controller  
/view  
/fragment  
/model  
formatter.js  
/util  
ErrorHandler.js  
manifest.json  
Component.js  
<br/>/app-reservation-approval  
/webapp  
/controller  
/view  
/fragment  
/model  
formatter.js  
/util  
ErrorHandler.js  
manifest.json  
Component.js

**App 1 - Gestione Budget**

**Fase 2 - Sviluppo App Gestione Budget**

**Obiettivo funzionale**

Consentire a utenti autorizzati di gestire il budget per:

- esercizio;
- centro di costo DG;
- centro di costo UO;
- importo assegnato;
- importo distribuito;
- importo consumato;
- residuo disponibile.

**UX consigliata**

**Pagina 1 - Lista Budget DG**

Controlli consigliati:

- sap.m.DynamicPage
- sap.ui.comp.smartfilterbar.SmartFilterBar, se il servizio espone metadata completi;
- oppure sap.m.FilterBar custom;
- sap.m.Table o sap.ui.table.Table se molti record;
- azioni: crea, modifica, elimina se ammesso, copia da esercizio precedente.

**Pagina 2 - Dettaglio Budget DG**

Contiene:

- dati testata DG;
- importo assegnato;
- importo distribuito read-only;
- residuo DG;
- tabella UO collegate;
- azione distribuisci budget;
- validazione: somma UO non superiore al budget DG.

**Pagina 3 / Dialog - Assegnazione Budget UO**

Campi:

Esercizio  
Centro di costo DG  
Centro di costo UO  
Importo assegnato

Campi read-only:

Importo consumato  
Utente assegnazione  
Data assegnazione  
Utente modifica  
Data modifica

**Task tecnici App Budget**

| **ID** | **Task**                             | **Note**                                |
| ------ | ------------------------------------ | --------------------------------------- |
| BUD-01 | Creazione shell UI5 e routing        | ListBudget, DetailBudget                |
| BUD-02 | Implementazione filtri esercizio/CdC | default esercizio corrente              |
| BUD-03 | Binding lista Budget DG              | OData V2 list binding                   |
| BUD-04 | Navigazione a dettaglio Budget DG    | key composta Gjahr, KostlDg             |
| BUD-05 | Binding tabella UO                   | navigation property o secondo EntitySet |
| BUD-06 | Dialog creazione/modifica Budget DG  | validazioni client leggere              |
| BUD-07 | Dialog assegnazione Budget UO        | controllo importi lato UI + backend     |
| BUD-08 | Gestione campi audit read-only       | mai editabili da UI                     |
| BUD-09 | Formatter importi/residui/stati      | centralizzato                           |
| BUD-10 | Gestione messaggi backend            | MessageBox, MessagePopover              |
| BUD-11 | Busy indicator mirato                | su table/action, non globale sempre     |
| BUD-12 | Test OData CRUD                      | BAS + browser + Gateway Client          |
| BUD-13 | Deploy su BTP                        | HTML5 App Repo                          |
| BUD-14 | Tile Launchpad/Work Zone             | semantic object/action                  |

**Regole UI da rispettare**

La UI può fare controlli preventivi, ma il controllo vero deve restare backend.

Esempio:

Budget DG assegnato: 100.000  
Budget UO già distribuito: 80.000  
Nuova assegnazione UO: 25.000  
<br/>UI: warning/blocco preventivo  
Backend: blocco definitivo

Non affiderei mai la consistenza del budget solo al frontend.

**App 2 - Approvazione Reservation**

**Fase 3 - Sviluppo App Approvazione Reservation**

**Obiettivo funzionale**

Gestire approvazione e rifiuto reservation su due livelli:

- approvatore UO;
- approvatore DG.

Se UO rifiuta, il processo termina.  
Se DG rifiuta, il processo termina.  
Se DG approva, backend abilita la reservation impostando il flag movimento sulle posizioni.

**UX consigliata**

**Pagina 1 - Worklist approvazioni**

Filtri:

Esercizio  
Centro di costo  
Stato  
Livello approvazione  
Numero reservation  
Data creazione

Colonne:

Reservation  
Centro di costo  
Data richiesta  
Richiedente  
Importo stimato  
Livello corrente  
Stato  
Approvatore previsto

Azioni:

Approva  
Rifiuta  
Apri dettaglio

**Pagina 2 - Dettaglio Reservation**

Sezioni:

Dati testata  
Dati centro di costo  
Stato approvativo  
Materiali / items  
Importo totale reservation  
Budget disponibile  
Storico approvazioni

Tabella items:

Materiale  
Descrizione  
Divisione  
Magazzino  
Quantità  
Prezzo medio mobile  
Valore stimato  
Flag cancellazione  
Flag movimento

Azioni:

Approva reservation  
Rifiuta reservation

Il rifiuto deve aprire un dialog obbligatorio per il motivo.

**Task tecnici App Approvazione**

| **ID** | **Task**                               | **Note**                            |
| ------ | -------------------------------------- | ----------------------------------- |
| APR-01 | Creazione progetto UI5 Freestyle       | app separata                        |
| APR-02 | Setup OData model approvazioni         | ZGW_RES_APPROVAL_SRV                |
| APR-03 | Implementazione worklist               | pending by current user             |
| APR-04 | Filtri stato/livello/CdC               | con sap.ui.model.Filter             |
| APR-05 | Detail page con \$expand=Items         | evitare chiamate multiple inutili   |
| APR-06 | Formatter stato approvativo            | colori semantic UI5                 |
| APR-07 | Azione Approva I livello               | Function Import backend             |
| APR-08 | Azione Approva II livello              | Function Import backend             |
| APR-09 | Dialog Rifiuto con motivo obbligatorio | max 50 char da requisito            |
| APR-10 | Azione Rifiuta                         | backend imposta DELETE_IND          |
| APR-11 | Refresh lista dopo azione              | no stale data                       |
| APR-12 | Gestione messaggi BAPI                 | mostrare messaggi SAP leggibili     |
| APR-13 | Autorizzazioni UI                      | visibilità azioni per ruolo/livello |
| APR-14 | Deep link a reservation                | utile dalla mail                    |
| APR-15 | Test scenari UO/DG                     | approve/reject/end-to-end           |
| APR-16 | Deploy BTP e tile                      | semantic object/action              |

**Deep link da mail**

La mail inviata dal backend dovrebbe puntare direttamente alla app custom di approvazione:

# ReservationApproval-display?Reservation=0001234567

La app deve gestire il route pattern:

display/{Reservation}

Così l'approvatore apre direttamente il dettaglio della reservation da approvare.

**Fase 4 - Integrazione con Fiori Standard F4839**

**Obiettivo**

Esporre la Fiori standard **F4839** su Launchpad/Work Zone e integrarla nel processo.

**Task**

| **Task**                                            | **Output**                        |
| --------------------------------------------------- | --------------------------------- |
| Attivazione app standard in S/4                     | cataloghi, servizi OData, ICF     |
| Esposizione su BTP Launchpad/Work Zone              | tile standard                     |
| Verifica accesso via destination                    | routing corretto                  |
| Test creazione reservation movimento 201            | caso positivo e sforamento budget |
| Verifica BADI MB_RESERVATION_BADI                   | blocco budget backend             |
| Verifica reservation creata senza movimento ammesso | prerequisito approvazione         |
| Eventuale Adaptation Project                        | solo se serve modifica UX         |

**Nota importante**

Io eviterei di estendere F4839 se non strettamente necessario.

Il motivo è semplice: il blocco budget, il flag movimento ammesso e la coerenza reservation devono essere garantiti da S/4. La standard app deve solo creare la reservation; il controllo vero avviene via BADI.

Estenderei F4839 solo se serve:

\- aggiungere messaggi informativi specifici;  
\- nascondere campi non rilevanti;  
\- precompilare valori;  
\- aggiungere link verso app budget o app approvazione;  
\- adattare label/testi per Regione Lombardia.

**Fase 5 - Sicurezza, ruoli e autorizzazioni**

**Ruoli applicativi consigliati**

| **Ruolo**            | **Accesso**                      |
| -------------------- | -------------------------------- |
| BudgetAdmin          | crea/modifica budget DG e UO     |
| BudgetViewer         | sola lettura budget              |
| ApproverUO           | approva/rifiuta I livello        |
| ApproverDG           | approva/rifiuta II livello       |
| ReservationRequester | usa F4839                        |
| AdminConfig          | mantiene tabella approvatori CdC |

**Task sicurezza**

| **Task**                      | **Note**                                 |
| ----------------------------- | ---------------------------------------- |
| Definire role collection BTP  | Launchpad/Work Zone visibility           |
| Mappare utenti BTP/SAP        | principal propagation o technical user   |
| Verificare autorizzazioni S/4 | oggetti autorizzativi su CdC/reservation |
| Proteggere action backend     | non solo UI hiding                       |
| Limitare dati per utente      | filtro backend su approvatore corrente   |
| Audit log                     | modifiche budget e approvazioni          |

**Fase 6 - Test end-to-end**

**Scenari minimi da testare**

| **Scenario**                                | **Atteso**                            |
| ------------------------------------------- | ------------------------------------- |
| Creazione budget DG                         | record creato con audit               |
| Distribuzione budget UO valida              | importo distribuito aggiornato        |
| Distribuzione budget superiore al DG        | errore bloccante                      |
| Creazione reservation sotto budget da F4839 | reservation creata                    |
| Creazione reservation sopra budget          | messaggio bloccante                   |
| Reservation creata                          | non movimentabile prima approvazione  |
| Approvazione UO                             | passa a DG                            |
| Rifiuto UO                                  | processo chiuso, posizioni cancellate |
| Approvazione DG                             | flag movimento impostato              |
| Rifiuto DG                                  | processo chiuso, posizioni cancellate |
| MIGO su reservation anno precedente         | blocco da MB_MIGO_BADI                |
| Reservation cancellata RESB-XLOEK = X       | esclusa da budget                     |

**Stima roadmap per sviluppatore esperto con AI**

Indicativamente, assumendo backend OData disponibile o sviluppato in parallelo.

| **Fase**                                 | **Effort stimato** |
| ---------------------------------------- | ------------------ |
| Analisi tecnica + contratti OData        | 2-4 gg             |
| Setup BTP/BAS/app router/deploy pipeline | 1-2 gg             |
| App Gestione Budget                      | 5-8 gg             |
| App Approvazione Reservation             | 6-10 gg            |
| Integrazione F4839 + Launchpad/Work Zone | 2-4 gg             |
| Sicurezza, ruoli, test E2E               | 4-6 gg             |
| Hardening, bug fixing, UAT support       | 3-5 gg             |

Totale realistico lato Fiori/BTP:

23-39 giorni uomo

Con sviluppatore senior e uso efficace di IA per generazione boilerplate, formatter, mock, test e refactoring, la fascia più probabile è:

25-30 giorni uomo

Questo non include lo sviluppo ABAP Gateway/BADI/BAPI completo, che è una dipendenza separata.

**Dipendenze backend critiche**

Queste vanno chiarite prima o durante la Fase 0, perché impattano direttamente la UI.

| **Dipendenza**                     | **Impatto UI**           |
| ---------------------------------- | ------------------------ |
| OData budget CRUD stabile          | blocca App Budget        |
| OData reservation approval         | blocca App Approvazione  |
| Determinazione approvatore da CdC  | filtri worklist          |
| Calcolo importo reservation        | dettaglio e warning      |
| Messaggi BAPI normalizzati         | gestione errori frontend |
| Deep link da mail                  | routing app approvazione |
| Autorizzazioni per utente corrente | visibilità dati/action   |
| Gestione lock/concorrenza          | approvazioni simultanee  |

**Alternative architetturali**

**Alternativa 1 - Usare My Inbox / Workflow**

Ha senso solo se si vuole introdurre un workflow formale BTP/SAP Workflow Management o Flexible Workflow.

**Pro:**

\- inbox standard;  
\- tracciabilità workflow;  
\- notifiche più strutturate;  
\- meno logica custom lato UI.

**Contro:**

\- maggiore setup;  
\- maggiore complessità architetturale;  
\- requisito attuale già prevede tabella custom e app custom;  
\- possibile overengineering.

Per questo requisito, io resterei sulla **custom approval app**.

**Alternativa 2 - Una sola app Freestyle con due moduli**

Si può fare una sola app con due sezioni:

/budget  
/approvals

**Pro:**

\- un solo deploy;  
\- componenti condivisi;  
\- meno configurazione BTP.

**Contro:**

\- ruoli più sporchi;  
\- ciclo di vita meno separato;  
\- app più grande;  
\- meno chiara per business.

Consiglio: **due app separate**, perché budget management e approvazione reservation hanno utenti, frequenza d'uso e autorizzazioni diverse.

**Alternativa 3 - Fiori Elements per App Budget**

Potrebbe avere senso se il servizio OData espone bene metadata, annotations e draft-like behavior.

**Pro:**

\- sviluppo più rapido;  
\- meno codice UI;  
\- standard UX.

**Contro:**

\- meno flessibile sulla distribuzione budget;  
\- più dipendente da annotation backend;  
\- meno controllo su validazioni e UX custom.

Dato che il requisito parla esplicitamente di **Fiori Freestyle**, terrei UI5 Freestyle.

**Check-list debug ambiente BTP/S/4**

**Destination + Cloud Connector**

Verificare:

BTP Destination reachable  
Cloud Connector connected  
Virtual host/port corretti  
Principal propagation o Basic/Auth configurata  
Location ID valorizzato se necessario  
Path /sap/opu/odata raggiungibile

Test rapido:

/sap/opu/odata/sap/ZGW_BUDGET_SRV/\$metadata  
/sap/opu/odata/sap/ZGW_RES_APPROVAL_SRV/\$metadata

**BAS / App Router**

Verificare:

xs-app.json route verso destination  
csrf token su POST/PUT  
model path nel manifest  
proxy locale BAS se usato  
CORS non gestito a mano: deve passare da approuter/destination

**Gateway S/4**

Per errori:

401/403 -> ruoli, auth, principal propagation  
404 -> servizio non attivo, path errato, EntitySet sbagliato  
500 -> dump/backend, BAPI, DPC_EXT  
CSRF -> token non recuperato o batch errato

Transazioni utili:

/IWFND/ERROR_LOG  
/IWFND/GW_CLIENT  
/IWFND/MAINT_SERVICE  
/IWBEP/ERROR_LOG  
ST22  
SU53  
SICF  
SLG1

**Priorità consigliata**

Io imposterei lo sviluppo così:

- **Bloccare contratto OData e mock metadata.**
- **Sviluppare App Approvazione Reservation prima della App Budget**, perché è più critica per il processo operativo.
- **Sviluppare App Budget subito dopo**, con focus su validazioni importi e audit.
- **Integrare F4839 senza estensione iniziale.**
- **Valutare extension/adaptation F4839 solo dopo primo test end-to-end.**

La parte Fiori deve rimanere sottile: gestione UX, filtri, stati, navigazione e messaggi. La coerenza di budget, reservation, approvazioni, flag SAP e commit BAPI deve restare centralizzata in S/4.

inseriremo dei controlli formali sugli input lato Fiori per vincolare il più possibile l'utente ad inserire dati realisticamente validi ai fini dei controlli su S4 (i warning che hai proposto. valuta se sia possibile aggiungerne altrove).

Sì: i controlli lato Fiori hanno senso, ma li tratterei come "pre-validazione guidata", mentre i blocchi definitivi restano in S/4. Li aggiungerei non solo nella App Budget, ma anche nell'app approvativa e, con cautela, sulla standard F4839 tramite estensione/adaptation se serve davvero.

Ragionato per 28s

**Contesto tecnico ricostruito**

Stiamo parlando delle **2 app Fiori Freestyle custom su BTP**: gestione budget e approvazione reservation, integrate via OData/Gateway con S/4 on-prem.  
I controlli lato Fiori servono a guidare l'utente ed evitare input palesemente incoerenti, ma il blocco definitivo resta in S/4 tramite BADI, BAPI e logiche Gateway, come previsto dal requisito.

**Principio consigliato**

Inserirei tre livelli di controllo:

1\. Controllo formale UI5  
Campi obbligatori, formato, lunghezza, importi positivi, date coerenti.  
<br/>2\. Controllo semantico tramite value help / OData  
CdC esistente, esercizio valido, materiale ammesso, divisione coerente, approvatore censito.  
<br/>3\. Pre-check backend non bloccante o bloccante  
Simulazione budget, disponibilità approvatore, stato reservation, coerenza workflow.

Il frontend deve ridurre l'errore umano, ma non deve mai diventare la fonte di verità.

**Dove aggiungerei i controlli**

**1\. App Gestione Budget**

Qui inserirei i controlli più forti, perché l'errore sul budget impatta tutto il processo.

**Controlli formali**

| **Campo**                    | **Controllo Fiori**                                     |
| ---------------------------- | ------------------------------------------------------- |
| Esercizio                    | obbligatorio, numerico, 4 cifre, range plausibile       |
| CdC DG                       | obbligatorio, selezionabile solo da value help          |
| CdC UO                       | obbligatorio, selezionabile solo da value help filtrata |
| Importo assegnato DG         | obbligatorio, maggiore di zero                          |
| Importo assegnato UO         | obbligatorio, maggiore di zero                          |
| Importo distribuito          | read-only                                               |
| Importo consumato            | read-only                                               |
| Data assegnazione/modifica   | read-only                                               |
| Utente assegnazione/modifica | read-only                                               |

**Controlli semantici App Budget**

Aggiungerei questi warning/blocchi:

| **Caso**                                          | **Tipo**         | **Messaggio suggerito**                                              |
| ------------------------------------------------- | ---------------- | -------------------------------------------------------------------- |
| Budget DG = 0                                     | warning/blocco   | "Inserire un importo budget maggiore di zero."                       |
| Budget UO superiore al residuo DG                 | blocco           | "Importo UO superiore al budget residuo DG."                         |
| Riduzione budget DG sotto importo già distribuito | blocco           | "Il budget DG non può essere inferiore all'importo già distribuito." |
| Riduzione budget UO sotto importo consumato       | blocco           | "Il budget UO non può essere inferiore all'importo già consumato."   |
| CdC DG già presente per esercizio                 | blocco           | "Budget già esistente per esercizio e centro di costo DG."           |
| CdC UO già assegnato a stessa DG/esercizio        | blocco           | "Centro di costo UO già presente per questa DG."                     |
| CdC UO uguale a CdC DG                            | warning o blocco | "Verificare: centro di costo UO uguale a centro di costo DG."        |
| Esercizio passato                                 | warning          | "Si sta modificando un esercizio precedente."                        |
| Esercizio futuro oltre N anni                     | warning          | "Esercizio futuro: verificare la correttezza dell'anno budget."      |

**App Budget - task aggiuntivi**

Aggiungerei alla roadmap questi task.

| **ID** | **Task**                                                       |
| ------ | -------------------------------------------------------------- |
| BUD-15 | Implementare validazione centralizzata input budget            |
| BUD-16 | Implementare value help CdC DG e CdC UO da OData               |
| BUD-17 | Implementare controllo duplicati lato UI prima del salvataggio |
| BUD-18 | Implementare controllo residuo DG prima di assegnazione UO     |
| BUD-19 | Bloccare riduzione budget sotto distribuito/consumato          |
| BUD-20 | Introdurre MessageManager UI5 per errori multipli              |
| BUD-21 | Introdurre pre-check backend ValidateBudgetAssignment          |
| BUD-22 | Evidenziare residuo budget con stato semantico                 |
| BUD-23 | Gestire refresh automatico dopo salvataggio/distribuzione      |

**OData consigliati per value help**

Eviterei input liberi su CdC, materiali e approvatori. Meglio value help filtrate.

Esempi:

GET /sap/opu/odata/sap/ZGW_BUDGET_SRV/CostCenterSet?\$filter=Gjahr eq '2026' and CostCenterType eq 'DG'  
<br/>GET /sap/opu/odata/sap/ZGW_BUDGET_SRV/CostCenterSet?\$filter=Gjahr eq '2026' and CostCenterType eq 'UO'  
<br/>GET /sap/opu/odata/sap/ZGW_BUDGET_SRV/BudgetDGSet(Gjahr='2026',KostlDg='D001')

Per pre-check:

POST /sap/opu/odata/sap/ZGW_BUDGET_SRV/ValidateBudgetAssignment

Payload esempio:

{  
"Gjahr": "2026",  
"KostlDg": "D001",  
"KostlUo": "U001",  
"Amount": "25000.00"  
}

Risposta consigliata:

{  
"Valid": true,  
"Severity": "S",  
"Message": "Assegnazione budget valida",  
"ResidualAmount": "75000.00"  
}

**Snippet UI5 consigliato**

**XML View - Input importo con stato errore**

<Input  
id="inpAmount"  
value="{  
path: 'budgetModel>/Amount',  
type: 'sap.ui.model.type.Float',  
constraints: {  
minimum: 0.01  
}  
}"  
valueState="{viewModel>/amountValueState}"  
valueStateText="{viewModel>/amountValueStateText}"  
textAlign="End"  
liveChange=".onBudgetAmountLiveChange" />

**Controller - validazione centralizzata**

// controller/BudgetDetail.controller.js  
<br/>\_validateBudgetInput: function () {  
const oViewModel = this.getView().getModel("viewModel");  
const oBudgetModel = this.getView().getModel("budgetModel");  
const oData = oBudgetModel.getData();  
<br/>let bValid = true;  
<br/>oViewModel.setProperty("/amountValueState", "None");  
oViewModel.setProperty("/amountValueStateText", "");  
<br/>if (!oData.Gjahr || !/^\\d{4}\$/.test(oData.Gjahr)) {  
bValid = false;  
oViewModel.setProperty("/gjahrValueState", "Error");  
oViewModel.setProperty("/gjahrValueStateText", "Inserire un esercizio valido a 4 cifre.");  
}  
<br/>if (!oData.KostlDg) {  
bValid = false;  
oViewModel.setProperty("/kostlDgValueState", "Error");  
oViewModel.setProperty("/kostlDgValueStateText", "Selezionare un centro di costo DG.");  
}  
<br/>const fAmount = Number(oData.Amount);  
<br/>if (!fAmount || fAmount <= 0) {  
bValid = false;  
oViewModel.setProperty("/amountValueState", "Error");  
oViewModel.setProperty("/amountValueStateText", "Inserire un importo maggiore di zero.");  
}  
<br/>if (Number(oData.Amount) < Number(oData.DistributedAmount || 0)) {  
bValid = false;  
oViewModel.setProperty("/amountValueState", "Error");  
oViewModel.setProperty(  
"/amountValueStateText",  
"Il budget assegnato non può essere inferiore all'importo già distribuito."  
);  
}  
<br/>return bValid;  
}

**2\. App Approvazione Reservation**

Anche qui aggiungerei controlli formali, soprattutto per evitare approvazioni non coerenti o rifiuti incompleti.

**Controlli consigliati**

| **Oggetto**                               | **Controllo Fiori**                                 |
| ----------------------------------------- | --------------------------------------------------- |
| Reservation                               | deve esistere ed essere ancora in stato approvabile |
| Livello corrente                          | l'utente può agire solo sul proprio livello         |
| Approvatore UO                            | solo se reservation in attesa I livello             |
| Approvatore DG                            | solo se I livello già approvato                     |
| Motivo rifiuto                            | obbligatorio, max 50 caratteri                      |
| Rifiuto                                   | conferma esplicita prima dell'invio                 |
| Approvazione finale                       | conferma esplicita perché abilita movimentazione    |
| Reservation già rifiutata/cancellata      | azioni disabilitate                                 |
| Reservation già approvata da altro utente | refresh e messaggio                                 |
| Items cancellati RESB-XLOEK = X           | non conteggiati / evidenziati                       |

**Warning aggiuntivi in approvazione**

| **Caso**                                             | **Tipo**               | **Messaggio suggerito**                                              |
| ---------------------------------------------------- | ---------------------- | -------------------------------------------------------------------- |
| Importo reservation vicino al residuo budget         | warning                | "La reservation utilizza oltre l'80% del budget residuo."            |
| Reservation con importo superiore al residuo attuale | warning/blocco backend | "Il budget residuo risulta insufficiente alla data odierna."         |
| Item con quantità anomala                            | warning                | "Verificare quantità elevata per il materiale selezionato."          |
| Reservation creata a dicembre                        | warning                | "Reservation di fine esercizio: verificare data registrazione MIGO." |
| Approvazione DG senza I livello                      | blocco                 | "Approvazione UO non presente."                                      |
| Utente approvatore non corrispondente                | blocco                 | "Utente non autorizzato per questo livello di approvazione."         |
| Stato modificato da altro utente                     | blocco con refresh     | "La reservation è stata aggiornata. Ricaricare i dati."              |

**App Approvazione - task aggiuntivi**

| **ID** | **Task**                                                    |
| ------ | ----------------------------------------------------------- |
| APR-17 | Validare stato reservation prima di approva/rifiuta         |
| APR-18 | Validare motivo rifiuto obbligatorio e lunghezza 50 char    |
| APR-19 | Disabilitare azioni non compatibili con livello corrente    |
| APR-20 | Implementare pre-check backend ValidateReservationApproval  |
| APR-21 | Evidenziare reservation critiche per budget residuo         |
| APR-22 | Gestire optimistic locking/stato modificato da altro utente |
| APR-23 | Conferma esplicita per approvazione finale DG               |
| APR-24 | Refresh automatico worklist dopo azione                     |
| APR-25 | Messaggi differenziati per warning, errore, successo        |

**Snippet rifiuto reservation**

**XML Fragment - dialog motivo rifiuto**

<Dialog  
id="rejectDialog"  
title="Rifiuta reservation"  
contentWidth="30rem">  
<br/>&lt;VBox class="sapUiMediumMargin"&gt;  
<Label  
text="Motivo rifiuto"  
required="true" />  
<br/><TextArea  
id="txtRejectReason"  
value="{approvalModel>/RejectReason}"  
maxLength="50"  
rows="4"  
width="100%"  
valueState="{viewModel>/rejectReasonValueState}"  
valueStateText="{viewModel>/rejectReasonValueStateText}"  
liveChange=".onRejectReasonLiveChange" />  
<br/><Text  
text="{= \${approvalModel>/RejectReason}.length + ' / 50 caratteri'}" />  
&lt;/VBox&gt;  
<br/>&lt;beginButton&gt;  
<Button  
text="Conferma rifiuto"  
type="Reject"  
press=".onConfirmReject" />  
&lt;/beginButton&gt;  
<br/>&lt;endButton&gt;  
<Button  
text="Annulla"  
press=".onCancelReject" />  
&lt;/endButton&gt;  
&lt;/Dialog&gt;

**Controller - controllo motivo**

onConfirmReject: function () {  
const oApprovalModel = this.getView().getModel("approvalModel");  
const oViewModel = this.getView().getModel("viewModel");  
const sReason = (oApprovalModel.getProperty("/RejectReason") || "").trim();  
<br/>oViewModel.setProperty("/rejectReasonValueState", "None");  
oViewModel.setProperty("/rejectReasonValueStateText", "");  
<br/>if (!sReason) {  
oViewModel.setProperty("/rejectReasonValueState", "Error");  
oViewModel.setProperty(  
"/rejectReasonValueStateText",  
"Il motivo del rifiuto è obbligatorio."  
);  
return;  
}  
<br/>if (sReason.length > 50) {  
oViewModel.setProperty("/rejectReasonValueState", "Error");  
oViewModel.setProperty(  
"/rejectReasonValueStateText",  
"Il motivo non può superare 50 caratteri."  
);  
return;  
}  
<br/>this.\_rejectReservation(sReason);  
}

**3\. Controlli sulla Fiori Standard F4839**

Qui distinguerei due livelli.

**Livello A - Senza estendere la standard**

È la soluzione che preferisco inizialmente.

I controlli principali restano backend:

BADI MB_RESERVATION_BADI - DATA_CHECK  
BADI MB_MIGO_BADI  
BAPI_RESERVATION_CHANGE

La Fiori standard mostrerà i messaggi bloccanti restituiti da S/4.

Controlli che devono stare sicuramente in backend:

| **Controllo**                                    | **Motivo**               |
| ------------------------------------------------ | ------------------------ |
| Budget disponibile                               | dato ufficiale S/4       |
| Esclusione reservation cancellate RESB-XLOEK = X | logica contabile/backend |
| Movimento 201                                    | coerenza MM              |
| Materiali cancelleria                            | master data backend      |
| Valore da MBEW-VERPR / PEINH                     | dato ufficiale SAP       |
| Reservation a cavallo anno                       | controllo su MIGO        |
| Impostazione XWAOK / MOVEMENT                    | modifica standard SAP    |
| Flag cancellazione DELETE_IND                    | irreversibile lato SAP   |

**Livello B - Con estensione/adaptation della F4839**

La valuterei solo se gli utenti rischiano molti errori in creazione reservation.

Possibili controlli UI aggiuntivi:

| **Campo F4839** | **Controllo UI**                                   |
| --------------- | -------------------------------------------------- |
| Movimento       | default o vincolo a 201, se tecnicamente possibile |
| Divisione       | value help filtrata                                |
| Centro di costo | value help filtrata su CdC abilitati a budget      |
| Materiale       | value help filtrata solo materiali cancelleria     |
| Quantità        | maggiore di zero                                   |
| Magazzino       | coerente con divisione/materiale                   |
| Data base       | warning se fine esercizio                          |
| Item multipli   | warning se totale stimato vicino al budget residuo |

Però eviterei modifiche invasive alla standard. Prima farei un test end-to-end con soli controlli backend.

**4\. Configurazione approvatori**

Nel requisito c'è una tabella custom con:

CdC  
Utente SAP approvatore I livello  
Mail approvatore I livello  
Utente SAP approvatore II livello  
Mail approvatore II livello

Io non la lascerei gestire solo da SM30, se il processo è business-critical. La includerei come sezione nella **App Gestione Budget** oppure come tab secondaria.

**Controlli consigliati**

| **Campo**                  | **Controllo**                                              |
| -------------------------- | ---------------------------------------------------------- |
| CdC                        | obbligatorio, esistente, attivo                            |
| Utente SAP I livello       | obbligatorio, utente valido                                |
| Mail I livello             | obbligatoria, formato email valido                         |
| Utente SAP II livello      | obbligatorio, utente valido                                |
| Mail II livello            | obbligatoria, formato email valido                         |
| UO = DG                    | warning o blocco secondo policy                            |
| CdC senza approvatori      | blocco su creazione reservation o warning forte in monitor |
| Approvatore non più valido | warning in app configurazione                              |

**Task aggiuntivo consigliato**

Aggiungerei alla roadmap:

| **ID** | **Task**                                          |
| ------ | ------------------------------------------------- |
| CFG-01 | Aggiungere sezione configurazione approvatori CdC |
| CFG-02 | Value help utenti SAP approvatori                 |
| CFG-03 | Validazione formato email                         |
| CFG-04 | Controllo duplicati per CdC                       |
| CFG-05 | Warning approvatore UO/DG coincidente             |
| CFG-06 | Verifica backend approvatore attivo               |
| CFG-07 | Blocco salvataggio se configurazione incompleta   |
| CFG-08 | Audit modifica configurazione approvatori         |

**5\. Controlli trasversali UI5**

Questi li implementerei in entrambe le app.

**MessageManager centralizzato**

// Component.js  
<br/>init: function () {  
UIComponent.prototype.init.apply(this, arguments);  
<br/>const oMessageManager = sap.ui.getCore().getMessageManager();  
this.setModel(oMessageManager.getMessageModel(), "message");  
<br/>this.getRouter().initialize();  
}

**ErrorHandler comune**

// util/ErrorHandler.js  
<br/>sap.ui.define(\[  
"sap/m/MessageBox"  
\], function (MessageBox) {  
"use strict";  
<br/>return {  
handleODataError: function (oError) {  
let sMessage = "Errore tecnico durante la comunicazione con S/4.";  
<br/>try {  
const oResponse = JSON.parse(oError.responseText);  
sMessage =  
oResponse.error &&  
oResponse.error.message &&  
oResponse.error.message.value  
? oResponse.error.message.value  
: sMessage;  
} catch (e) {  
// fallback standard  
}  
<br/>MessageBox.error(sMessage);  
}  
};  
});

**6\. Validazioni da non fare solo lato Fiori**

Questi controlli possono avere warning lato UI, ma il blocco deve restare backend:

| **Controllo**                          | **Backend owner**  |
| -------------------------------------- | ------------------ |
| Budget realmente disponibile           | S/4 Gateway/BADI   |
| Reservation già consumata o cancellata | S/4                |
| Stato approvativo ufficiale            | tabella custom S/4 |
| Abilitazione movimentazione            | BAPI backend       |
| Calcolo valore materiale               | MBEW               |
| Reservation a cavallo anno             | MB_MIGO_BADI       |
| Autorizzazione approvatore             | S/4/BTP roles      |
| Concorrenza approvazioni               | backend lock/stato |

**Roadmap aggiornata lato Fiori**

Aggiungerei una fase dedicata.

**Nuova Fase - Validazione e pre-check**

| **Fase** | **Task**                                         |
| -------- | ------------------------------------------------ |
| VAL-01   | Definire matrice controlli UI/backend            |
| VAL-02   | Implementare validazioni formali UI5             |
| VAL-03   | Implementare value help vincolate                |
| VAL-04   | Implementare MessageManager centralizzato        |
| VAL-05   | Implementare pre-check OData per budget          |
| VAL-06   | Implementare pre-check OData per approvazione    |
| VAL-07   | Implementare warning semantici su residuo budget |
| VAL-08   | Implementare gestione stato stale/concorrenza    |
| VAL-09   | Test messaggi warning/error/success              |
| VAL-10   | UAT con casi limite business                     |

Effort aggiuntivo stimato:

3-5 giorni uomo

Se includiamo anche configurazione approvatori in UI:

+2-4 giorni uomo

**Consiglio finale**

Io imposterei i controlli così:

App Budget:  
controlli molto forti, quasi bloccanti, perché governa i dati master del processo.  
<br/>App Approvazione:  
controlli su stato, autorizzazione, motivo rifiuto, coerenza livello e warning budget.  
<br/>F4839 standard:  
inizialmente solo controlli backend via BADI.  
Estensione UI solo dopo UAT, se emerge che gli utenti sbagliano spesso movimento, materiale, CdC o date.

In pratica, aggiungerei i warning anche su:

\- residuo budget basso;  
\- esercizio passato/futuro;  
\- budget ridotto sotto distribuito/consumato;  
\- CdC senza approvatori;  
\- approvatore non configurato;  
\- reservation di fine anno;  
\- reservation già modificata da altro utente;  
\- rifiuto senza motivo;  
\- approvazione finale DG con impatto su movimentazione.

la parte ABAP non sarà di mia cura. ti fornirò io di volta in volta gli endpoint, ed eventualmente filtri e payload da integrare lato UI5. partiremo dallo sviluppo interfacce e logiche di controllo e navigazione frontend, mentre il Backend sviluppa i servizi che chiameremo.

Ragionato per 4s

**Contesto tecnico aggiornato**

Sviluppiamo **solo la parte Fiori Freestyle/UI5 su BTP**, partendo da BAS, con logiche frontend, navigazione, validazioni formali, mock temporanei e integrazione progressiva degli endpoint OData/REST che verranno forniti dal backend S/4.  
Le logiche ABAP/BADI/BAPI/Gateway restano fuori perimetro, ma la UI dovrà essere pronta a consumarle quando disponibili.

**Impostazione corretta**

Procederei con un approccio **frontend-first**, usando inizialmente:

\- JSONModel mock per sviluppo UI;  
\- service wrapper centralizzato;  
\- validazioni UI5 già definitive;  
\- routing e layout completi;  
\- adapter per sostituire facilmente mock con OData/REST reale;  
\- gestione errori già compatibile con risposte Gateway.

In pratica: sviluppiamo le app come se gli endpoint esistessero già, ma isolando tutte le chiamate in layer dedicati.

**Architettura frontend consigliata**

Per entrambe le app userei questa struttura:

/webapp  
/controller  
BudgetList.controller.js  
BudgetDetail.controller.js  
ApprovalList.controller.js  
ApprovalDetail.controller.js  
<br/>/view  
BudgetList.view.xml  
BudgetDetail.view.xml  
ApprovalList.view.xml  
ApprovalDetail.view.xml  
<br/>/fragment  
BudgetEditDialog.fragment.xml  
BudgetUoDialog.fragment.xml  
RejectReservationDialog.fragment.xml  
MessagePopover.fragment.xml  
<br/>/model  
formatter.js  
models.js  
<br/>/service  
BudgetService.js  
ApprovalService.js  
MockService.js  
<br/>/util  
Validator.js  
ErrorHandler.js  
Constants.js  
Navigation.js  
<br/>Component.js  
manifest.json

Il punto chiave è questo:

Controller  
↓  
Service JS  
↓  
Mock JSONModel oggi  
↓  
ODataModel / REST endpoint domani

Così quando mi fornirai endpoint, filtri e payload, cambieremo solo il service layer, non tutta la UI.

**Roadmap frontend-first aggiornata**

**Fase 1 - Setup base progetto UI5**

**Task**

| **ID** | **Task**                             | **Output**                            |
| ------ | ------------------------------------ | ------------------------------------- |
| FE-01  | Creazione progetto BAS UI5 Freestyle | app funzionante localmente            |
| FE-02  | Setup manifest.json                  | routing, models, i18n                 |
| FE-03  | Setup modelli JSON locali            | viewModel, filterModel, appStateModel |
| FE-04  | Setup formatter centralizzato        | importi, date, stati                  |
| FE-05  | Setup validator centralizzato        | validazioni formali                   |
| FE-06  | Setup error handler                  | errori Gateway/REST/mock              |
| FE-07  | Setup mock data                      | JSON locali realistici                |
| FE-08  | Setup navigazione                    | list/detail/dialog                    |

**App 1 - Gestione Budget**

**Sviluppo iniziale senza backend**

Per la App Budget possiamo sviluppare subito:

\- lista budget DG;  
\- filtri esercizio / centro di costo;  
\- dettaglio budget DG;  
\- tabella budget UO;  
\- dialog creazione/modifica budget DG;  
\- dialog assegnazione/modifica budget UO;  
\- calcolo residui lato UI;  
\- warning e blocchi formali;  
\- navigazione;  
\- busy state;  
\- message handling.

**Mock dati iniziali**

File:

webapp/localService/mockdata/BudgetDG.json

\[  
{  
"Gjahr": "2026",  
"KostlDg": "DG001",  
"KostlDgText": "Direzione Generale Acquisti",  
"AssignedAmount": 100000,  
"DistributedAmount": 75000,  
"ResidualAmount": 25000,  
"AssignmentDate": "2026-01-10",  
"AssignmentUser": "USER_BUDGET",  
"ChangedBy": "USER_BUDGET",  
"ChangedAt": "2026-02-01"  
}  
\]

File:

webapp/localService/mockdata/BudgetUO.json

\[  
{  
"Gjahr": "2026",  
"KostlDg": "DG001",  
"KostlUo": "UO001",  
"KostlUoText": "Ufficio Economato",  
"AssignedAmount": 30000,  
"ConsumedAmount": 12000,  
"ResidualAmount": 18000  
},  
{  
"Gjahr": "2026",  
"KostlDg": "DG001",  
"KostlUo": "UO002",  
"KostlUoText": "Ufficio Servizi Generali",  
"AssignedAmount": 45000,  
"ConsumedAmount": 10000,  
"ResidualAmount": 35000  
}  
\]

**Service mock-ready**

File:

webapp/service/BudgetService.js

sap.ui.define(\[\], function () {  
"use strict";  
<br/>return {  
getBudgetDGList: function (oComponent, oFilters) {  
const oModel = oComponent.getModel("mockBudget");  
const aData = oModel.getProperty("/BudgetDG") || \[\];  
<br/>let aResult = aData;  
<br/>if (oFilters?.Gjahr) {  
aResult = aResult.filter((oItem) => oItem.Gjahr === oFilters.Gjahr);  
}  
<br/>if (oFilters?.KostlDg) {  
aResult = aResult.filter((oItem) => oItem.KostlDg === oFilters.KostlDg);  
}  
<br/>return Promise.resolve(aResult);  
},  
<br/>getBudgetDGDetail: function (oComponent, sGjahr, sKostlDg) {  
const oModel = oComponent.getModel("mockBudget");  
const aBudgetDG = oModel.getProperty("/BudgetDG") || \[\];  
const aBudgetUO = oModel.getProperty("/BudgetUO") || \[\];  
<br/>const oHeader = aBudgetDG.find((oItem) =>  
oItem.Gjahr === sGjahr && oItem.KostlDg === sKostlDg  
);  
<br/>if (!oHeader) {  
return Promise.reject({  
message: "Budget DG non trovato."  
});  
}  
<br/>const aItems = aBudgetUO.filter((oItem) =>  
oItem.Gjahr === sGjahr && oItem.KostlDg === sKostlDg  
);  
<br/>return Promise.resolve({  
Header: oHeader,  
Items: aItems  
});  
},  
<br/>validateBudgetAssignment: function (oPayload) {  
const fAssignedAmount = Number(oPayload.AssignedAmount);  
const fDistributedAmount = Number(oPayload.DistributedAmount || 0);  
const fConsumedAmount = Number(oPayload.ConsumedAmount || 0);  
<br/>if (!oPayload.Gjahr || !/^\\d{4}\$/.test(oPayload.Gjahr)) {  
return Promise.reject({  
message: "Inserire un esercizio valido."  
});  
}  
<br/>if (!oPayload.KostlDg) {  
return Promise.reject({  
message: "Selezionare un centro di costo DG."  
});  
}  
<br/>if (!fAssignedAmount || fAssignedAmount <= 0) {  
return Promise.reject({  
message: "Inserire un importo maggiore di zero."  
});  
}  
<br/>if (fAssignedAmount < fDistributedAmount) {  
return Promise.reject({  
message: "Il budget DG non può essere inferiore all'importo già distribuito."  
});  
}  
<br/>if (fAssignedAmount < fConsumedAmount) {  
return Promise.reject({  
message: "Il budget UO non può essere inferiore all'importo già consumato."  
});  
}  
<br/>return Promise.resolve({  
Valid: true,  
Message: "Controlli formali superati."  
});  
}  
};  
});

Quando arriverà l'OData reale, sostituiremo internamente getBudgetDGList, getBudgetDGDetail, validateBudgetAssignment con chiamate tipo:

GET /BudgetDGSet?\$filter=Gjahr eq '2026'  
GET /BudgetDGSet(Gjahr='2026',KostlDg='DG001')?\$expand=ToBudgetUO  
POST /ValidateBudgetAssignment

I controller resteranno quasi invariati.

**App 2 - Approvazione Reservation**

**Sviluppo iniziale senza backend**

Possiamo sviluppare subito:

\- worklist reservation da approvare;  
\- filtri per stato, CdC, livello, esercizio;  
\- dettaglio reservation;  
\- lista item reservation;  
\- stato approvativo;  
\- pulsanti approva/rifiuta;  
\- dialog motivo rifiuto;  
\- deep link da mail;  
\- warning su budget residuo;  
\- gestione reservation già modificata;  
\- refresh lista dopo azione.

**Mock dati reservation**

File:

webapp/localService/mockdata/ReservationApprovals.json

\[  
{  
"Reservation": "0001234567",  
"Gjahr": "2026",  
"CostCenter": "UO001",  
"CostCenterText": "Ufficio Economato",  
"RequestDate": "2026-03-15",  
"Requester": "MROSSI",  
"TotalAmount": 1450.75,  
"BudgetResidualAmount": 18000,  
"ApprovalLevel": "UO",  
"Status": "PENDING_UO",  
"CurrentApprover": "APPROVER_UO_01",  
"IsActionAllowed": true  
},  
{  
"Reservation": "0001234568",  
"Gjahr": "2026",  
"CostCenter": "UO002",  
"CostCenterText": "Ufficio Servizi Generali",  
"RequestDate": "2026-03-16",  
"Requester": "LBIANCHI",  
"TotalAmount": 32000,  
"BudgetResidualAmount": 35000,  
"ApprovalLevel": "DG",  
"Status": "PENDING_DG",  
"CurrentApprover": "APPROVER_DG_01",  
"IsActionAllowed": true  
}  
\]

File:

webapp/localService/mockdata/ReservationItems.json

\[  
{  
"Reservation": "0001234567",  
"Item": "0010",  
"Material": "MAT001",  
"MaterialText": "Carta A4",  
"Plant": "1000",  
"StorageLocation": "0001",  
"Quantity": 50,  
"Unit": "PAC",  
"MovingAveragePrice": 4.5,  
"PriceUnit": 1,  
"ItemAmount": 225,  
"DeletionFlag": false,  
"MovementAllowed": false  
}  
\]

**Service approvazioni mock-ready**

File:

webapp/service/ApprovalService.js

sap.ui.define(\[\], function () {  
"use strict";  
<br/>return {  
getApprovalList: function (oComponent, oFilters) {  
const oModel = oComponent.getModel("mockApproval");  
let aData = oModel.getProperty("/ReservationApprovals") || \[\];  
<br/>if (oFilters?.Gjahr) {  
aData = aData.filter((oItem) => oItem.Gjahr === oFilters.Gjahr);  
}  
<br/>if (oFilters?.Status) {  
aData = aData.filter((oItem) => oItem.Status === oFilters.Status);  
}  
<br/>if (oFilters?.CostCenter) {  
aData = aData.filter((oItem) => oItem.CostCenter === oFilters.CostCenter);  
}  
<br/>return Promise.resolve(aData);  
},  
<br/>getApprovalDetail: function (oComponent, sReservation) {  
const oModel = oComponent.getModel("mockApproval");  
const aHeaders = oModel.getProperty("/ReservationApprovals") || \[\];  
const aItems = oModel.getProperty("/ReservationItems") || \[\];  
<br/>const oHeader = aHeaders.find((oItem) => oItem.Reservation === sReservation);  
<br/>if (!oHeader) {  
return Promise.reject({  
message: "Reservation non trovata."  
});  
}  
<br/>return Promise.resolve({  
Header: oHeader,  
Items: aItems.filter((oItem) => oItem.Reservation === sReservation)  
});  
},  
<br/>validateApprovalAction: function (oReservation, sAction, sRejectReason) {  
if (!oReservation) {  
return Promise.reject({  
message: "Reservation non disponibile."  
});  
}  
<br/>if (!oReservation.IsActionAllowed) {  
return Promise.reject({  
message: "Utente non autorizzato ad agire su questa reservation."  
});  
}  
<br/>if (!\["PENDING_UO", "PENDING_DG"\].includes(oReservation.Status)) {  
return Promise.reject({  
message: "La reservation non è in stato approvabile."  
});  
}  
<br/>if (sAction === "REJECT" && !sRejectReason?.trim()) {  
return Promise.reject({  
message: "Il motivo del rifiuto è obbligatorio."  
});  
}  
<br/>if (sAction === "REJECT" && sRejectReason.trim().length > 50) {  
return Promise.reject({  
message: "Il motivo del rifiuto non può superare 50 caratteri."  
});  
}  
<br/>return Promise.resolve({  
Valid: true  
});  
},  
<br/>approveReservation: function (oComponent, sReservation) {  
return Promise.resolve({  
Reservation: sReservation,  
Status: "APPROVED",  
Message: "Reservation approvata correttamente."  
});  
},  
<br/>rejectReservation: function (oComponent, sReservation, sRejectReason) {  
return Promise.resolve({  
Reservation: sReservation,  
Status: "REJECTED",  
RejectReason: sRejectReason,  
Message: "Reservation rifiutata correttamente."  
});  
}  
};  
});

Quando arriveranno gli endpoint reali, mapperemo questi metodi verso:

GET /ReservationApprovalSet?\$filter=...  
GET /ReservationApprovalSet('0001234567')?\$expand=Items  
POST /ApproveReservation  
POST /RejectReservation  
POST /ValidateReservationApproval

**Validazioni frontend da implementare subito**

**App Budget**

| **Controllo**                          | **Tipo**                           |
| -------------------------------------- | ---------------------------------- |
| Esercizio obbligatorio                 | blocco UI                          |
| Esercizio formato YYYY                 | blocco UI                          |
| CdC DG obbligatorio                    | blocco UI                          |
| CdC UO obbligatorio                    | blocco UI                          |
| Importo maggiore di zero               | blocco UI                          |
| Budget DG non inferiore al distribuito | blocco UI                          |
| Budget UO non inferiore al consumato   | blocco UI                          |
| Importo UO non superiore al residuo DG | blocco UI                          |
| Esercizio passato                      | warning                            |
| Esercizio futuro oltre soglia          | warning                            |
| CdC senza approvatori                  | warning/blocco, in base a processo |

**App Approvazione Reservation**

| **Controllo**                                | **Tipo**             |
| -------------------------------------------- | -------------------- |
| Reservation in stato approvabile             | blocco UI            |
| Utente abilitato all'azione                  | blocco UI            |
| Livello approvativo coerente                 | blocco UI            |
| Motivo rifiuto obbligatorio                  | blocco UI            |
| Motivo rifiuto max 50 caratteri              | blocco UI            |
| Approvazione finale DG                       | conferma esplicita   |
| Reservation vicino al limite budget          | warning              |
| Reservation superiore al budget residuo noto | warning forte        |
| Reservation di fine anno                     | warning              |
| Dati modificati da altro utente              | refresh obbligatorio |

**Validator centralizzato**

File:

webapp/util/Validator.js

sap.ui.define(\[\], function () {  
"use strict";  
<br/>return {  
isFiscalYear: function (sValue) {  
return /^\\d{4}\$/.test(String(sValue || ""));  
},  
<br/>isPositiveAmount: function (vValue) {  
const fValue = Number(vValue);  
return !Number.isNaN(fValue) && fValue > 0;  
},  
<br/>isNonEmpty: function (sValue) {  
return String(sValue || "").trim().length > 0;  
},  
<br/>isMaxLength: function (sValue, iMaxLength) {  
return String(sValue || "").trim().length <= iMaxLength;  
},  
<br/>isEmail: function (sValue) {  
return /^\[^\\s@\]+@\[^\\s@\]+\\.\[^\\s@\]+\$/.test(String(sValue || ""));  
},  
<br/>isBudgetReductionAllowed: function (fNewAmount, fAlreadyUsedAmount) {  
return Number(fNewAmount) >= Number(fAlreadyUsedAmount || 0);  
},  
<br/>isReservationCriticalForBudget: function (fReservationAmount, fResidualAmount) {  
if (!fResidualAmount || Number(fResidualAmount) <= 0) {  
return true;  
}  
<br/>return Number(fReservationAmount) / Number(fResidualAmount) >= 0.8;  
}  
};  
});

**Formatter centralizzato**

File:

webapp/model/formatter.js

sap.ui.define(\[\], function () {  
"use strict";  
<br/>return {  
amountState: function (fResidualAmount) {  
const fValue = Number(fResidualAmount);  
<br/>if (Number.isNaN(fValue)) {  
return "None";  
}  
<br/>if (fValue < 0) {  
return "Error";  
}  
<br/>if (fValue === 0) {  
return "Warning";  
}  
<br/>return "Success";  
},  
<br/>approvalStatusState: function (sStatus) {  
switch (sStatus) {  
case "PENDING_UO":  
case "PENDING_DG":  
return "Warning";  
case "APPROVED":  
return "Success";  
case "REJECTED":  
return "Error";  
default:  
return "None";  
}  
},  
<br/>approvalStatusText: function (sStatus) {  
switch (sStatus) {  
case "PENDING_UO":  
return "In approvazione UO";  
case "PENDING_DG":  
return "In approvazione DG";  
case "APPROVED":  
return "Approvata";  
case "REJECTED":  
return "Rifiutata";  
default:  
return sStatus || "";  
}  
}  
};  
});

**Controller pattern consigliato**

Nei controller eviterei logica OData diretta.

Esempio:

sap.ui.define(\[  
"sap/ui/core/mvc/Controller",  
"sap/m/MessageBox",  
"../service/ApprovalService",  
"../util/ErrorHandler"  
\], function (  
Controller,  
MessageBox,  
ApprovalService,  
ErrorHandler  
) {  
"use strict";  
<br/>return Controller.extend("namespace.controller.ApprovalDetail", {  
onInit: function () {  
this.getOwnerComponent()  
.getRouter()  
.getRoute("approvalDetail")  
.attachPatternMatched(this.\_onRouteMatched, this);  
},  
<br/>\_onRouteMatched: function (oEvent) {  
const sReservation = oEvent.getParameter("arguments").Reservation;  
this.\_loadReservation(sReservation);  
},  
<br/>\_loadReservation: function (sReservation) {  
const oViewModel = this.getView().getModel("viewModel");  
<br/>oViewModel.setProperty("/busy", true);  
<br/>ApprovalService.getApprovalDetail(this.getOwnerComponent(), sReservation)  
.then((oData) => {  
this.getView().getModel("approvalModel").setData(oData);  
})  
.catch((oError) => {  
ErrorHandler.handle(oError);  
})  
.finally(() => {  
oViewModel.setProperty("/busy", false);  
});  
},  
<br/>onApprovePress: function () {  
const oReservation = this.getView()  
.getModel("approvalModel")  
.getProperty("/Header");  
<br/>ApprovalService.validateApprovalAction(oReservation, "APPROVE")  
.then(() => {  
MessageBox.confirm(  
"Confermare l'approvazione della reservation?",  
{  
actions: \[MessageBox.Action.OK, MessageBox.Action.CANCEL\],  
emphasizedAction: MessageBox.Action.OK,  
onClose: (sAction) => {  
if (sAction === MessageBox.Action.OK) {  
this.\_approveReservation(oReservation.Reservation);  
}  
}  
}  
);  
})  
.catch((oError) => {  
ErrorHandler.handle(oError);  
});  
},  
<br/>\_approveReservation: function (sReservation) {  
const oViewModel = this.getView().getModel("viewModel");  
<br/>oViewModel.setProperty("/busy", true);  
<br/>ApprovalService.approveReservation(this.getOwnerComponent(), sReservation)  
.then((oResult) => {  
MessageBox.success(oResult.Message);  
this.\_loadReservation(sReservation);  
})  
.catch((oError) => {  
ErrorHandler.handle(oError);  
})  
.finally(() => {  
oViewModel.setProperty("/busy", false);  
});  
}  
});  
});

**Quando mi fornirai gli endpoint**

Per ogni endpoint mi serviranno questi dati minimi:

\- metodo HTTP: GET, POST, PUT, PATCH, DELETE;  
\- URL completo relativo, es. /sap/opu/odata/sap/ZGW\_.../EntitySet;  
\- tipo servizio: OData V2, OData V4, REST custom;  
\- filtri supportati;  
\- chiavi EntitySet;  
\- payload request;  
\- payload response;  
\- formato errori;  
\- eventuale CSRF;  
\- eventuale batch sì/no;  
\- campi obbligatori;  
\- codifica valori, soprattutto chiavi con zeri iniziali.

Esempio formato ideale:

Endpoint:  
POST /sap/opu/odata/sap/ZGW_RES_APPROVAL_SRV/ApproveReservation  
<br/>Payload:  
{  
"Reservation": "0001234567",  
"ApprovalLevel": "DG"  
}  
<br/>Response:  
{  
"Reservation": "0001234567",  
"Status": "APPROVED",  
"Message": "Reservation approvata correttamente"  
}

A quel punto io mapperò il metodo:

ApprovalService.approveReservation(...)

senza impattare controller, view e navigazione.

**Configurazione manifest.json**

Partirei già con model reali disabilitati/commentati concettualmente e mock attivo.

{  
"sap.ui5": {  
"models": {  
"": {  
"dataSource": "mainService",  
"settings": {  
"useBatch": true,  
"defaultBindingMode": "TwoWay",  
"defaultCountMode": "Inline"  
},  
"type": "sap.ui.model.odata.v2.ODataModel"  
},  
"viewModel": {  
"type": "sap.ui.model.json.JSONModel"  
},  
"budgetModel": {  
"type": "sap.ui.model.json.JSONModel"  
},  
"approvalModel": {  
"type": "sap.ui.model.json.JSONModel"  
},  
"filterModel": {  
"type": "sap.ui.model.json.JSONModel"  
}  
},  
"routing": {  
"config": {  
"routerClass": "sap.m.routing.Router",  
"viewType": "XML",  
"viewPath": "namespace.view",  
"controlId": "app",  
"controlAggregation": "pages",  
"async": true  
},  
"routes": \[  
{  
"name": "budgetList",  
"pattern": "",  
"target": "budgetList"  
},  
{  
"name": "budgetDetail",  
"pattern": "budget/{Gjahr}/{KostlDg}",  
"target": "budgetDetail"  
},  
{  
"name": "approvalList",  
"pattern": "approvals",  
"target": "approvalList"  
},  
{  
"name": "approvalDetail",  
"pattern": "approval/{Reservation}",  
"target": "approvalDetail"  
}  
\],  
"targets": {  
"budgetList": {  
"viewName": "BudgetList"  
},  
"budgetDetail": {  
"viewName": "BudgetDetail"  
},  
"approvalList": {  
"viewName": "ApprovalList"  
},  
"approvalDetail": {  
"viewName": "ApprovalDetail"  
}  
}  
}  
}  
}

**Alternative sensate**

**Alternativa A - Sviluppo mock con JSONModel puro**

È quella che consiglio adesso.

**Pro:**

\- massima velocità;  
\- nessuna dipendenza dal backend;  
\- UI navigabile subito;  
\- validazioni sviluppabili da subito;  
\- facile demo agli utenti.

**Contro:**

\- serve poi mappare bene i nomi campo reali;  
\- rischio differenza tra mock e OData finale.

**Alternativa B - MockServer OData UI5**

Utile se il backend ti fornisce presto il \$metadata.

**Pro:**

\- più vicino a OData reale;  
\- binding OData già testabile;  
\- filtri e metadata simulabili.

**Contro:**

\- più setup;  
\- se il metadata cambia spesso, rallenta;  
\- meno flessibile all'inizio.

La userei solo quando avremo un primo \$metadata stabile.

**Alternativa C - Chiamate dirette OData già nei controller**

La eviterei.

**Contro:**

\- controller sporchi;  
\- refactoring pesante quando cambiano endpoint;  
\- logica duplicata;  
\- test più difficili.

**Check-list debug quando inizierà l'integrazione reale**

Quando passeremo da mock a S/4, per ogni chiamata controlleremo:

1\. /\$metadata raggiungibile  
2\. EntitySet corretto  
3\. chiavi corrette e zeri iniziali preservati  
4\. filtri OData encodati correttamente  
5\. CSRF token su POST/PUT/PATCH  
6\. batch attivo o disattivo coerente con Gateway  
7\. response error parsata correttamente  
8\. busy indicator rilasciato anche in errore  
9\. refresh dati dopo modifica  
10\. messaggi backend mostrati all'utente

Per errori HTTP:

401/403 -> ruoli, destination, principal propagation  
404 -> path servizio, EntitySet, routing approuter  
500 -> errore backend, controllare /IWFND/ERROR_LOG  
CSRF -> token mancante o scaduto

**Conclusione operativa**

La roadmap diventa quindi:

1\. Creare scheletro UI5 delle due app.  
2\. Implementare navigazione e layout.  
3\. Inserire mock JSON realistici.  
4\. Implementare validazioni formali e warning.  
5\. Centralizzare service layer.  
6\. Centralizzare error handling.  
7\. Preparare i metodi service con nomi definitivi.  
8\. Integrare progressivamente gli endpoint reali appena disponibili.

La scelta migliore è sviluppare ora una UI completa e navigabile, ma con backend intercambiabile. Quando arriveranno endpoint, filtri e payload, mapperemo i service uno alla volta senza riscrivere view e controller.