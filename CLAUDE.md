# Mappa benzina – Mini Digital Festival 2026

Progetto di data journalism sui prezzi dei carburanti in Italia. Il lavoro si fa in Pandas nel notebook `data-journalism.ipynb`.

## Modo di lavorare

- Si risponde in italiano.
- Il codice va scritto nella risposta in chat, come celle da copiare, e non direttamente nel notebook, a meno che l'utente non lo chieda.
- Prima di proporre il codice, provalo sui CSV veri. Nel notebook `display()` funziona; fuori dal notebook usa `print()`.

## I dati

Nella cartella ci sono due estrazioni, del 2026-11-02:

- `anagrafica.csv`: un impianto per riga, circa 23.930 righe. Colonne: `idImpianto|Gestore|Bandiera|Tipo Impianto|Nome Impianto|Indirizzo|Comune|Provincia|Latitudine|Longitudine`.
- `prezzi.csv`: un prezzo per riga, circa 92.700 righe. Colonne: `idImpianto|descCarburante|prezzo|isSelf|dtComu`.

Alcune cose da sapere sul formato:

- Il separatore è `|`. La prima riga ("Estrazione del ...") non è l'intestazione e va saltata.
- In `anagrafica.csv`, 109 righe hanno un `|` dentro `Nome Impianto` (es. `STOIL SIMPLE | gestori.prezzibenzina.it`), quindi `read_csv` dà errore. Leggi il file riga per riga: tieni fissi i primi 4 campi e gli ultimi 5, e unisci con `|` quello che sta in mezzo.
- `dtComu` è nel formato `%d/%m/%Y %H:%M:%S`.
- `isSelf` vale 0 per il servito e 1 per il self.
- `idImpianto` è unico in anagrafica, e tutti gli id dei prezzi esistono in anagrafica.
- `descCarburante` ha 59 valori diversi. I tipi di metano sono "Metano" (gas naturale compresso per auto), "GNL" (metano liquefatto) e "L-GNC".
- L'anagrafica non dice che carburanti vende ogni impianto: per saperlo si usano i prezzi.

## Caricamento

```python
import pandas as pd

with open("anagrafica.csv", encoding="utf-8") as f:
    next(f)
    header = next(f).rstrip("\n").split("|")
    rows = []
    for line in f:
        parts = line.rstrip("\n").split("|")
        rows.append(parts[:4] + ["|".join(parts[4:-5])] + parts[-5:])
anagrafica = pd.DataFrame(rows, columns=header)
anagrafica["idImpianto"] = anagrafica["idImpianto"].astype(int)
anagrafica[["Latitudine", "Longitudine"]] = anagrafica[["Latitudine", "Longitudine"]].apply(pd.to_numeric, errors="coerce")

prezzi = pd.read_csv("prezzi.csv", sep="|", skiprows=1,
                     parse_dates=["dtComu"], date_format="%d/%m/%Y %H:%M:%S")
```

## La tabella `dati`

Le scelte decise con l'utente:

- Dei prezzi si tengono tutte le colonne.
- Dell'anagrafica si tengono `idImpianto`, `Bandiera`, `Tipo Impianto`, `Comune`, `Provincia`, `Latitudine` e `Longitudine`. Latitudine e longitudine sono arrotondate a 4 decimali.
- Si tolgono i 3 impianti senza coordinate.
- Si tengono solo i prezzi con `dtComu >= "2026-10-26"`, cioè degli ultimi 10 giorni circa.
- Il join è `inner` su `idImpianto`. Il risultato ha circa 90.932 righe.
- Un impianto ha `Tipo Impianto` = "Pompe Bianche": è un errore, perché quello è un valore di `Bandiera`. Si corregge in "Stradale". Gli altri valori sono "Stradale" e "Autostradale".

## Problemi noti nei dati

- **Prezzi impossibili:** Blue Super a 0,100 € a Cortemaggiore (PC); HiQ Perform+ a 0,113 e 0,123 € ad Andezeno (TO).
- **Metano anomalo:** 4,999 € a Formigine (MO) e un prezzo tondo di 1,000 €. La metà centrale dei prezzi del metano sta tra 1,915 e 2,199 €.
- **Prezzi vecchi:** in `prezzi.csv` ci sono date dal 2013 e un massimo di 8,888 €. Con il filtro sulla data spariscono.
- **GPL:** costa intorno a 0,78 € (la metà centrale dei prezzi sta tra 0,76 e 0,80 €, il minimo è 0,61 €). È un prezzo plausibile, non un errore: nell'istogramma dei prezzi forma un picco separato da quello degli altri carburanti, intorno ai 2 €.
- **Soglie:** non sono ancora state applicate. Proposte: `prezzo >= 0.5` in generale, `between(1.2, 3.5)` per il metano.

## Esportazioni in Excel

Si esporta con `df.to_excel(..., index=False, freeze_panes=(1, 0))`. Serve `openpyxl`, che nel Python di sistema non è installato: se manca, nel notebook si installa con `%pip install openpyxl`.

File proposti finora:

- `dati_prezzi_carburanti.xlsx`: tutta la tabella `dati`.
- `distributori_metano.xlsx`: le righe dell'anagrafica degli impianti con almeno un prezzo "Metano", circa 1.499. Includendo GNL e L-GNC diventano 1.606; usando solo i prezzi recenti, 1.484.
- `prezzi_metano.xlsx`: le righe di `dati` con `descCarburante == "Metano"`, circa 1.552 da 1.484 impianti.
