+++
title = "Arena Pattern in Rust: gestire riferimenti ciclici con gli indici"
date = 2026-02-25
description = "C'è un rito di passaggio che ogni programmatore Rust deve affrontare: scrivere una doubly linked list. In C++ è un esercizio da prima elementare. In Python ci vogliono tre righe di codice. In Rust, è il momento in cui molti principianti sbattono la testa contro un muro di mattoni: il borrow checker. Improvvisamente, concetti semplici come \"il nodo A punta a B e B punta ad A\" diventano un incubo di lifetime, RefCell, e panic a runtime. Ti senti stupido per non riuscire a scrivere una struttura dati che un programmatore C scriveva a occhi chiusi nel 1985. La buona notizia? Non sei tu il problema, è il modello mentale. In questo articolo smetteremo di combattere il compilatore e impareremo a risolvere il problema alla radice, usando l'Arena Pattern."

[taxonomies]
tags = ["rust", "pattern"]
+++

C'è un rito di passaggio che ogni programmatore Rust deve affrontare: scrivere una doubly linked list.

In C++ è un esercizio da prima elementare. In Python ci vogliono tre righe di codice. In Rust, è il momento in cui molti principianti sbattono contro un muro di mattoni: il borrow checker. Improvvisamente, concetti semplici come "il nodo A punta a B e B punta ad A" diventano un incubo di lifetime, `RefCell`, e panic a runtime. Ti senti stupido per non riuscire a scrivere una struttura dati che un programmatore C scriveva a occhi chiusi nel 1985.

La buona notizia? Non sei tu il problema, è il modello mentale. In questo articolo smetteremo di combattere il compilatore e impareremo a risolvere il problema alla radice, usando l'**Arena Pattern**.

---

## Ma partiamo dall'inizio

In Python, la doubly linked list di cui parliamo si scrive così:

```python
class Node:
    def __init__(self, value, next=None, prev=None):
        self.value = value
        self.next = next
        self.prev = prev
```

Nessun problema di ownership, nessun lifetime, nessun compilatore che ti blocca. I riferimenti sono semplicemente puntatori, e il garbage collector si occupa di tutto.

Anche in Rust, una **singly linked list** è fattibile senza troppi problemi. La versione ricorsiva classica con `Box` funziona:

```rust
enum List<T> {
    Nil,
    Cons(T, Box<List<T>>),
}
```

L'ownership è chiara: ogni nodo possiede il successivo tramite `Box`. Una catena lineare, senza ambiguità. Il compilatore è soddisfatto e tutto fila liscio.

Ma cosa succede quando vogliamo andare **anche indietro**?

---

## Il muro: la doubly linked list

Proviamo l'approccio più naturale, ogni nodo ha un puntatore `next` e un puntatore `prev`:

```rust
struct Node<T> {
    value: T,
    next: Option<Box<Node<T>>>,
    prev: Option<Box<Node<T>>>,
}
```

La struct compila, ma `Box` implica ownership esclusiva, quindi la struttura risultante è un albero, non un grafo. Una doubly linked list richiede ownership condivisa o puntatori non-owning; con soli `Box` non è possibile rappresentare correttamente i collegamenti bidirezionali senza violare le regole di ownership.

Proviamo allora con i riferimenti:

```rust
struct Node<'a, T> {
    value: T,
    next: Option<&'a mut Node<'a, T>>,
    prev: Option<&'a mut Node<'a, T>>,
}
```

Anche qui la definizione compila, ma provare a usarla è dove tutto crolla, e per ragioni più profonde di quanto sembri a prima vista. Il primo problema è tecnico: non puoi avere due `&mut` allo stesso nodo (uno dal predecessore, uno dal successore), e i lifetime diventano un incubo ricorsivo.

