+++
title = "Rust, C e C++: oltre la guerra dei linguaggi"
date = 2026-02-11
description = "C'è un genere di contenuto che popola il web tecnologico con una regolarità quasi _liturgica_: il confronto tra linguaggi di programmazione. 'X vs Y: quale scegliere?', 'Perché ho abbandonato X per Y', 'X è morto, lunga vita a Y'. Titoli costruiti per generare reazioni, commenti infuocati, condivisioni indignate. E funziona. Funziona benissimo. Ma fermiamoci un momento a chiederci: a chi giova davvero questa narrazione? Non certo agli sviluppatori, che si ritrovano intrappolati in dibattiti sterili dove l'identità professionale si fonde con la scelta dello strumento. Non alle aziende, che hanno bisogno di decisioni pragmatiche basate su contesto e vincoli reali. Non alla crescita della comunità tech nel suo complesso."

# draft=true
[taxonomies]
tags = ["thoughts", "cpp", "c", "rust"]
[extra]
comments = true
+++

## Le guerre di religione dell'era digitale

C'è un genere di contenuto che popola il web tecnologico con una regolarità quasi _liturgica_: il confronto tra linguaggi di programmazione. "X vs Y: quale scegliere?", "Perché ho abbandonato X per Y", "X è morto, lunga vita a Y". Titoli costruiti per generare reazioni, commenti infuocati, condivisioni indignate.

E funziona. Funziona benissimo.

Ma fermiamoci un momento a chiederci: a chi giova davvero questa narrazione? Non certo agli sviluppatori, che si ritrovano intrappolati in dibattiti sterili dove l'identità professionale si fonde con la scelta dello strumento. Non alle aziende, che hanno bisogno di decisioni pragmatiche basate su contesto e vincoli reali. Non alla crescita della comunità tech nel suo complesso.

I veri vincitori sono le piattaforme. Ogni flame war su Reddit, ogni thread polemico su Twitter, ogni video YouTube dal titolo provocatorio genera traffico, engagement, tempo speso sulla piattaforma. E dove c'è attenzione, ci sono click pubblicitari. L'economia dell'attenzione si nutre di conflitto, e le guerre tra linguaggi sono conflitto a basso costo: appassionato, interminabile, e fondamentalmente innocuo per chi lo ospita.

**Queste sono le guerre di religione dell'era digitale**. E come le guerre di religione storiche, producono molto rumore, molta appartenenza tribale, e pochissima illuminazione.

Del resto, il mondo informatico è sempre stato terreno fertile per questo tipo di conflitti. Vim contro Emacs, Java contro C#, tabs contro spazi, Linux contro Windows, Angular contro React. La lista è lunga, e si allunga ogni anno. È quasi un pattern riconoscibile ormai: ogni volta che emergono due alternative valide per risolvere lo stesso problema, si formano fazioni, si alzano barricate, e parte **la guerra santa**.

Chi lavora nel settore da qualche anno riconosce il copione al primo atto. Eppure, puntualmente, ci ricaschiamo. Forse perché difendere i propri strumenti è un modo per difendere le proprie scelte, il proprio tempo investito, la propria identità professionale. 

Ma questa è psicologia, non ingegneria.

## Il problema della domanda sbagliata

"Quale linguaggio è migliore?" è una domanda mal posta. È come chiedere quale sia lo strumento migliore tra un martello e un cacciavite. La risposta sensata è sempre la stessa: dipende da cosa ci devi fare.

Ogni linguaggio di programmazione nasce in un momento storico specifico, per rispondere a problemi specifici, con una visione del mondo che riflette le priorità e i vincoli di quell'epoca. C nasce negli anni '70, quando le risorse computazionali erano scarsissime e serviva un'astrazione minimale sopra l'hardware. C++ nasce negli anni '80, quando la complessità del software crescente richiedeva strumenti di astrazione più potenti. Rust nasce nel 2010, quando decenni di vulnerabilità di sicurezza avevano reso evidente che certi tipi di errori andavano prevenuti a livello di linguaggio.

