# KARE: Knowledge-Aware & Probabilistic Reasoning for Wind Turbine PHM


**KARE** (*Knowledge-Aware Reasoning Engine*) è un framework ibrido di Intelligenza Artificiale per la **Prognostica e la Gestione dello Stato di Salute (PHM - Prognostics and Health Management)** applicato a parchi eolici industriali.

Il sistema supera i limiti dei modelli "black-box" integrando tre paradigmi complementari dell'Intelligenza Artificiale:
1. **Sistemi Basati su Conoscenza (Knowledge-Based Logic Engine):** Regole fisiche ed euristiche di primo ordine per il rilevamento immediato e spiegabile delle anomalie.
2. **Ragionamento Probabilistico (Reti Bayesiane / BN):** Modellazione causale dell'incertezza, stima della probabilità di guasto e calcolo della *Remaining Useful Life* (RUL).
3. **Programmazione a Vincoli (CSP / Constraint Programming):** Ottimizzazione e schedulazione operativa degli interventi manutentivi sotto vincoli stringenti di sicurezza meteo, risorse e budget.

---

## Architettura del Sistema

L'architettura di KARE è suddivisa in una pipeline sequenziale a 4 livelli:

1. TELEMETRIA SCADA & PREPROCESSING  (Serie temporali, Deriva termica, Rolling Z-Score, Target RUL)

2. MOTORE LOGICO (KB): Regole termodinamiche e cinematiche, Diagnosi deterministica

3. RETE BAYESIANA (BN): Inferenza causale e calcolo del Failure Risk Score (compreso in [0, 1])

4. OTTIMIZZATORE CSP: Pianificazione squadre, Vincoli meteo (Vento < 12 m/s), Minimizzazione Costi

---

##  I Tre Pilastri Fondamentali

### 1. Knowledge Base & Logic Engine (`logic_engine.py`)
Modella la fisica dei principali sottosistemi dell'aerogeneratore:
* **Moltiplicatore di Giri (Gearbox):** Correlazione tra temperatura dell'olio lubrificante, temperatura cuscinetti e Z-Score mobile.
* **Sistema Idraulico del Passo Pale (Pitch System):** Monitoraggio della pressione del circuito idraulico per prevenire blocchi delle pale.
* **Generatore Elettrico:** Controllo di sovravelocità ($RPM$) e deriva termica degli avvolgimenti statorici.
* **Efficienza Aerodinamica:** Scostamento della potenza attiva ($kW$) rispetto alla curva teorica di potenza:
  $$\eta = \frac{P_{\text{effettiva}}}{\frac{1}{2}\rho A C_p v_{\text{vento}}^3}$$

### 2. Modellazione Probabilistica Bayesiana (`bayesian_learner.py`)
Costruisce un Grafo Aciclico Diretto (DAG) causale per inferire la distribuzione a posteriori dello stato di salute:
$$P(\text{HealthState} \mid \text{Vento}, \text{TempOlio}, \text{StressCuscinetti}, \text{PressioneIdraulica})$$

Gli stati discreti stimati sono:
* **`HEALTHY` (0):** Funzionamento nominale ($\text{RUL} > 300\text{ ore}$).
* **`WARNING` (1):** Usura termica iniziale o deriva idraulica ($100 < \text{RUL} \le 300\text{ ore}$).
* **`CRITICAL` (2):** Guasto meccanico o elettrico imminente ($\text{RUL} \le 100\text{ ore}$).

### 3. Pianificazione a Vincoli CSP (`maintenance_optimizer.py`)
Formula il problema di assegnazione come un **Constraint Satisfaction and Optimization Problem (CSOP)** risolto con Google OR-Tools CP-SAT:

* **Variabili Decisionali:** $X_{t, d, c} \in \{0, 1\}$ (Turbina $t$, Giorno $d$, Squadra $c$).
* **Hard Constraints:**
  1. *Unicità:* $\sum_{d} \sum_{c} X_{t, d, c} \le 1 \quad \forall t$
  2. *Deadline Critico:* $\sum_{d \le \text{Deadline}(t)} \sum_{c} X_{t, d, c} = 1 \quad \forall t \in \text{Critical}$
  3. *Capacità Squadre:* $\sum_{t} X_{t, d, c} \le 1 \quad \forall d, c$
  4. *Sicurezza Meteo:* $X_{t, d, c} = 0 \quad \text{se } v_{\text{vento\_previs}}(d) > 12.0\text{ m/s}$
* **Funzione Obiettivo:** Minimizzare la somma dei costi di intervento correttivo/preventivo, le perdite da mancata produzione energetica e il rischio residuo delle macchine non riparate.

---

## Struttura della repository

```text
├── config.py                         # Configurazione centralizzata, parametri fisici e soglie
├── data_loader.py                    # Pipeline SCADA, feature engineering e generatore dati
├── logic_engine.py                   # Knowledge-Based System e motore a regole logiche
├── bayesian_learner.py               # Rete Bayesiana, apprendimento CPD e inferenza MAP
├── maintenance_optimizer.py          # Modellazione e risoluzione del CSP con OR-Tools
├── main.py                           # Orchestratore della pipeline end-to-end
│
├── kb_evaluation.py                  # Metriche di classificazione e confusion matrix per la KB
├── cv_bayes.py                       # GroupKFold Cross-Validation della Rete Bayesiana
├── csp_evaluation.py                 # Stress test e benchmark scalabilità del solver CSP
├── experiment_runner.py              # Suite completa per l'esecuzione di tutti gli esperimenti
├── generate_documentation_assets.py  # Generazione automatica di tutti i grafici e figure
│
├── data/                             # Dati SCADA (generati o reali)
├── figures_doc/                      # Figure e grafici generati direttamente dal codice
├── results/                          # Report, matrici di confusione e file CSV esportati
└── requirements.txt                  # Dipendenze e librerie Python richieste