Ma c'è un problema ancora più fondamentale: **dove vivono i nodi?** Un riferimento `&'a Node` non possiede nulla, punta a qualcosa che deve già esistere da qualche parte. E quel "qualche parte" è il vero nodo della questione.

Se li allochi sullo stack, devi dichiararli tutti prima di collegarli, il che rende impossibile costruire la lista dinamicamente:

```rust
// Ogni nodo è una variabile locale... non scala
let mut node_a = Node { value: 1, next: None, prev: None };
let mut node_b = Node { value: 2, next: None, prev: None };
// Come collego node_b.prev a node_a se ho già preso &mut node_a per next?
```

Se li allochi sullo heap con `Box`, ricadi nel problema dell'ownership esclusiva: il `Box` possiede il nodo, e non puoi avere un riferimento `prev` a qualcosa posseduto dal `Box` di un altro nodo senza violare le regole del borrow checker.

È un cortocircuito logico: per usare i riferimenti, i nodi devono esistere da qualche parte, ma ogni "qualche parte" in Rust ha regole di ownership che confliggono con i riferimenti bidirezionali.

Il problema di fondo è che la doubly linked list ha **riferimenti bidirezionali**: ogni nodo è puntato da due direzioni. In Rust, dove l'ownership è esclusiva e i lifetime sono rigorosi, questa è una contraddizione fondamentale.

---

## La soluzione classica: `Rc<RefCell<T>>`

La risposta tradizionale di Rust ai riferimenti condivisi e mutabili è la coppia `Rc<RefCell<T>>`:

```rust
use std::cell::RefCell;
use std::rc::{Rc, Weak};

struct Node<T> {
    value: T,
    next: Option<Rc<RefCell<Node<T>>>>,
    prev: Option<Weak<RefCell<Node<T>>>>,
}
```

`Rc` (Reference Counted) permette ownership condivisa: più parti del codice possono possedere lo stesso nodo. `RefCell` sposta il borrow checking dal compile-time al runtime, permettendo mutabilità interna. `Weak` è una versione debole di `Rc` che non contribuisce al conteggio dei riferimenti, necessaria per il puntatore `prev` per evitare cicli di riferimento che causerebbero memory leak.

Funziona, ma a quale prezzo? Guardiamo un'operazione semplice, inserire un nuovo nodo dopo uno esistente:

```rust
fn insert_after<T>(node: &Rc<RefCell<Node<T>>>, value: T) {
    let new_node = Rc::new(RefCell::new(Node {
        value,
        next: None,
        prev: None,
    }));

    let mut node_borrow = node.borrow_mut();
    let old_next = node_borrow.next.take();

    // Collega new_node -> old_next
    if let Some(ref old_next_rc) = old_next {
        old_next_rc.borrow_mut().prev = Some(Rc::downgrade(&new_node));
    }
    new_node.borrow_mut().next = old_next;

    // Collega node -> new_node
    new_node.borrow_mut().prev = Some(Rc::downgrade(node));
    node_borrow.next = Some(new_node);
}
```

Per un'operazione che concettualmente è "aggiungi un nodo tra due nodi e aggiorna quattro puntatori", il codice è sorprendentemente denso. Ma il vero problema non è la verbosità, sono i trade-off nascosti:

- **Overhead runtime**: ogni `Rc::clone()` e ogni `drop` incrementa/decrementa un contatore atomico. Ogni `borrow()` e `borrow_mut()` effettua un controllo a runtime.
- **Panic a runtime**: se chiami `borrow_mut()` mentre un `borrow()` è già attivo sullo stesso `RefCell`, il programma va in panic. Hai spostato un errore di compilazione in un errore a runtime, un passo indietro.
- **Memory leak silenzioso**: se dimentichi di usare `Weak` per `prev` e usi `Rc`, crei un ciclo di riferimenti che non verrà mai deallocato. Nessun warning, nessun errore, solo memoria che cresce.
- **Fallibilità di `Weak`**: `prev.upgrade()` restituisce `Option<Rc<...>>` perché il nodo potrebbe essere già stato deallocato. Devi gestire questo caso ovunque.

C'è qualcosa di profondamente insoddisfacente in tutto questo. Una delle promesse fondamentali di Rust è liberare il programmatore dalla *checklist invisibile* che chi viene da C/C++ conosce bene: quel sottofondo costante di domande difensive ("questo puntatore è null?", "ho liberato questa memoria?", "questa reference è ancora valida?") che drena energie mentali che dovrebbero essere dedicate alla logica del problema. Con `Rc<RefCell<T>>` stiamo reintroducendo una checklist invisibile dentro Rust: ricordati di usare `Weak` per i riferimenti all'indietro, non fare `borrow_mut()` mentre un `borrow()` è attivo, gestisci `upgrade()` che può fallire. Le nostre energie tornano a essere spese in difesa, non in costruzione. (Per un approfondimento su questo tema, consiglio [Passare da C/C++ a Rust: la checklist invisibile](https://marc0x71.github.io/blog/checklist-invisibile/) che esplora la differenza tra la *muscle memory evitativa* del C++ e quella *costruttiva* di Rust.)

Esiste un modo migliore?

---

## Cambiamo prospettiva: l'Arena Pattern

Facciamo un passo indietro e riflettiamo. Il problema fondamentale con la doubly linked list non è la struttura dati in sé, è che stiamo chiedendo a **ogni nodo di gestire la proprietà dei suoi vicini**. E se rovesciassimo il modello? E se **nessun nodo possedesse nessun altro nodo**?

Questa è l'intuizione alla base dell'Arena Pattern: una struttura centrale, l'*arena*, possiede tutti i nodi. I nodi non si possiedono tra loro; si **riferiscono** l'un l'altro tramite semplici indici numerici nell'arena.

```rust
type NodeId = usize;