Nessuno di questi linguaggi è "migliore" degli altri. Ognuno rappresenta una risposta diversa a una domanda diversa, formulata in un contesto diverso.

## Tre filosofie, non tre concorrenti

Quello che trovo più interessante, e più utile, è studiare questi linguaggi come espressioni di filosofie diverse del problem-solving.

`C` incarna una fiducia radicale nel programmatore. Ti dà accesso diretto alla macchina, non ti nasconde nulla, e in cambio si aspetta che tu sappia esattamente cosa stai facendo. È un patto tra adulti: massima libertà, massima responsabilità. Questa filosofia ha prodotto sistemi operativi, database, e infrastrutture che reggono il mondo digitale ancora oggi da decenni.

`C++` parte dalla stessa base ma aggiunge l'ambizione dell'astrazione. Vuole permetterti di costruire concetti complessi (classi, gerarchie, template) mantenendo le prestazioni del codice vicino alla macchina. È un linguaggio che ha accumulato strumenti per trent'anni, diventando enormemente potente e, inevitabilmente, enormemente complesso.

`Rust` prende una strada diversa. Osserva i decenni di bug, vulnerabilità e comportamenti indefiniti che hanno afflitto C e C++, e decide che certe garanzie non possono essere lasciate alla disciplina del programmatore. Devono essere imposte dal compilatore. È una filosofia che sacrifica un po' di libertà immediata in cambio di sicurezza.

## Una riflessione sulla parentela

Qui arrivo a una considerazione personale che mi ha fatto riflettere. Spesso si confronta Rust con C++, probabilmente perché competono nello stesso dominio applicativo: sistemi ad alte prestazioni, browser, game engine, infrastrutture critiche.

Ma a livello di filosofia del dato, Rust mi sembra più vicino a C. Entrambi hanno un approccio quasi spartano: il dato è centrale, le operazioni agiscono sul dato, non ci sono gerarchie di classi o ereditarietà complesse. In Rust non esistono classi nel senso tradizionale dell'object-oriented. Esistono strutture e implementazioni, un modello mentale che un programmatore C riconoscerebbe immediatamente.

C++ invece ha abbracciato l'orientamento agli oggetti in modo profondo, con tutto ciò che ne consegue: ereditarietà multipla, funzioni virtuali, polimorfismo dinamico. È un paradigma potente, ma è anche un paradigma che Rust ha esplicitamente scelto di non adottare.

Questo non rende un approccio migliore dell'altro. Significa semplicemente che Rust e C condividono una certa visione del mondo, orientata al dato, trasparente, senza magia nascosta, che li distingue dal C++ più idiomatico.

## Acquisire valore invece di schierarsi

La mia filosofia è semplice: vivi e lascia vivere. Ogni linguaggio ha la sua ragion d'essere, la sua comunità, i suoi casi d'uso ideali. Schierarsi in una "fazione" significa precludersi la possibilità di imparare.

E c'è molto da imparare. Studiare come Rust gestisce la memoria mi ha reso più consapevole dei rischi che corro quando scrivo C++. Capire la semplicità di C mi ricorda di non over-engineerare soluzioni quando non serve. Apprezzare le astrazioni di C++ mi mostra pattern che posso portare in altri contesti.

I linguaggi di programmazione sono strumenti, si, ma sono anche cristallizzazioni di idee. Sono il risultato di persone intelligenti che hanno pensato a lungo su come risolvere problemi difficili. Ogni linguaggio contiene lezioni, se siamo disposti ad ascoltare invece di giudicare.

## Conclusione: oltre il tifo da stadio

La prossima volta che vedete un titolo che vi invita a schierarvi in una guerra tra linguaggi, fermatevi un momento. Chiedetevi chi beneficia davvero di quel click, di quel commento indignato, di quel tempo speso a difendere il vostro strumento preferito?

Il vero valore non sta nello scegliere il linguaggio "giusto". Sta nel capire perché esistono risposte diverse alla stessa domanda, e nell'arricchire il proprio modo di pensare attraverso questa comprensione.

I linguaggi passano, evolvono, a volte muoiono. 

Le idee che contengono, quelle restano.


