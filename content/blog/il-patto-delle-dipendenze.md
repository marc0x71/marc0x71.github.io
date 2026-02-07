+++
title = "Il patto delle dipendenze: quando diventa un Contratto col Diavolo"
date = 2026-01-28
description = "Ogni cargo add è un patto: comodità oggi, caos domani. Quando una delle 202 dipendenze transitive esplode venerdì alle 15, capisci il vero costo. Perché il problema non è la dipendenza, sei tu che non sai cosa hai importato"

[taxonomies]
tags = ["programmazione"]
+++
Ogni volta che scrivo `cargo add qualcosa`, mi piace pensare che sto semplicemente aggiungendo una funzionalità al mio progetto. È comodo, è veloce, è moderno. Ma se sono onesto con me stesso, quello che sto davvero facendo è firmare un contratto trilaterale: tra me, una libreria scritta da uno sconosciuto su internet, e l'utente finale che si fida ciecamente del mio software.

E come tutti i contratti che firmiamo senza leggere, le clausole scritte in piccolo prima o poi vengono a bussare alla porta.

## Il giorno in cui tutto va a puttane

Immaginati questa scena, che probabilmente hai già vissuto: sono le 3 del pomeriggio di venerdì. Il cliente ti chiama incazzato nero perché l'applicazione che hai deployato la settimana scorsa sta perdendo dati. Non pochi dati. Dati importanti. Dati che valgono soldi veri.

Tu apri il progetto, guardi il codice che hai scritto. È perfetto. Compila, i test passano, la logica è cristallina. Ma c'è un problema: il bug non è nel TUO codice. È in una delle quaranta dipendenze che hai nel Cargo.toml. O peggio ancora, in una delle centocinquanta dipendenze transitive che non sapevi nemmeno di avere.

Benvenuto nel vero costo delle dipendenze.

## Le Due Vie dell'Inevitabilità (o: Perché Siamo Tutti Condannati)

Quando aggiungiamo una dipendenza, fondamentalmente lo facciamo per due ragioni. E nessuna delle due è particolarmente confortante se ci pensi bene.

**Prima ragione: non so come fare quella cosa.**

Va bene, lo ammetto. Non so scrivere un parser HTTP sicuro. Non so implementare un sistema crittografico che non sia vulnerabile a timing attacks. Non so gestire la compressione di file senza perdere pezzi per strada. E quindi uso reqwest, uso ring, uso flate2.

È una scelta saggia? Certo che sì. Solo un idiota cercherebbe di reinventare la crittografia. Ma ecco il punto: se non so come fare quella cosa, come faccio a debuggare quando quella cosa si rompe? Come faccio a capire se il problema è nel mio codice o nella dipendenza? Come faccio a valutare se quel panic che vedo è un bug loro o un uso scorretto mio?

La risposta è: posso, ma solo fino a un certo punto, e il costo mentale sale vertiginosamente. Certo, posso leggere stack trace, testare input diversi, aprire issue su GitHub sperando che qualcuno risponda. Ma se il problema è veramente dentro quella dipendenza, devo comunque tuffarmi nel codice sorgente e cercare di capire cosa diavolo sta succedendo. E indovina quando potrei dover fare tutto questo? Esatto: alle 3 del pomeriggio di venerdì, con il cliente che urla e una deadline che si avvicina.

**Seconda ragione: potrei farlo, ma non ho voglia.**

"Perché dovrei scrivere un generatore di UUID quando esiste già una crate per quello?"

"Perché dovrei implementare un serializzatore JSON quando serde fa tutto così bene?"

"Perché dovrei perdere tempo con un template engine quando posso usare tera?"

Risposta breve: perché forse quel tempo "perso" ora ti avrebbe risparmiato una settimana di inferno dopo.

Il paradosso è straordinario: eviti di scrivere qualcosa perché "ti ruberebbe troppo tempo", ma quando quella dipendenza si rompe, sei costretto a capirla lo stesso. Solo che ora lo devi fare:

- Sotto pressione
- Con il cliente che ti bombarda di email
- Cercando di orientarti in un codice scritto da altri, con convenzioni diverse dalle tue
- Con la concreta possibilità che dovrai comunque riscrivere tutto da zero perché la dipendenza è abandonware o il maintainer non risponde

Congratulazioni: hai risparmiato due giorni tre mesi fa per perderne sette adesso.

## Il Vero Problema Non È la Dipendenza, Sei Tu

Lo so, sembra duro. Ma lasciami spiegare.

Il punto non è che usare dipendenze sia male. Il punto è che le trattiamo come soluzioni magiche invece che come quello che realmente sono: codice scritto da esseri umani fallibili che potrebbe esploderti in faccia in qualsiasi momento.