struct Node<T> {
    value: T,
    next: Option<NodeId>,
    prev: Option<NodeId>,
}

struct LinkedList<T> {
    arena: Vec<Node<T>>,
    head: Option<NodeId>,
    tail: Option<NodeId>,
}
```

Nessun lifetime, nessun `Rc`, nessun `RefCell`. Un `NodeId` è semplicemente un `usize`, un indice nel `Vec` che funge da arena. E `usize` è `Copy`, non ha lifetime, non crea borrow. Può essere passato, copiato, memorizzato ovunque senza che il borrow checker abbia nulla da ridire.

Rivediamo l'operazione `insert_after` con questo approccio:

```rust
impl<T> LinkedList<T> {
    fn new() -> Self {
        LinkedList {
            arena: Vec::new(),
            head: None,
            tail: None,
        }
    }

    fn push_back(&mut self, value: T) -> NodeId {
        let new_id = self.arena.len();
        self.arena.push(Node {
            value,
            prev: self.tail,
            next: None,
        });

        if let Some(tail_id) = self.tail {
            self.arena[tail_id].next = Some(new_id);
        } else {
            self.head = Some(new_id);
        }

        self.tail = Some(new_id);
        new_id
    }

    fn insert_after(&mut self, node_id: NodeId, value: T) -> NodeId {
        let new_id = self.arena.len();
        let old_next = self.arena[node_id].next;

        self.arena.push(Node {
            value,
            prev: Some(node_id),
            next: old_next,
        });

        self.arena[node_id].next = Some(new_id);

        if let Some(next_id) = old_next {
            self.arena[next_id].prev = Some(new_id);
        } else {
            self.tail = Some(new_id);
        }

        new_id
    }
}
```

Il codice è lineare, leggibile, e fa esattamente ciò che dice. Nessuna cerimonia, nessun overhead a runtime, nessun rischio di panic da `borrow_mut()`. Il compilatore è felice e anche noi.

---

## Costruire un'Arena generica

Estraiamo il pattern in una struttura riutilizzabile. Nella sua forma più semplice, un'arena è un `Vec` con un metodo di allocazione:

```rust
struct Arena<T> {
    items: Vec<T>,
}

impl<T> Arena<T> {
    fn new() -> Self {
        Arena { items: Vec::new() }
    }

    fn alloc(&mut self, value: T) -> usize {
        let id = self.items.len();
        self.items.push(value);
        id
    }

    fn get(&self, id: usize) -> &T {
        &self.items[id]
    }

    fn get_mut(&mut self, id: usize) -> &mut T {
        &mut self.items[id]
    }
}
```

Allocare è O(1) ammortizzato (come un `Vec::push`). Accedere è O(1) (un accesso diretto all'array). Semplice, veloce, prevedibile.

Ma questa arena ha un limite importante: è **append-only**. Una volta allocato un elemento, non lo puoi rimuovere. Per molti casi d'uso (AST di un compilatore, nodi di un parser) questo è perfettamente accettabile: allochi tutto durante una fase, usi tutto, e poi scarti l'intera arena. Ma per la nostra linked list, dove vogliamo poter rimuovere nodi, serve qualcosa in più.

---

## La rimozione: primo approccio con la free list

Il problema della rimozione è più sottile di quanto sembri. La prima cosa che viene in mente è usare `Vec::remove(index)`, ma questo compatta il vettore spostando tutti gli elementi successivi indietro di una posizione. Il nodo che era all'indice 4 ora è al 3, quello al 5 è al 4, e così via: tutti gli ID conservati altrove diventano sbagliati. In una linked list, dove ogni nodo memorizza gli indici dei suoi vicini, sarebbe un disastro.

L'alternativa è non rimuovere davvero l'elemento, ma sostituirlo con un segnaposto (`None`) lasciando un "buco" nel vettore. Gli indici restano stabili, ma i buchi si accumulano e le allocazioni successive vanno comunque in coda al `Vec`, senza riusare lo spazio libero. Col tempo l'arena cresce senza motivo.

La soluzione è mantenere una **free list**, un vettore ausiliario che tiene traccia degli indici degli slot liberi:

```rust
struct Arena<T> {
    items: Vec<Option<T>>,
    free_list: Vec<usize>,
}

