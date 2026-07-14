+++
title = "Vibe coding: il programma che non ho scritto"
date = 2026-07-14
description = "Ho lasciato che l'IA riscrivesse in cinque minuti un progetto a cui pensavo da anni. Ha funzionato benissimo. Ma qualcosa non tornava."
[taxonomies]
tags = ["ia", "coding"]
+++

## Il capo che ho sempre voluto essere

C'è un vecchio luogo comune sul capoufficio: quello che entra, dice "fatemi questa cosa", e sparisce fino a quando non è pronta. Non gli interessa il _come_, gli interessa il risultato. E se il risultato è buono, in fondo, il merito un po' se lo prende sempre anche lui.

Per anni ho guardato questa figura con un misto di fastidio e — diciamolo — un po' di invidia. Dev'essere comodo, no? Dire cosa vuoi e vederlo comparire.

Poi è arrivato il _vibe coding_, e improvvisamente quel capo potevo essere io.

## Il progettino che aspettava da anni

Avevo in un cassetto digitale — una di quelle cartelle che si aprono una volta all'anno, di solito con un misto di tenerezza e imbarazzo — un piccolo progetto scritto tempo fa. Python 2. Non Python 3, proprio Python 2, quello con `print` senza parentesi, per capirci quanto tempo fosse passato.

Funzionava. Anzi, funzionava benissimo — non è mai stato un problema di funzionalità. Era nato com'è nato: un prototipo buttato giù al volo, perché in quel momento mi serviva una soluzione e basta, non un'opera da tramandare ai posteri. Il classico script che risolve un problema preciso e poi resta lì, intoccato, perché "funziona, perché rischiare?".

Ma quel pensiero non se n'è mai andato del tutto: dargli una forma più solida, più _strutturata_, meno "scriptosa". Farlo diventare qualcosa che qualcuno potesse chiamare _programma_ senza virgolette ironiche. Magari riscriverlo in un linguaggio più adatto — Rust, C++ — qualcosa con un tipo di rigore che Python 2, per sua natura e per come l'avevo scritto io all'epoca, non aveva.

Ogni tanto ci pensavo. Poi arrivava qualcosa di nuovo, di più urgente, di più luccicante, e il progettino tornava nel cassetto. Come sempre.

## Cinque minuti (non cronometrati, ma ci giurerei)

Qualche giorno fa ho deciso che era il momento. Ho preso il codice, l'ho incollato in Claude Code, ho spiegato cosa volevo: riscrittura in C++.

Tre o quattro domande di chiarimento — quelle giuste, per inciso, non domande a caso — e poi silenzio operativo. Non ho cronometrato con precisione, ma per come l'ho vissuto non sono passati più di cinque minuti.

Il codice era pronto. Compilava. Funzionava. C'erano perfino i test.

Un lavoro che, onestamente, se l'avessi affrontato io da solo mi avrebbe portato via tre o quattro giorni — tra sintassi da riprendere in mano, gestione della memoria a cui ripensare con la testa giusta, i soliti errori di compilazione che si accumulano nelle prime ore di un porting — era già lì, finito, funzionante.

E qui viene la parte che non mi aspettavo: non ero felice. Ero triste.

## Non è la velocità il problema

Bisogna essere onesti: non è la prima volta che uso l'intelligenza artificiale nel mio lavoro, e nemmeno un'esperienza negativa in generale. La uso spesso, e in modi che considero sani: per valutare un'idea prima di buttarmici a capofitto, per farmi suggerire soluzioni alternative da vagliare con occhio critico, per liberarmi di quei compiti che definirei "dovuti ma noiosi" — la documentazione, gli unit test, quella parte del lavoro che sappiamo fare ma che non aspettiamo con ansia la domenica sera.

In quei casi l'ho sempre vissuta come il confronto con un collega esperto. Uno che magari non ti stupisce con idee rivoluzionarie — perché no, non è lì la sua forza — ma su cui puoi contare per i fondamentali, per le buone pratiche, per non reinventare male la ruota. Un collega con cui discuti, a cui chiedi "e se facessimo così?", che ti risponde, ti corregge, ti conferma. Un dialogo.

Il vibe coding è un'altra cosa. Non è un dialogo, è una delega totale. Dici cosa vuoi, e qualcuno — qualcosa — lo fa al posto tuo. Non tre domande e poi un lavoro insieme: tre domande e poi il compito sparisce dalle tue mani.

## Il codice che non sento mio

Ho ripensato a lungo a perché quella riscrittura in C++, tecnicamente ineccepibile, mi lasciasse questa sensazione strana. E credo di aver trovato una risposta, anche se parziale.

Quel codice non porta dentro nessuna delle mie tre giornate immaginarie di fatica. Non c'è dentro il pomeriggio passato a fissare uno stack trace incomprensibile. Non c'è la soddisfazione — piccola, ma vera — di capire finalmente perché quel puntatore stava puntando dove non doveva. Non c'è nemmeno l'imprecazione sommessa davanti a un errore di compilazione stupido, di quelli che si risolvono con una virgola.

Non c'è, insomma, la sofferenza. E mi sono reso conto — con un certo disagio, perché non è una conclusione comoda — che una parte del mio senso di possesso verso il codice che scrivo nasce proprio da lì. Dalla fatica. Dal tempo investito. Dagli errori fatti e corretti con le mie mani, letteralmente.

Un codice nato in cinque minuti, per quanto corretto, per quanto elegante, resta un oggetto estraneo. Funziona, ma non lo sento mio.

C'è anche da dire, per completezza, che non è stato tutto perfetto: alcune scelte architetturali fatte nella riscrittura non mi convincono fino in fondo — qualche astrazione che trovo eccessiva per la dimensione del progetto, qualche pattern applicato più per abitudine che per reale necessità. Sono proprio i punti su cui voglio tornare nella mia prossima sessione di vibe coding, provando a "discutere" con l'IA invece di limitarmi ad accettare il risultato.

E qui vale una precisazione, giusto per chiamare le cose con il loro nome: quello che ho fatto è vibe coding vero e proprio, anche stando alla distinzione netta che ne dà Antirez, il creatore di Redis, che riserva questo termine alla delega totale e chiama invece _automatic programming_ il lavoro in cui il programmatore guida costantemente le decisioni. Io non ho guidato niente. Ho chiesto, ho aspettato, ho ricevuto.

Ed è forse proprio questo il punto: non mi manca soffrire dietro a un puntatore che si comporta male, né passare un pomeriggio a fissare uno stack trace. Mi manca non aver costruito, passo dopo passo, la rappresentazione mentale del programma. È durante quel percorso che un software smette di essere semplicemente qualcosa che funziona e diventa qualcosa che sento mio.

Pensavo di voler essere quel capo che dice "fatemi questa cosa" e torna quando è pronta. Poi ci sono riuscito. E ho scoperto che la parte che mi piaceva davvero non era ricevere il programma. Era scriverlo.


