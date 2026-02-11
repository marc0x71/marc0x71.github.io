+++
title = "Passare da C/C++ a Rust: la checklist invisibile"
date = 2026-02-02
description = "Dopo anni passati a scrivere codice in C e C++, il passaggio a Rust mi ha costretto a una riflessione che non mi aspettavo. Non riguarda la sintassi, le performance, o le feature del linguaggio. Riguarda qualcosa di più sottile: dove spendo le mie energie mentali mentre programmo"
[taxonomies]
tags = ["rust", "c", "cpp", "programing"]
[extra]
comments = true
+++

Dopo anni passati a scrivere codice in C e C++, il passaggio a Rust mi ha costretto a una riflessione che non mi aspettavo. Non riguarda la sintassi, le performance, o le feature del linguaggio. Riguarda qualcosa di più sottile: **dove spendo le mie energie mentali mentre programmo**.

## La checklist invisibile

Quando scrivi in C o C++, c'è una checklist che devi tenere sempre attiva nella testa. Non è scritta da nessuna parte, non appare in nessun warning, ma è lì:

- Questo puntatore potrebbe essere null?
- Ho liberato questa memoria? L'ho liberata due volte?
- Questa reference sarà ancora valida quando la userò?
- Questo cast è sicuro?
- Sto invocando undefined behavior senza saperlo?

È una fatica **invisibile**. Non la percepisci come ostacolo perché è diventata la normalità. È il sottofondo costante di ogni riga di codice che scrivi. E il problema più insidioso è che il feedback, quando arriva, arriva **tardi**: un crash in produzione, un bug report settimane dopo, un sanitizer che ti dice che quel codice che "funzionava" in realtà non funzionava affatto.

O peggio: non arriva mai. Il codice gira, sembra funzionare, ma sta invocando undefined behavior che per puro caso non si manifesta. Fino a quando non lo fa.

## Il "no" che arriva subito

Ammetto che i primi tempi con Rust non sono stati semplicissimi. Il borrow checker mi bloccava in continuazione. Cose che in C++ avrei scritto in trenta secondi richiedevano ripensamenti, ristrutturazioni, a volte riscritture complete dell'approccio.

Ma poi ho realizzato una cosa: quel "no" che arrivava subito era **un regalo**.

Quando il compilatore ti ferma, succede qualcosa di importante:

1. **Sei ancora nel contesto mentale del problema.** Non devi ricostruire settimane dopo cosa stavi pensando quando hai scritto quel codice.

2. **Puoi correggere mentre ragioni.** La soluzione è ancora fresca, le alternative sono ancora visibili.

3. **Impari perché quella cosa è problematica.** Il messaggio di errore (spesso sorprendentemente utile) ti spiega *cosa* non va e *perché*.

È la differenza tra un insegnante che ti corregge mentre parli e uno che ti dà un voto tre mesi dopo senza spiegarti dove hai sbagliato.

## Due tipi di "muscle memory"

C'è un'altra differenza che ho notato, più sottile.

In C++, la "muscle memory" che sviluppi nel tempo è di tipo **evitativo**. Impari a *non fare* certe cose. È una memoria negativa, non in senso dispregiativo, ma nel senso che devi ricordare cosa *non* fare invece di cosa fare: una lista di pattern da evitare, di trappole da schivare. E questa lista cresce continuamente, perché il linguaggio non ti impedisce di cadere nelle trappole, devi solo ricordarti che esistono.

In Rust, la muscle memory è di tipo **costruttivo**. Impari pattern che sono *intrinsecamente* corretti. Non devi ricordare cosa evitare perché il compilatore non ti permette di farlo. Invece di pensare "devo ricordarmi di non lasciare la porta aperta", impari a costruire una porta che si chiude da sola.

Questo libera spazio mentale. Spazio che posso usare per pensare alla logica del dominio, all'architettura, al problema che sto effettivamente cercando di risolvere, invece che a non inciampare nei miei stessi errori.

## Non è una questione di "meglio" o "peggio"

Non sto dicendo che C o C++ siano linguaggi sbagliati, o che chi li usa stia facendo un errore. Hanno i loro contesti d'uso legittimi, ecosistemi maturi, decenni di codice esistente che funziona.

La domanda che mi sono posto è diversa: **dove preferisco spendere le mie energie cognitive?**

Alcuni programmatori preferiscono la libertà totale e accettano la responsabilità che ne deriva. È una scelta legittima. Altri, e mi ci metto dentro, preferiscono vincoli più stretti che liberano bandwidth mentale per concentrarsi su altro.

Il costo c'è in entrambi i casi. In C++ è distribuito nel tempo, invisibile, perpetuo. In Rust è concentrato all'inizio, esplicito, e diminuisce man mano che interiorizzi i pattern.

Per me, il secondo trade-off è più favorevole. Non perché sia oggettivamente migliore, ma perché si adatta meglio al modo in cui funziona la mia testa, e al tipo di errori che tendo a fare.

## Una riflessione finale

La cosa che più mi ha sorpreso di Rust non è stata la velocità, o le feature del linguaggio, o l'ecosistema. È stata la **sensazione** di programmare.

Quella checklist invisibile che portavo nella testa da anni? Non c'è più. O meglio: è stata esternalizzata. Il compilatore la tiene per me.

E questo, alla fine, mi ha fatto capire quanto fosse pesante portarla.