impl<T> Arena<T> {
    fn new() -> Self {
        Arena {
            items: Vec::new(),
            free_list: Vec::new(),
        }
    }

    fn alloc(&mut self, value: T) -> usize {
        if let Some(index) = self.free_list.pop() {
            self.items[index] = Some(value);
            index
        } else {
            let index = self.items.len();
            self.items.push(Some(value));
            index
        }
    }

    fn remove(&mut self, id: usize) -> Option<T> {
        let value = self.items[id].take();
        if value.is_some() {
            self.free_list.push(id);
        }
        value
    }

    fn get(&self, id: usize) -> Option<&T> {
        self.items.get(id)?.as_ref()
    }
}
```

Funziona. Quando allochi, prima controlli se c'è uno slot libero nella free list; se sì lo riusi, altrimenti espandi il `Vec`. Quando rimuovi, metti `None` nello slot e aggiungi l'indice alla free list.

Ma c'è un'ineleganza: stai mantenendo **due strutture separate**, il `Vec<Option<T>>` per i dati e il `Vec<usize>` per gli slot liberi. Gli slot vuoti contengono `None`, occupando comunque spazio per `Option<T>`, e nel frattempo il loro indice è duplicato nella free list. Possiamo fare di meglio.

---

## La free list intrusiva: un'arena più elegante

L'idea chiave: invece di tenere la lista degli slot liberi in un vettore separato, **codifichiamola direttamente dentro l'arena**. Ogni slot è un'enum: o contiene un valore (`Occupied`), o è vuoto e punta al prossimo slot vuoto (`Vacant`). Gli slot vuoti formano una linked list intrusiva dentro l'arena stessa.

```rust
enum Slot<T> {
    Occupied(T),
    Vacant { next_free: Option<usize> },
}

struct Arena<T> {
    slots: Vec<Slot<T>>,
    head_free: Option<usize>,
    len: usize,
}
```

Lo spazio che un nodo rimosso lascia vuoto non viene sprecato: contiene il puntatore al prossimo slot libero. Zero overhead.

### Implementazione completa

```rust
enum Slot<T> {
    Occupied(T),
    Vacant { next_free: Option<usize> },
}

struct Arena<T> {
    slots: Vec<Slot<T>>,
    head_free: Option<usize>,
    len: usize,
}

impl<T> Arena<T> {
    /// Crea un'arena vuota
    fn new() -> Self {
        Arena {
            slots: Vec::new(),
            head_free: None,
            len: 0,
        }
    }

    /// Crea un'arena con `capacity` slot pre-allocati.
    /// Tutti gli slot iniziano come Vacant e formano una catena.
    fn with_capacity(capacity: usize) -> Self {
        let mut slots = Vec::with_capacity(capacity);
        for i in 0..capacity {
            let next = if i + 1 < capacity { Some(i + 1) } else { None };
            slots.push(Slot::Vacant { next_free: next });
        }
        Arena {
            slots,
            head_free: if capacity > 0 { Some(0) } else { None },
            len: 0,
        }
    }

    /// Alloca un valore nell'arena, restituisce il suo ID.
    fn alloc(&mut self, value: T) -> usize {
        self.len += 1;

        if let Some(index) = self.head_free {
            let next = match self.slots[index] {
                Slot::Vacant { next_free } => next_free,
                Slot::Occupied(_) => unreachable!(),
            };
            self.slots[index] = Slot::Occupied(value);
            self.head_free = next;
            index
        } else {
            let index = self.slots.len();
            self.slots.push(Slot::Occupied(value));
            index
        }
    }