Quante volte hai aggiunto una dipendenza senza nemmeno aprire il repository per vedere com'è fatto il codice? Quante volte hai guardato solo il numero di download su crates.io pensando "vabbè, se lo usano in tanti sarà affidabile"? Quante volte hai pensato "tanto se c'è un problema faccio un issue su GitHub e qualcuno lo fixerà"?

Ecco, forse la vera incompetenza non è non sapere scrivere una libreria HTTP. Forse la vera incompetenza è delegare a sconosciuti su internet pezzi critici della nostra applicazione senza nemmeno sapere come funzionano.

Quando aggiungi una dipendenza, non stai solo aggiungendo funzionalità. Stai aggiungendo:

- **Un punto di failure** che non controlli
- **Una superficie d'attacco** per bug che non hai scritto tu
- **Un debito di comprensione** che prima o poi dovrai pagare
- **Una catena di dipendenze transitive** che potrebbero essere decine o centinaia
- **Una responsabilità verso il tuo utente finale** che hai delegato a uno sconosciuto

E la cosa folle? Spesso non lo sappiamo nemmeno. Aggiungi una dipendenza pensando che sia "solo quella", ma in realtà stai portandoti dietro un intero ecosistema. Quel crate carino da 500 righe che hai aggiunto? Ha quindici dipendenze. Quelle quindici ne hanno altre trenta. E improvvisamente ti ritrovi con centinaia di migliaia di righe di codice di cui sei responsabile ma che non hai mai letto.

## Supply Chain Attacks: Quando il Pericolo Viene da Dentro

Parliamoci chiaro: quando aggiungi una dipendenza, stai letteralmente eseguendo codice scritto da uno sconosciuto su internet. Nella tua macchina. Con i tuoi permessi. Durante la compilazione, durante il runtime, sempre.

E non sto parlando di scenari paranoici da film. Negli ultimi anni abbiamo visto di tutto: dipendenze microscopiche rimosse che hanno mandato in crash interi ecosistemi. Maintainer che, stufi di lavorare gratis per multinazionali miliardarie, hanno deliberatamente introdotto bug nei loro progetti come forma di protesta. Attaccanti che hanno preso il controllo di progetti legittimi e hanno iniettato malware per rubare credenziali o criptovalute.

"Ma quello succede in altri linguaggi", dirai tu. "Noi in Rust siamo diversi, abbiamo il borrow checker, abbiamo la safety!"

Certo. Il borrow checker ti protegge dai data race. Ma dimmi: ti protegge da un build.rs malevolo che, durante la compilazione, esfiltra le tue chiavi SSH? Ti protegge da una dipendenza che decide di mandare telemetria dei tuoi dati a un server sconosciuto? Ti protegge da un maintainer che, esausto dal lavoro non pagato, vende il controllo del progetto al miglior offerente senza dire niente a nessuno?

La risposta è no. Il compilatore non può aiutarti se il problema non è il codice unsafe, ma le intenzioni di chi l'ha scritto.

La supply chain del software è il Far West della sicurezza informatica. Ogni dipendenza è una porta aperta. E il bello (o il brutto, dipende dai punti di vista) è che spesso non ci pensiamo nemmeno. Aggiungiamo una crate perché "ha tante stelle su GitHub" o "tanti download su crates.io", come se la popolarità fosse garanzia di sicurezza o integrità.

Ma la popolarità non ti dice nulla su chi mantiene quel progetto, su quanto sia motivato a continuare, su quanto sia resistente alla corruzione o alla stanchezza. Ogni dipendenza che aggiungi è un potenziale cavallo di Troia. E la cosa più spaventosa? Probabilmente non te ne accorgeresti nemmeno finché non è troppo tardi.

## L'Abbandono: La Tragedia Silenziosa

Poi c'è lo scenario più sottile e più comune: l'abbandono.

Trovi una crate perfetta. Fa esattamente quello che ti serve. Il codice è pulito, i test passano, la documentazione è buona. La aggiungi al progetto. Tutto funziona. Per sei mesi, un anno, due anni.

Poi un giorno apri una issue perché hai trovato un edge case. Nessuna risposta. Ne apri un'altra. Silenzio. Guardi la data dell'ultimo commit: tre anni fa. Guardi gli altri issue aperti: decine, tutti senza risposta.

Il maintainer è sparito. Forse ha cambiato lavoro, forse ha perso interesse, forse è semplicemente bruciato dal dover mantenere gratis un progetto che usano migliaia di aziende che non contribuiscono mai.