    /// Rimuove un valore dall'arena, restituendolo.
    fn remove(&mut self, id: usize) -> Option<T> {
        let slot = self.slots.get_mut(id)?;
        match std::mem::replace(
            slot,
            Slot::Vacant { next_free: self.head_free },
        ) {
            Slot::Occupied(value) => {
                self.head_free = Some(id);
                self.len -= 1;
                Some(value)
            }
            vacant @ Slot::Vacant { .. } => {
                // Era già vuoto, ripristina
                *slot = vacant;
                None
            }
        }
    }

    /// Accesso immutabile a un valore.
    fn get(&self, id: usize) -> Option<&T> {
        match self.slots.get(id)? {
            Slot::Occupied(value) => Some(value),
            Slot::Vacant { .. } => None,
        }
    }

    /// Accesso mutabile a un valore.
    fn get_mut(&mut self, id: usize) -> Option<&mut T> {
        match self.slots.get_mut(id)? {
            Slot::Occupied(value) => Some(value),
            Slot::Vacant { .. } => None,
        }
    }

    /// Numero di elementi attivi nell'arena.
    fn len(&self) -> usize {
        self.len
    }
}
```

---

## I limiti: gli stale ID

L'arena con la free list intrusiva è elegante e performante, ma ha un limite importante da conoscere. Quando uno slot viene rimosso e poi riusato, un vecchio indice conservato da qualche parte nel codice punta ora a un **oggetto completamente diverso**:

```rust
let mut arena = Arena::new();

let id_carlo = arena.alloc("Carlo");   // id = 0
arena.remove(id_carlo);                // slot 0 → Vacant
let id_diana = arena.alloc("Diana");   // riusa slot 0, id = 0

// ⚠️ id_carlo è ancora 0, ma ora punta a "Diana"!
println!("{:?}", arena.get(id_carlo)); // Some("Diana"), BUG silenzioso!
```

È l'equivalente logico di un **use-after-free**: l'ID è formalmente valido (lo slot è `Occupied`), ma si riferisce a un oggetto diverso da quello originale. Nessun panic, nessun errore, solo dati corrotti.

Per scenari dove la rimozione è frequente e gli ID vengono conservati a lungo, esistono soluzioni come il **generational index** (ogni slot ha un contatore di "generazione" che invalida i vecchi ID) e il **newtype pattern con PhantomData** (per impedire di mischiare ID di arene diverse a compile-time). Ma questi raffinamenti meritano un articolo dedicato; per ora, è sufficiente essere consapevoli del rischio e adottare buone pratiche: non conservare ID dopo una `remove`, o verificare sempre il risultato di `get` che restituisce `Option`.

> **Nota a margine**: il pattern Occupied/Vacant che abbiamo costruito non è limitato alle strutture dati in-memory tradizionali. Lo stesso approccio si applica, ad esempio, alla gestione di record in shared memory per la comunicazione inter-processo, dove una tabella pre-allocata con slot occupati e vuoti permette inserimenti, ricerche e rimozioni O(1) in un contesto dove le allocazioni dinamiche non sono possibili. Ho usato questo stesso pattern in [inmemorytable-rs](https://github.com/marc0x71/inmemorytable-rs), una libreria che ho sviluppato per la gestione di record tipizzati in POSIX shared memory.

---

## Mettere tutto insieme: la doubly linked list con l'Arena

Torniamo al nostro punto di partenza. Con l'arena a free list intrusiva, possiamo finalmente scrivere una doubly linked list completa, con inserimento e rimozione, senza `Rc`, senza `RefCell`, senza lifetime:

```rust
type NodeId = usize;

struct ListNode<T> {
    value: T,
    prev: Option<NodeId>,
    next: Option<NodeId>,
}

struct DoublyLinkedList<T> {
    arena: Arena<ListNode<T>>,
    head: Option<NodeId>,
    tail: Option<NodeId>,
}

impl<T> DoublyLinkedList<T> {
    fn new() -> Self {
        DoublyLinkedList {
            arena: Arena::new(),
            head: None,
            tail: None,
        }
    }

    fn push_back(&mut self, value: T) -> NodeId {
        let node = ListNode {
            value,
            prev: self.tail,
            next: None,
        };
        let id = self.arena.alloc(node);

        if let Some(tail_id) = self.tail {
            self.arena.get_mut(tail_id).unwrap().next = Some(id);
        } else {
            self.head = Some(id);
        }

        self.tail = Some(id);
        id
    }

    fn push_front(&mut self, value: T) -> NodeId {
        let node = ListNode {
            value,
            prev: None,
            next: self.head,
        };
        let id = self.arena.alloc(node);

        if let Some(head_id) = self.head {
            self.arena.get_mut(head_id).unwrap().prev = Some(id);
        } else {
            self.tail = Some(id);
        }

        self.head = Some(id);
        id
    }

    fn remove(&mut self, id: NodeId) -> Option<T> {
        let node = self.arena.get(id)?;
        let prev = node.prev;
        let next = node.next;

        match prev {
            Some(prev_id) => {
                self.arena.get_mut(prev_id).unwrap().next = next;
            }
            None => self.head = next,
        }

        match next {
            Some(next_id) => {
                self.arena.get_mut(next_id).unwrap().prev = prev;
            }
            None => self.tail = prev,
        }

        self.arena.remove(id).map(|node| node.value)
    }

    fn iter(&self) -> impl Iterator<Item = &T> {
        let arena = &self.arena;
        let mut current = self.head;

        std::iter::from_fn(move || {
            let id = current?;
            let node = arena.get(id)?;
            current = node.next;
            Some(&node.value)
        })
    }
}
```

---

## L'ecosistema: crate pronte all'uso

Non è necessario reimplementare l'arena da zero per ogni progetto. L'ecosistema Rust offre diverse crate mature che risolvono anche i limiti che abbiamo menzionato (stale ID, type-safety tra arene diverse):

- **[slotmap](https://crates.io/crates/slotmap)**: la più popolare. Offre `SlotMap`, `DenseSlotMap` e `HopSlotMap` con diversi trade-off tra velocità di iterazione e velocità di inserimento/rimozione. Usa internamente un generational index per invalidare automaticamente gli ID obsoleti.

- **[thunderdome](https://crates.io/crates/thunderdome)**: alternativa a `slotmap`, ottimizzata per semplicità e dimensione minima dell'ID (compatta generazione e indice in 64 bit).

- **[generational-arena](https://crates.io/crates/generational-arena)**: API simile a `slotmap`, focalizzata su semplicità e correttezza.

- **[id-arena](https://crates.io/crates/id-arena)**: arena con ID tipizzati ma senza generational index. Più semplice, adatta a scenari append-only dove la rimozione non serve.

---

## Quando usare (e quando evitare) il pattern

L'arena a indici è uno strumento potente, ma non è la risposta a tutto.

**Usala quando:**

- Hai strutture con riferimenti bidirezionali o ciclici (grafi, alberi, linked list).
- I dati hanno un ciclo di vita uniforme (tutti creati e distrutti nella stessa fase).
- Hai bisogno di serializzazione semplice (gli indici sono banali da serializzare).
- Le performance contano: l'allocazione è O(1), i dati sono contigui in memoria.

**Evitala quando:**

- La struttura è un semplice albero con ownership unidirezionale, `Box<T>` o `Vec<T>` bastano.
- Hai bisogno di garanzie a compile-time sui riferimenti, i lifetime di Rust, per quanto complicati, offrono garanzie che gli indici non possono dare.
- Gli elementi hanno cicli di vita molto diversi e la rimozione è rara, un semplice `Vec` con `Option<T>` potrebbe bastare.

---

## Conclusione

La doubly linked list in Rust è un problema che espone una tensione profonda del linguaggio: le regole di ownership che rendono Rust sicuro sono le stesse che rendono certi pattern classici difficili da esprimere. L'arena con indici non aggira queste regole, le rispetta, spostando tutta l'ownership in un unico punto e sostituendo i puntatori con indici numerici.

È un compromesso: perdi alcune garanzie statiche (un indice può diventare stale dopo una rimozione), ma guadagni semplicità, performance e sanità mentale. Per i casi dove questo rischio è rilevante, le crate dell'ecosistema offrono soluzioni pronte all'uso come il generational index e gli ID tipizzati, ma questo è materiale per un prossimo articolo.

La prossima volta che il borrow checker ti blocca su una struttura con riferimenti complessi, prima di cercare `Rc<RefCell<T>>`, prova a chiederti: *"E se usassi un'arena?"*.