E ora? La community Rust ha iniziato a discutere di policy per le crate abbandonate, di come passare la manutenzione in modo più fluido, di come gestire questi casi. Ma sul campo, quando ti capita davvero, il risultato pratico è semplice: o forki il progetto e te lo mantieni da solo (con tutto il lavoro che comporta), o lo sostituisci (e buona fortuna a trovare qualcosa di compatibile), o lo riscrivi da zero (quello che avresti potuto fare dall'inizio, se solo non fossi stato "pigro").

In ogni caso, sei TU che ti ritrovi a gestire il problema.

## Il Patto Faustiano

Mefistofele offre a Faust la conoscenza universale in cambio della sua anima. Le dipendenze ci offrono potenza immediata in cambio del nostro controllo futuro.

È un deal che facciamo costantemente, spesso inconsapevolmente. Cargo add e via, abbiamo appena barattato due ore di sviluppo oggi contro potenziali giorni o settimane di debugging domani.

Non sto dicendo che dovremmo smettere di usare dipendenze. Sarebbe stupido e controproducente. Ma sto dicendo che dovremmo smettere di trattarle come soluzioni gratuite e senza conseguenze.

Ogni dipendenza ha un costo. Il fatto che questo costo non si manifesti immediatamente non significa che non esista. È come il debito con gli interessi: più lo ignori, più crescono.

## Quindi? Che Facciamo?

Non ho una soluzione magica. Se l'avessi, probabilmente la pacchettizzerei in una crate e la metterei su crates.io, creando una meta-dipendenza infinitamente ironica.

Ma ho qualche pensiero:

**Primo**: smetti di aggiungere dipendenze in automatico. Ogni volta che stai per scrivere cargo add, fermati e chiediti: "Ho davvero bisogno di questo? Potrei scriverlo io in tempo ragionevole? Se si rompe, sono almeno in grado di orientarmi nel codice per capire cosa sta succedendo?".

Attenzione: non sto dicendo che devi saper reimplementare tutto. Ci sono dipendenze critiche - crypto, TLS, database driver, HTTP stack - che nessuno dovrebbe mai provare a scrivere da zero, ma che sono mantenute da team solidi, con auditing costante e community ampie. Il punto non è "se non puoi fixarla non usarla", ma "se nessuno nel team è in grado neanche di orientarsi in quel codice in caso di emergenza, forse la stai usando in modo troppo fiducioso".

**Secondo**: se aggiungi una dipendenza, aprila e leggila. Non tutta, non sei pazzo. Ma almeno i file principali, le API pubbliche, le decisioni architetturali fondamentali. Impiega un pomeriggio a capire come funziona. Quel pomeriggio potrebbe risparmiarti una settimana di inferno in futuro.

**Terzo**: nel tuo team, ogni dipendenza dovrebbe avere un "owner mentale". Qualcuno che l'ha letta, che la capisce, che potrebbe forkare e patchare se necessario. Se nessuno vuole assumersi questa responsabilità, forse quella dipendenza è un problema.

**Quarto**: preferisci dipendenze piccole, focalizzate, leggibili a mostri monolitici. In Rust siamo fortunati: l'ecosistema tende verso crate piccole e composibili. Sfrutta questa filosofia. Una dipendenza di 500 righe è infinitamente più gestibile di una di 50.000.

**Quinto**: ricordati sempre che nel mondo Rust, dove il compilatore ci protegge da tanti errori, le dipendenze rimangono l'ultimo regno del caos. Il borrow checker non può aiutarti se il problema è in codice che non hai scritto tu e che non capisci.

## La Vera Domanda

Alla fine, la domanda non è "Posso evitare questa dipendenza?", ma "Sono disposto a pagare il prezzo quando verrà il momento?".

Perché il momento arriverà. Forse tra un mese, forse tra un anno, forse tra cinque. Ma arriverà. E quando arriverà, sarai solo tu, il tuo codice, e un bug in mezzo a migliaia di righe scritte da qualcun altro.

In quel momento, capirai il vero costo delle dipendenze. Non sono i megabyte di disco che occupano o i millisecondi in più di compilazione. È la perdita di controllo. È la delega di responsabilità. È il debito tecnico travestito da comodità.

Ogni volta che aggiungi una dipendenza, stai facendo una scommessa: scommetti che il tempo che risparmi ora vale più del dolore potenziale futuro. A volte è una buona scommessa. Spesso no.

Il progetto zero-dipendenze è un estremismo inutile. Ma il progetto che tira dentro tutto Crates.io "perché tanto è gratis" è altrettanto folle.

La virtù sta nel mezzo consapevole. Nel riconoscere che ogni dipendenza è un patto faustiano, e nel decidere deliberatamente quali patti siamo disposti a fare.

Perché alla fine, quando il tuo software esploderà (e lo farà, è solo questione di tempo), la domanda non sarà "Di chi è la colpa?". La domanda sarà "Tu sapevi cosa stavi importando?".

E se la risposta è no, forse il problema non era la dipendenza.

Eri tu.

---

_P.S.: Dopo aver scritto questo articolo, ho controllato le dipendenze del mio ultimo progetto. 6 dirette, 202 transitive. Non so nemmeno cosa facciano la metà di queste. Forse dovrei cominciare a praticare quello che predico. O forse no. Dopotutto, che potrebbe mai andare storto?_
