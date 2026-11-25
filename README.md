# Python notes for System Design, Routing, Algorithms &amp; Data Structures made by me

## Indice

1. [Struttura](#1-struttura)
2. [Fondamentali Python indispensabili](#2-fondamentali-python-indispensabili)
3. [OOP, dataclass e ordinamento degli oggetti](#3-oop-dataclass-e-ordinamento-degli-oggetti)
4. [Livello 1 — Consistent hashing](#4-livello-1--consistent-hashing)
5. [Livello 2 — Deduplicazione, indici e aggiornamenti](#5-livello-2--deduplicazione-indici-e-aggiornamenti)
6. [Livello 3 — Virtual nodes e health-aware routing](#6-livello-3--virtual-nodes-e-health-aware-routing)
7. [Livello 4 — Repliche, LRU e fallback RAM-aware](#7-livello-4--repliche-lru-e-fallback-ram-aware)
8. [Livello 5 — Isteresi, failure tracking, telemetria](#8-livello-5--isteresi-failure-tracking-telemetria)
9. [Big O, errori frequenti e attenzione ai test](#9-big-o-errori-frequenti-e-attenzione-ai-test)
10. [Checklist finale e promemoria da riscrivere a memoria](#10-checklist-finale-e-promemoria-da-riscrivere-a-memoria)

---

# 1. Struttura

| Livello | Competenze nuove | Risultato da ottenere |
|---|---|---|
| 1 | `dataclass`, FNV-1a, `insort`, `bisect_right` | Ring funzionante e ordinato |
| 2 | `dict`, `set`, idempotenza | Niente duplicati e aggiornamenti efficienti |
| 3 | `defaultdict`, virtual nodes, health | Routing su soli server sani |
| 4 | `OrderedDict`, LRU, fallback, RAM | Decisioni multiple e caching |
| 5 | `deque`, isteresi, fallimenti, dual epoch | Routing resiliente alle variazioni di telemetria |

---

# 2. Fondamentali Python indispensabili

## 2.1 Tipi, mutabilità e identità

```python
name = "server-A"     # str
capacity = 1024       # int
load = 0.75           # float
healthy = True        # bool
assigned = None       # NoneType
```

| Tipo | Esempio | Modificabile direttamente? |
|---|---|---|
| `int`, `float`, `bool`, `str` | `42`, `3.5`, `True`, `"A"` | No |
| `list` | `[1, 2]` | Sì |
| `tuple` | `(1, 2)` | No (ma può contenere oggetti mutabili) |
| `dict` | `{"A": 100}` | Sì |
| `set` | `{"A", "B"}` | Sì |
| `None` | `None` | No |

**Riferimenti agli oggetti:**

```python
a = [10, 20]
b = a
b.append(30)
assert a == [10, 20, 30]   # stesso oggetto
assert a is b

c = a.copy()                # nuova lista esterna
assert c == a and c is not a
```

Due operatori da non confondere: `==` controlla uguaglianza di valore, `is` identità dell'oggetto. `copy()` è **superficiale** (*shallow copy*): per strutture annidate usare `copy.deepcopy` se serve realmente una duplicazione ricorsiva.

## 2.2 Operatori da conoscere

```python
17 % 5      # 2: resto; utile per wrap-around
10 // 4     # 2: divisione intera per difetto
2 ** 3      # 8: potenza
1 << 64     # 2**64: shift a sinistra
5 ^ 3       # 6: XOR bit a bit (FNV-1a)
5 & 3       # 1: AND bit a bit (maschere)
```

**Truthiness**: `None`, `False`, `0`, `""`, `[]`, `{}`, `set()` sono falsy. `if not self.servers:` è il controllo idiomatico del ring vuoto.

## 2.3 Cicli e funzioni utili

```python
for i, server in enumerate(["A", "B", "C"]):
    print(i, server)            # indice + valore

for i in range(3):              # 0, 1, 2
    print(i)

numbers = [1, 2, 3, 4, 5, 6]
evens = [x for x in numbers if x % 2 == 0]  # [2, 4, 6]

assert any(x > 5 for x in numbers)
assert all(x > 0 for x in numbers)

def multiply(a: int, b: int = 2) -> int:
    return a * b
```

`break` interrompe il ciclo; `continue` salta all'iterazione seguente. `range(start, stop)` esclude `stop`.

## 2.4 Liste (`list`)

```python
servers = ["A", "B", "C"]
servers.append("D")         # aggiunge in coda
servers.insert(1, "X")      # inserisce in posizione 1
servers.remove("B")         # rimuove per valore (ValueError se assente)
first = servers.pop(0)      # rimuove per indice e restituisce il valore
last = servers[-1]         # accesso da destra
subset = servers[1:3]      # indice finale escluso
```

**`sort()` modifica la lista e restituisce `None`; `sorted(lista)` restituisce una nuova lista.**

```python
values = [3, 1, 2]
values.sort()                # values = [1, 2, 3]
ordered = sorted(values)     # nuova lista
```

## 2.5 Tuple (`tuple`)

```python
server = ("A", 1024, True)
server_id, ram, healthy = server   # unpacking
```

Sono sequenze immutabili, utili per conservare campi fissi, ad esempio tuple di cache `(primary, replicas, epoch, k, threshold)`. Una tupla è hashable solo se i suoi componenti lo sono.

## 2.6 Set (`set`)

```python
server_ids = set()
server_ids.add("A")
server_ids.add("A")         # nessun duplicato
assert "A" in server_ids
server_ids.discard("Z")     # nessun errore se assente
# server_ids.remove("Z")   # KeyError se assente
```

Uso primario nel ring: controllare che un server fisico non venga aggiunto due volte, o che non venga selezionato due volte come fallback.

## 2.7 Dizionari (`dict`)

```python
chat_to_server = {"c1": "A", "c2": "B"}
assert chat_to_server.get("c1") == "A"
assert chat_to_server.get("missing") is None
assert chat_to_server.get("missing", -1) == -1
chat_to_server["c1"] = "C"  # aggiorna chiave esistente
for chat_id, server_id in chat_to_server.items():
    print(chat_id, server_id)
```

In un `dict` le chiavi sono uniche e devono essere hashable. Accesso, inserimento e ricerca per chiave sono **O(1) medi**, non garantiti nel caso pessimo.

## 2.8 Pattern chiave: lista + dizionario sullo stesso oggetto

```python
chats = []
chat_by_id = {}
record = {"chat_id": "c1", "server": "A"}
chats.append(record)
chat_by_id["c1"] = record
chat_by_id["c1"]["server"] = "B"
assert chats[0]["server"] == "B"
assert chats[0] is chat_by_id["c1"]
```

**Mutare l'oggetto condiviso** mantiene coerenti lista e dizionario. **Sostituire la sola voce del dizionario** con un nuovo oggetto lascerebbe invece in lista il vecchio oggetto.

### Esercizio di allenamento — Registro chat

Implementa `register_chat(chats, chat_by_id, chat_id, server_id)`: nuova chat → aggiungi a entrambe le strutture e restituisci `True`; chat già presente → aggiorna il record originale e restituisci `False`. La lista non deve contenere duplicati.

**Soluzione proposta :**

```python
def register_chat(chats, chat_by_id, chat_id, server_id):
    chat = chat_by_id.get(chat_id)
    if chat is None:
        chat = {"chat_id": chat_id, "server_id": server_id}
        chats.append(chat)
        chat_by_id[chat_id] = chat
        return True
    chat["server_id"] = server_id
    return False

chats, index = [], {}
assert register_chat(chats, index, "c1", "A") is True
assert register_chat(chats, index, "c1", "B") is False
assert len(chats) == 1
assert chats[0] is index["c1"]
assert chats[0]["server_id"] == "B"
```

---

# 3. OOP, dataclass e ordinamento degli oggetti

## 3.1 Classi, oggetti, `self` e `__init__`

```python
class Server:
    def __init__(self, server_id: str, free_ram: int):
        self.server_id = server_id
        self.free_ram = free_ram
        self.healthy = True

    def update_ram(self, ram: int) -> None:
        self.free_ram = ram

    def can_accept(self, required_ram: int) -> bool:
        return self.healthy and self.free_ram >= required_ram

s = Server("A", 1024)
assert s.can_accept(512)
s.update_ram(256)
assert not s.can_accept(512)
```

- **Classe:** modello; **istanza:** oggetto concreto; **attributo:** dato; **metodo:** comportamento.
- `self` rappresenta l'istanza corrente; `s.update_ram(256)` passa automaticamente `s` come primo argomento.
- `__init__` inizializza la nuova istanza (formalmente la creazione è gestita da `__new__`).
- Prefisso `_`, come `_chat_by_id`: convenzione per attributi destinati all'uso interno, non privacy rigida.

**Errore comune — lista condivisa come attributo di classe:**

```python
class WrongRegistry:
    chats = []               # condivisa da tutte le istanze

class RightRegistry:
    def __init__(self):
        self.chats = []       # lista separata per istanza
```

## 3.2 Dataclass

```python
from dataclasses import dataclass, field

@dataclass
class BasicServer:
    server_id: str
    ram: int

assert BasicServer("A", 512) == BasicServer("A", 512)
```

`@dataclass` genera normalmente `__init__`, `__repr__`, `__eq__`. Con `order=True` genera anche `<`, `<=`, `>`, `>=`, confrontando i campi nell'ordine di dichiarazione, salvo quelli esclusi con `compare=False`.

```python
@dataclass(order=True, frozen=True)
class Server:
    h_str: int
    server_id: str = field(compare=False)

assert Server(100, "A") < Server(200, "B")
assert Server(100, "A") == Server(100, "B")  # confronta solo l'hash!
```

**Perché `field(compare=False)`?** Il ring ordina e confronta i server **solo per `h_str`**. Se due ID diversi hanno lo stesso hash, le istanze risulteranno uguali secondo il confronto generato: per identificare un server fisico continua a usare `server_id`.

**Perché `frozen=True`?** Impedisce la normale riassegnazione di attributi: è utile se il campo usato per ordinare non deve mutare mentre l'oggetto resta nella lista ordinata. Non rende profondamente immutabili gli eventuali oggetti mutabili contenuti.

```python
from typing import Optional

@dataclass(order=True)
class Chat:
    h_str: int
    chat_id: str = field(compare=False)
    server: Optional[Server] = field(default=None, compare=False)

chat = Chat(123, "c1")
assert chat.server is None
chat.server = Server(200, "A")  # consentito: Chat non è frozen
```

`Optional[Server]` equivale (da Python 3.10) a `Server | None`. I campi richiesti devono normalmente precedere i campi con default.

**Default mutabili: usare `default_factory`:**

```python
@dataclass
class Group:
    name: str
    members: list[str] = field(default_factory=list)
    metadata: dict[str, str] = field(default_factory=dict)
```

Ogni `Group` ottiene una nuova lista e un nuovo dizionario.

## 3.3 Composizione e riferimenti

```python
chat = Chat(100, "chat-A", Server(200, "S1"))
assert chat.server.server_id == "S1"
chat.server = Server(300, "S2")
assert chat.server.server_id == "S2"
```

La classe `ServerRing` gestisce una **lista di `Server` e una lista di `Chat`**, ciascuna chat a sua volta fa riferimento al server assegnato.

## 3.4 `bisect_right`, `bisect_left`, `insort`

```python
from bisect import insort, bisect_left, bisect_right

values = [10, 20, 20, 30]
assert bisect_left(values, 20) == 1
assert bisect_right(values, 20) == 3
insort(values, 25)
assert values == [10, 20, 20, 25, 30]
```

- `bisect_left`: indice **prima** degli elementi uguali.
- `bisect_right`: indice **dopo** degli elementi uguali → ricerca primo hash **strettamente maggiore**.
- `insort`: inserisce mantenendo la lista ordinata; ricerca O(log n), spostamenti della lista O(n).

Con una dataclass che confronta solo `h_str`:

```python
servers = []
insort(servers, Server(300, "C"))
insort(servers, Server(100, "A"))
insort(servers, Server(200, "B"))

idx = bisect_right(servers, Server(150, ""))  # 1
assert servers[idx].server_id == "B"

idx = bisect_right(servers, Server(400, ""))  # 3
idx %= len(servers)                            # 0: wrap-around
assert servers[idx].server_id == "A"
```

**Attenzione:** `idx %= len(servers)` è valido solo se `servers` non è vuota; prima fai `if not servers: ...`.

### Esercizio di allenamento — `NumberRing`

Crea un anello di nodi con posizione numerica, ordinati per `position`; rifiuta nomi duplicati, seleziona il primo nodo con posizione strettamente maggiore, con wrap-around.

**Soluzione proposta:**

```python
from dataclasses import dataclass, field
from bisect import insort, bisect_right

@dataclass(order=True, frozen=True)
class Node:
    position: int
    name: str = field(compare=False)

class NumberRing:
    def __init__(self):
        self.nodes: list[Node] = []
        self._names: set[str] = set()

    def add_node(self, position: int, name: str) -> bool:
        if name in self._names:
            return False
        insort(self.nodes, Node(position, name))
        self._names.add(name)
        return True

    def next_node(self, position: int) -> str | None:
        if not self.nodes:
            return None
        idx = bisect_right(self.nodes, Node(position, ""))
        return self.nodes[idx % len(self.nodes)].name

ring = NumberRing()
assert ring.next_node(10) is None
assert ring.add_node(30, "C")
assert ring.add_node(10, "A")
assert ring.add_node(20, "B")
assert ring.add_node(40, "A") is False
assert ring.next_node(10) == "B"
assert ring.next_node(30) == "A"
```

---

# 4. Consistent hashing

## 4.2 Concetto: anello e clockwise routing

Pensiamo ai valori hash come punti disposti lungo una circonferenza. Una chat con hash `h` viene assegnata al **primo server con hash maggiore di h**; se nessuno lo è, si torna al più piccolo (posizione 0).

Esempio semplificato:

```text
Server ordinati: [10 (A), 30 (B), 50 (C)]
Chat a 25       -> B (hash 30)
Chat a 30       -> C (hash 50): bisect_right esclude l'uguale
Chat a 70       -> A (hash 10): wrap-around
```

## 4.3 L'hash FNV-1a a 64 bit

```python
MASK64 = (1 << 64) - 1

def stable_hash(s: str) -> int:
    h = 0xCBF29CE484222325  # offset basis
    fnv_prime = 0x100000001B3
    for b in s.encode("utf-8"):
        h ^= b
        h = (h * fnv_prime) & MASK64
    return h
```

- `encode('utf-8')`: itera sui **byte**, non sui caratteri Unicode astratti.
- `^=`: XOR.
- `& MASK64`: restringe ai 64 bit meno significativi (Python `int` è arbitrariamente grande).
- L'hash built-in `hash("testo")` **non sostituisce** un hash deterministico tra esecuzioni.

## 4.4 Soluzione di riferimento didattica, completa e autonoma

```python
from __future__ import annotations
from dataclasses import dataclass, field
from bisect import insort, bisect_right
from typing import Optional

MASK64 = (1 << 64) - 1

def stable_hash(s: str) -> int:
    h = 0xCBF29CE484222325
    fnv_prime = 0x100000001B3
    for b in s.encode("utf-8"):
        h ^= b
        h = (h * fnv_prime) & MASK64
    return h

@dataclass(order=True, frozen=True)
class Server:
    h_str: int
    server_id: str = field(compare=False)

@dataclass(order=True)
class Chat:
    h_str: int
    chat_id: str = field(compare=False)
    server: Optional[Server] = field(default=None, compare=False)

class ServerRing:
    def __init__(self) -> None:
        self.servers: list[Server] = []
        self.chats: list[Chat] = []

    def add_server(self, server_id: str) -> bool:
        if any(s.server_id == server_id for s in self.servers):
            return False
        insort(self.servers, Server(stable_hash(server_id), server_id))
        return True

    def assign_server(self, chat_id: str) -> int:
        if not self.servers:
            return -1
        h = stable_hash(chat_id)
        idx = bisect_right(self.servers, Server(h, ""))
        idx %= len(self.servers)
        server = self.servers[idx]
        self.chats.append(Chat(h, chat_id, server))
        return server.h_str
```

## 4.5 Test essenziali

```python
ring = ServerRing()
assert ring.assign_server("empty") == -1
assert ring.add_server("B") is True
assert ring.add_server("A") is True
assert ring.add_server("A") is False
assert [s.h_str for s in ring.servers] == sorted(s.h_str for s in ring.servers)
result = ring.assign_server("chat1")
assert result in {s.h_str for s in ring.servers}
assert len(ring.chats) == 1
assert ring.chats[0].server.h_str == result
```

**Domande da saper rispondere:** perché `bisect_right` e non `bisect_left`? Che cosa significa `Server(h, "")`? Che cosa restituisce `assign_server`? Perché verificare prima la lista vuota? Perché il campo `server_id` ha `compare=False`?

---

# 5. Livello 2 — Deduplicazione, indici e aggiornamenti

## 5.1 Nuovi requisiti

- `remove_server(server_id) -> bool`: rimuovere il server fisico, mantenendo l'ordinamento.
- `get_chat_server_hash(chat_id) -> int`: hash attualmente memorizzato, o `-1` se sconosciuto/non assegnato.
- `assign_server(chat_id)`: nuova chat → inserire; esistente → **aggiornare lo stesso oggetto**, senza duplicati.
- Introdurre un set di ID e un dizionario di chat con lookup medio O(1).
- Conservare le API del livello 1. I record storici possono referenziare server rimossi finché non vengono riassegnati.

## 5.2 Nuove strutture e responsabilità

```python
self.servers: list[Server] = []            # ordinata per hash
self.chats: list[Chat] = []                # record condivisi
self._server_ids: set[str] = set()         # dedup degli ID
self._chat_by_id: dict[str, Chat] = {}     # lookup veloce
```

**Idempotenza:** a ring invariato, ripetere `assign_server("chat1")` non crea nuovi record; restituisce la stessa decisione. Dopo una modifica del ring può cambiare il server assegnato, ma il record deve rimanere unico.

### Schema upsert (create or update)

```python
chat = self._chat_by_id.get(chat_id)
if chat is None:
    chat = Chat(h, chat_id, assigned)
    self._chat_by_id[chat_id] = chat
    self.chats.append(chat)
else:
    chat.h_str = h
    chat.server = assigned
```

## 5.3 Soluzione didattica — Classe `ServerRing` aggiornata


```python
class ServerRing:
    def __init__(self) -> None:
        self.servers: list[Server] = []
        self.chats: list[Chat] = []
        self._server_ids: set[str] = set()
        self._chat_by_id: dict[str, Chat] = {}

    def add_server(self, server_id: str) -> bool:
        if server_id in self._server_ids:
            return False
        insort(self.servers, Server(stable_hash(server_id), server_id))
        self._server_ids.add(server_id)
        return True

    def remove_server(self, server_id: str) -> bool:
        if server_id not in self._server_ids:
            return False
        self._server_ids.remove(server_id)
        for i, server in enumerate(self.servers):
            if server.server_id == server_id:
                del self.servers[i]
                break
        return True

    def _choose_server(self, h: int) -> Optional[Server]:
        if not self.servers:
            return None
        idx = bisect_right(self.servers, Server(h, ""))
        return self.servers[idx % len(self.servers)]

    def assign_server(self, chat_id: str) -> int:
        h = stable_hash(chat_id)
        assigned = self._choose_server(h)
        chat = self._chat_by_id.get(chat_id)
        if chat is None:
            chat = Chat(h, chat_id, assigned)
            self._chat_by_id[chat_id] = chat
            self.chats.append(chat)
        else:
            chat.h_str = h
            chat.server = assigned
        return assigned.h_str if assigned is not None else -1

    def get_chat_server_hash(self, chat_id: str) -> int:
        chat = self._chat_by_id.get(chat_id)
        if chat is None or chat.server is None:
            return -1
        return chat.server.h_str
```

**Nota rispetto alla reference:** la gestione del ring vuoto per una chat già esistente è formulata in modo leggermente differente nel codice originale. Qui, durante `assign_server` senza server disponibili, l'assegnazione viene esplicitamente impostata a `None`; è una scelta didattica coerente con una riassegnazione corrente e con il ritorno `-1`. In prova, segui sempre gli assert dei test forniti.

## 5.4 Esercizio di allenamento — `Registry`

Implementa un registro con `add_server`, `remove_server`, `assign(chat_id, server_id)` e `lookup(chat_id)`. `assign` deve rifiutare server non attivi, aggiornare senza duplicare le chat e restituire `True` su aggiunta/aggiornamento. La rimozione lascia i record storici.

**Soluzione proposta:**

```python
from dataclasses import dataclass

@dataclass
class ChatRecord:
    chat_id: str
    server_id: str | None = None

class Registry:
    def __init__(self):
        self.servers: set[str] = set()
        self.chats: list[ChatRecord] = []
        self._chat_by_id: dict[str, ChatRecord] = {}

    def add_server(self, server_id: str) -> bool:
        if server_id in self.servers:
            return False
        self.servers.add(server_id)
        return True

    def remove_server(self, server_id: str) -> bool:
        if server_id not in self.servers:
            return False
        self.servers.remove(server_id)
        return True

    def assign(self, chat_id: str, server_id: str) -> bool:
        if server_id not in self.servers:
            return False
        chat = self._chat_by_id.get(chat_id)
        if chat is None:
            chat = ChatRecord(chat_id, server_id)
            self._chat_by_id[chat_id] = chat
            self.chats.append(chat)
        else:
            chat.server_id = server_id
        return True

    def lookup(self, chat_id: str) -> str | None:
        chat = self._chat_by_id.get(chat_id)
        return None if chat is None else chat.server_id

r = Registry()
assert r.add_server("A") and r.add_server("B")
assert r.add_server("A") is False
assert r.assign("c1", "A") and r.assign("c1", "B")
assert len(r.chats) == 1
assert r.lookup("c1") == "B"
assert r.chats[0] is r._chat_by_id["c1"]
assert r.assign("c2", "Z") is False
assert r.remove_server("B") is True
assert r.lookup("c1") == "B"  # storico
```

**Cosa memorizzare:** il set accelera la dedup, il dizionario accelera il recupero, la lista conserva i record, la mutazione dell'oggetto mantiene coerenti i riferimenti. La rimozione dalla lista resta O(n).

---
# 6. Livello 3 — Virtual nodes e health-aware routing

## 6.1 Perché cambiare il ring?

Con un solo nodo per server fisico, il carico può essere sbilanciato: gli intervalli tra hash successivi possono avere dimensioni molto diverse. Un server fisico viene quindi rappresentato da più **virtual nodes (vnode)** distribuiti sull'anello.

```text
Server fisico A, replicas = 3:
  vnode A#0 = stable_hash("A#0")
  vnode A#1 = stable_hash("A#1")
  vnode A#2 = stable_hash("A#2")

Tutti i vnode conservano server_id = "A" ma h_str differenti.
```

Il server fisico può essere **healthy** oppure **unhealthy**. I vnode unhealthy restano nell'anello, ma vengono saltati durante la ricerca della destinazione.

## 6.2 Requisiti ufficiali nuovi

- `add_server(server_id, replicas=64)`: inserisce `replicas` vnode mediante `insort`.
- Hash dei vnode: `stable_hash(f"{server_id}#{i}")` per `i in range(replicas)`.
- Mappatura `server_id -> list[vnode_hashes]` per conoscere le repliche del server.
- `remove_server(server_id)`: elimina **tutti** i vnode del server fisico.
- `mark_server_unhealthy(server_id)`, `mark_server_healthy(server_id)`: mantengono il server nel ring, cambiando la disponibilità.
- `assign_server(chat_id)`: `bisect_right` su hash chat e scansione circolare fino al primo nodo healthy.
- Deve utilizzare almeno una struttura di `collections` (es. `defaultdict`).

## 6.3 `collections.defaultdict`

```python
from collections import defaultdict

vnodes = defaultdict(list)
vnodes["A"].append(100)
vnodes["A"].append(200)
assert vnodes["A"] == [100, 200]

counter = defaultdict(int)
counter["A"] += 1
assert counter["A"] == 1
```

`defaultdict(list)` crea una lista vuota **quando si accede con `[]` a una nuova chiave**. `defaultdict(int)` crea il valore iniziale `0`. Per il solo controllo di esistenza, usare `key in mapping` oppure `.get()`: evitare accessi che creano involontariamente chiavi.

## 6.4 Health management con set

```python
self._unhealthy: set[str] = set()

# Se l'ID fisico è noto:
self._unhealthy.add(server_id)      # diventa unhealthy
self._unhealthy.discard(server_id)  # torna healthy, anche se non era presente

healthy = (server_id in self._server_ids
           and server_id not in self._unhealthy)
```

Lo stato di health si riferisce **all'ID fisico**, non al singolo vnode: se A è unhealthy, vanno ignorate tutte le sue repliche.

## 6.5 Algoritmo centrale: scansione circolare

```python
def _choose_server_health_aware(self, h: int) -> Server | None:
    if not self.servers:
        return None

    start = bisect_right(self.servers, Server(h, ""))
    n = len(self.servers)

    for step in range(n):
        idx = (start + step) % n
        vnode = self.servers[idx]
        if vnode.server_id not in self._unhealthy:
            return vnode

    return None
```

La formula **`(start + step) % n`** effettua il wrap-around. Il ciclo non può andare oltre `n` passi, quindi termina anche se tutti i server sono unhealthy. Per una ricerca rapida in media, `bisect_right` vale O(log N); la scansione aggiunge fino a O(N) nel caso peggiore (N = numero di vnode).

## 6.6 Implementazione di consolidamento — `ServerRing` livello 3

> Riutilizzare gli import, `stable_hash`, `Server` e `Chat`.

```python
from collections import defaultdict

class ServerRing:
    def __init__(self) -> None:
        self.servers: list[Server] = []
        self.chats: list[Chat] = []
        self._server_ids: set[str] = set()
        self._chat_by_id: dict[str, Chat] = {}
        self._vnodes_by_server = defaultdict(list)
        self._unhealthy: set[str] = set()

    def add_server(self, server_id: str, replicas: int = 64) -> bool:
        if server_id in self._server_ids:
            return False
        if replicas <= 0:
            replicas = 1
        self._server_ids.add(server_id)
        hashes = []
        for i in range(replicas):
            h = stable_hash(f"{server_id}#{i}")
            insort(self.servers, Server(h, server_id))
            hashes.append(h)
        self._vnodes_by_server[server_id] = hashes
        return True

    def remove_server(self, server_id: str) -> bool:
        if server_id not in self._server_ids:
            return False
        self._server_ids.remove(server_id)
        self._unhealthy.discard(server_id)
        self._vnodes_by_server.pop(server_id, None)
        self.servers = [s for s in self.servers
                        if s.server_id != server_id]
        return True

    def mark_server_unhealthy(self, server_id: str) -> bool:
        if server_id not in self._server_ids:
            return False
        self._unhealthy.add(server_id)
        return True

    def mark_server_healthy(self, server_id: str) -> bool:
        if server_id not in self._server_ids:
            return False
        self._unhealthy.discard(server_id)
        return True

    def _choose_server(self, h: int) -> Optional[Server]:
        if not self.servers:
            return None
        start = bisect_right(self.servers, Server(h, ""))
        n = len(self.servers)
        for step in range(n):
            s = self.servers[(start + step) % n]
            if s.server_id not in self._unhealthy:
                return s
        return None

    def assign_server(self, chat_id: str) -> int:
        h = stable_hash(chat_id)
        assigned = self._choose_server(h)
        chat = self._chat_by_id.get(chat_id)
        if chat is None:
            chat = Chat(h, chat_id, assigned)
            self._chat_by_id[chat_id] = chat
            self.chats.append(chat)
        else:
            chat.h_str = h
            chat.server = assigned
        return assigned.h_str if assigned is not None else -1

    def get_chat_server_hash(self, chat_id: str) -> int:
        chat = self._chat_by_id.get(chat_id)
        return -1 if chat is None or chat.server is None else chat.server.h_str
```

**Nota prestazionale:** `_vnodes_by_server` consente di recuperare rapidamente la lista degli hash di un server, ma la versione didattica di `remove_server` usa una comprensione che filtra la lista globale O(N). Quindi la presenza dell'indice non rende da sola tutta la rimozione O(replicas). È lo stesso tipo di compromesso semplice adottato dalla reference.

## 6.7 Esercizio di allenamento — `HealthyRing`

Progetta un ring di vnode con posizioni già note (non calcolate con FNV), e metodi `add`, `remove`, `mark_unhealthy`, `mark_healthy`, `pick`. `pick` deve restituire l'**ID fisico** del primo vnode healthy successivo alla posizione richiesta; nessun server sano → `None`.

**Soluzione proposta:**

```python
from dataclasses import dataclass, field
from collections import defaultdict
from bisect import bisect_right, insort

@dataclass(order=True, frozen=True)
class VNode:
    position: int
    server_id: str = field(compare=False)

class HealthyRing:
    def __init__(self):
        self.nodes: list[VNode] = []
        self._vnodes_by_server = defaultdict(list)
        self._unhealthy: set[str] = set()

    def add(self, server_id: str, positions: list[int]) -> bool:
        if server_id in self._vnodes_by_server:
            return False
        self._vnodes_by_server[server_id] = list(positions)
        for position in positions:
            insort(self.nodes, VNode(position, server_id))
        return True

    def remove(self, server_id: str) -> bool:
        if server_id not in self._vnodes_by_server:
            return False
        del self._vnodes_by_server[server_id]
        self._unhealthy.discard(server_id)
        self.nodes = [node for node in self.nodes
                      if node.server_id != server_id]
        return True

    def mark_unhealthy(self, server_id: str) -> bool:
        if server_id not in self._vnodes_by_server:
            return False
        self._unhealthy.add(server_id)
        return True

    def mark_healthy(self, server_id: str) -> bool:
        if server_id not in self._vnodes_by_server:
            return False
        self._unhealthy.discard(server_id)
        return True

    def pick(self, position: int) -> str | None:
        if not self.nodes:
            return None
        start = bisect_right(self.nodes, VNode(position, ""))
        for step in range(len(self.nodes)):
            node = self.nodes[(start + step) % len(self.nodes)]
            if node.server_id not in self._unhealthy:
                return node.server_id
        return None

ring = HealthyRing()
assert ring.add("A", [10, 50])
assert ring.add("B", [20, 60])
assert ring.add("C", [30, 70])
assert ring.pick(15) == "B"
assert ring.mark_unhealthy("B")
assert ring.pick(15) == "C"
assert ring.mark_unhealthy("C")
assert ring.pick(15) == "A"
assert ring.mark_unhealthy("A")
assert ring.pick(15) is None
assert ring.mark_healthy("B")
assert ring.pick(15) == "B"
assert ring.remove("B")
assert len(ring.nodes) == 4
```

---

# 7. Livello 4 — Repliche, LRU e fallback RAM-aware

## 7.1 Requisiti nuovi

1. `update_server_ram(server_id, free_ram_mb) -> bool`: aggiorna la telemetria RAM del server fisico; nessuna modifica della topologia, **non** incrementa `_epoch`.
2. `assign_servers(chat_id, k=3, min_free_ram_mb=0) -> list[int]`: restituisce fino a `k` hash di vnode appartenenti a **server fisici distinti**, tutti healthy.
3. Il **primary** è sempre il primo server healthy incontrato lungo l'anello: selezione deterministica, **indipendente dalla RAM**.
4. Nei **fallback**, preferire candidati con RAM nota e sufficiente; se non bastano, completare con gli altri healthy. Con soglia `<= 0`, nessun filtro RAM.
5. `assign_server(chat_id, primary_failed=False, min_free_ram_mb=0)`: ritorna il primary normale, o il miglior fallback quando `primary_failed=True`; `-1` se nessuna opzione utilizzabile.
6. Cache LRU con `OrderedDict`, **capacità 8192**. Invalidare in modo globale quando cambia topologia/health; aggiornamenti RAM possono lasciare dati cached temporaneamente stale.

## 7.2 Replica routing: distinzione tra vnode e server fisici

```text
Ordine di incontro dei vnode dopo la chat:
  A#1 -> B#0 -> A#2 -> C#4 -> D#0

Server fisici distinti, k=3:
  [A, B, C]   (A#2 ignorato: A era già selezionato)
```

**Attenzione alla specifica:** il risultato del vero `assign_servers` è una **lista di hash di vnode (`list[int]`)**, non una lista di ID fisici (`list[str]`). Negli esercizi semplificati usiamo ID per visualizzare più facilmente la logica.

## 7.3 RAM e politica dei fallback

Supponiamo che, in ordine orario, i candidati unici siano:

| Ordine | Server | RAM libera |
|---|---|---:|
| 1 | A | 100 MB |
| 2 | B | 400 MB |
| 3 | C | 3000 MB |
| 4 | D | 2000 MB |

Per `k=3`, soglia `1000 MB`: **primary A**, fallback preferiti **C, D**. B non è escluso per sempre: con `k=4` può essere aggiunto come fallback di riserva.

**Pattern a due passaggi:**

1. Scansione oraria dei candidati healthy, senza ripetere server fisici, cercando fallback che soddisfano la soglia.
2. Se la lista è ancora corta, nuova scansione per aggiungere gli altri healthy.

Questo significa che i fallback preferiti possono passare avanti a candidati incontrati prima ma con RAM insufficiente: l'ordine di selezione segue due scansioni orarie, non sempre un singolo ordine globale immutato.

## 7.4 `OrderedDict` e cache LRU

```python
from collections import OrderedDict
cache = OrderedDict()
cache["c1"] = "A"
cache["c2"] = "B"
cache["c3"] = "C"

cache.move_to_end("c1")        # c1 diventa il più recente
oldest_key, value = cache.popitem(last=False)  # rimuove c2
```

LRU = *Least Recently Used*. Alla lettura di una voce valida, richiamare `move_to_end`. All'inserimento oltre capacità, richiamare `popitem(last=False)`.

La cache della reference salva, per chat:

```python
(primary_hash, replica_hashes_tuple, ring_epoch, k, min_free_ram_mb)
```

Non basta controllare il solo `chat_id`: le richieste con `k` e soglie RAM differenti possono richiedere risultati differenti.

## 7.5 Topologia e `_epoch`

```python
self._epoch: int = 0
self._lru = OrderedDict()

def _bump_epoch(self) -> None:
    self._epoch += 1
    self._lru.clear()
```

- `add_server`, `remove_server`, cambi health → `_bump_epoch()`; la cache non è più valida.
- `update_server_ram` → **non** incrementare `_epoch` al livello 4. Possibile cache stale rispetto alla RAM: compromesso prestazionale esplicito della specifica.
- Una voce stale rispetto a `_epoch`, oppure con `k`/soglia incompatibile, è un cache miss.

## 7.6 `primary_failed`

Se `assign_servers` restituisce `[primary_hash, fallback1_hash, fallback2_hash]`:

```python
if not replicas:
    return -1
if not primary_failed:
    return replicas[0]
return replicas[1] if len(replicas) > 1 else -1
```

La selezione di `replicas[1]` è appropriata **se** `assign_servers` ha già ordinato i fallback secondo la politica di RAM. Il record `Chat.server` continua a memorizzare il **primary normale**, anche se una chiamata chiede un fallback temporaneo.

## 7.7 Esercizio A — Preferenza dei fallback

Implementa `choose_fallbacks(candidates, k, threshold)` dove `candidates` è una lista ordinata di `(server_id, free_ram_mb)` e tutti i candidati sono healthy. Mantieni il primo come primary, evita duplicati e preferisci i fallback con RAM sufficiente.

**Soluzione proposta:**

```python
def choose_fallbacks(
    candidates: list[tuple[str, int | None]],
    k: int,
    threshold: int
) -> list[str]:
    if not candidates or k <= 0:
        return []
    chosen = [candidates[0][0]]  # primary indipendente dalla RAM
    used = {chosen[0]}

    def fill(require_ram: bool) -> None:
        for server_id, ram in candidates[1:]:
            if len(chosen) >= k:
                return
            if server_id in used:
                continue
            if require_ram and (ram is None or ram < threshold):
                continue
            chosen.append(server_id)
            used.add(server_id)

    if threshold > 0:
        fill(require_ram=True)
    fill(require_ram=False)
    return chosen

candidates = [("A", 100), ("B", 400), ("C", 3000),
              ("B", 400), ("D", 2000)]
assert choose_fallbacks(candidates, 3, 1000) == ["A", "C", "D"]
assert choose_fallbacks(candidates, 4, 1000) == ["A", "C", "D", "B"]
```

## 7.8 Esercizio B — Cache `RoutingCache`

Implementa cache con capacità configurabile, chiave `chat_id`, contenuto `(epoch, k, threshold, replicas)`; deve riutilizzare `k` inferiori e stessa soglia, marcare gli accessi LRU e svuotarsi su cambio della topologia.

**Soluzione proposta:**

```python
from collections import OrderedDict

class RoutingCache:
    def __init__(self, capacity: int = 2):
        self._cache = OrderedDict()
        self._capacity = capacity
        self._epoch = 0

    def get(self, chat_id: str, k: int, threshold: int) -> list[str] | None:
        entry = self._cache.get(chat_id)
        if entry is None:
            return None
        epoch, stored_k, stored_threshold, reps = entry
        if (epoch != self._epoch or stored_k < k
                or threshold != stored_threshold):
            self._cache.pop(chat_id, None)
            return None
        self._cache.move_to_end(chat_id)
        return list(reps[:k])

    def put(self, chat_id: str, k: int, threshold: int,
            replicas: list[str]) -> None:
        self._cache[chat_id] = (self._epoch, k, threshold, tuple(replicas))
        self._cache.move_to_end(chat_id)
        if len(self._cache) > self._capacity:
            self._cache.popitem(last=False)

    def topology_changed(self) -> None:
        self._epoch += 1
        self._cache.clear()

cache = RoutingCache(capacity=2)
cache.put("c1", 3, 1000, ["A", "C", "D"])
cache.put("c2", 1, 0, ["A"])
assert cache.get("c1", 2, 1000) == ["A", "C"]
cache.put("c3", 1, 0, ["C"])
assert cache.get("c2", 1, 0) is None  # LRU eliminata
cache.topology_changed()
assert cache.get("c1", 2, 1000) is None
```

## 7.9 Checklist di implementazione nel vero `ServerRing`

1. Nuova RAM map: `_free_ram_mb: dict[str, int]`.
2. Nuova LRU: `OrderedDict`, `_CACHE_CAP = 8192` e `_epoch`.
3. `add_server`, `remove_server`, `mark_server_healthy/unhealthy` aggiornano epoch e cache.
4. `assign_servers`: posizione chat → primo vnode sano (`primary`) → scansione fallback ad alta RAM → completamento fallback healthy.
5. Mantenere **ID fisici unici** mentre si restituiscono **hash dei vnode**.
6. Aggiornare `Chat.server` al primary, non al fallback.
7. Cache hit solo se entry valida per epoch, `k` e threshold; hit deve essere portato alla fine dell'LRU.
8. `assign_server` delega a `assign_servers` e seleziona primary/fallback secondo `primary_failed`.
9. Testare `k <= 0`, nessun server, tutti unhealthy, un solo server, RAM sconosciuta, `k` superiore al numero di server fisici.

> **Semplificazione esplicita:** gli esercizi A e B coprono rispettivamente algoritmo di selezione e cache.

---
# 8. Livello 5 — Isteresi, failure tracking, telemetria

## 8.1 Requisiti nuovi

Il livello 5 introduce una politica di pressione sulle risorse (*RAM pressure*) e una cache che distingue due tipi di modifiche:

- `set_ram_thresholds(min_free_ram_mb, hysteresis_mb=256) -> None`.
- `record_server_failure(server_id) -> bool`: `False` per server fisico inesistente.
- `assign_servers(chat_id, k=3, min_free_ram_mb: Optional[int] = None) -> list[int]`: se la soglia è `None`, usa quella globale.
- **Primary:** primo nodo healthy in senso orario, indipendentemente dalla pressione.
- **Fallback:** prima candidati healthy a pressione accettabile; se insufficienti, aggiungi gli altri healthy.
- `_epoch` per topologia/health, con invalidazione globale della cache.
- `_tele_epoch` per RAM/fallimenti, con invalidazione **lazy** (solo quando si accede alla voce).
- Richiesti `OrderedDict` e almeno uno tra `deque`, `defaultdict` per registrare i fallimenti.

## 8.2 Il problema del flapping

Senza isteresi, con soglia 1000 MB:

```text
999 MB  -> non idoneo
1001 MB -> idoneo
998 MB  -> non idoneo
1002 MB -> idoneo
```

Il server oscillerebbe continuamente tra i due stati.

### Isteresi: soglia di attivazione diversa dalla soglia di disattivazione

Con `min_free_ram_mb=1000` e `hysteresis_mb=200`:

- Diventa RAM-eligible solo quando RAM `>= 1200` MB.
- Se è già RAM-eligible, rimane tale finché RAM `>= 1000` MB.
- Diventa non eligible se RAM `< 1000` MB.

Quindi una sequenza come `1100 -> 1200 -> 1100 -> 999` produce gli stati `False -> True -> True -> False` (partendo da non idoneo).

```python
def update_server_ram(self, server_id: str, ram: int) -> bool:
    if server_id not in self._server_ids:
        return False
    ram = max(0, ram)
    self._free_ram_mb[server_id] = ram
    if server_id in self._ram_eligible:
        if ram < self._min_free_ram_mb:
            self._ram_eligible.remove(server_id)
    else:
        activation = self._min_free_ram_mb + self._hysteresis_mb
        if ram >= activation:
            self._ram_eligible.add(server_id)
    self._bump_tele_epoch()
    return True
```

È una **macchina a stati a due stati** (eligible/non eligible), non un semplice controllo `ram >= threshold`.

## 8.3 `deque` e fallimenti

```python
from collections import defaultdict, deque

failures = defaultdict(lambda: deque(maxlen=32))
failures["A"].append(1)
failures["A"].append(1)
assert len(failures["A"]) == 2

items = deque(maxlen=3)
for i in [1, 2, 3, 4]:
    items.append(i)
assert list(items) == [2, 3, 4]
```

La `deque` è una coda a due estremità: `append`, `appendleft`, `pop`, `popleft` sono O(1) alle estremità. `maxlen` limita la memoria ed elimina automaticamente gli elementi più vecchi quando necessario.

**Limite della reference:** usa i marcatori fino a 32, ma non conserva timestamp né applica un vero timeout. È una memoria limitata di fallimenti, non una finestra temporale rigorosa.

## 8.4 Pressure score / preferenza del fallback

Una versione didattica della politica è:

```python
def _pressure_ok(self, server_id: str) -> bool:
    if self._min_free_ram_mb > 0:
        if server_id not in self._ram_eligible:
            return False
    failures = len(self._failures.get(server_id, ()))
    return failures <= 2
```

**Health** e **pressure** non sono la stessa cosa:

- Un server **unhealthy** non deve essere selezionato, né come primary né come fallback.
- Un server healthy ma **sotto pressione** può essere scelto come fallback di riserva, quando non ci sono abbastanza alternative migliori.
- Il primary resta quello determinato dal ring e dalla health: non cambia a ogni variazione di RAM.

## 8.5 Dual epoch e lazy invalidation

```python
self._epoch = 0             # topologia + health
self._tele_epoch = 0        # RAM + fallimenti
self._lru = OrderedDict()

def _bump_epoch(self) -> None:
    self._epoch += 1
    self._lru.clear()

def _bump_tele_epoch(self) -> None:
    self._tele_epoch += 1
    # non cancellare tutto subito
```

Una cache entry deve essere riutilizzabile soltanto se entrambe le versioni coincidono (oltre a `k` e soglia compatibili).

```python
if cached_ring != self._epoch or cached_tele != self._tele_epoch:
    # cache miss, elimina questa voce e ricalcola
    pass
```

**Invalidazione eager (globale):** svuoti la cache al cambiamento della topologia.  
**Invalidazione lazy:** un cambio di telemetria incrementa solo la versione; le entry vecchie occupano memoria fino al prossimo accesso o all'evizione LRU, ma non vengono riutilizzate dopo che sono risultate obsolete.

## 8.6 Esercizio A — `PressureMonitor`

Implementa il monitor di RAM e fallimenti. Parti da stato non eligible, usa isteresi, conteggia gli ultimi 32 marcatori di failure e incrementa `tele_epoch` a ogni update. Un server è `preferred` se soddisfa la politica RAM e ha al massimo due fallimenti.

**Soluzione proposta:**

```python
from collections import defaultdict, deque

class PressureMonitor:
    def __init__(self, threshold: int = 1000,
                 hysteresis: int = 200):
        self.threshold = threshold
        self.hysteresis = hysteresis
        self.free_ram: dict[str, int] = {}
        self.eligible: set[str] = set()
        self.failures = defaultdict(lambda: deque(maxlen=32))
        self.tele_epoch = 0

    def update_ram(self, server_id: str, mb: int) -> None:
        mb = max(0, mb)
        self.free_ram[server_id] = mb
        if server_id in self.eligible:
            if mb < self.threshold:
                self.eligible.remove(server_id)
        else:
            if mb >= self.threshold + self.hysteresis:
                self.eligible.add(server_id)
        self.tele_epoch += 1

    def record_failure(self, server_id: str) -> None:
        self.failures[server_id].append(1)
        self.tele_epoch += 1

    def preferred(self, server_id: str) -> bool:
        ram_ok = self.threshold <= 0 or server_id in self.eligible
        failures_ok = len(self.failures[server_id]) <= 2
        return ram_ok and failures_ok

monitor = PressureMonitor()
monitor.update_ram("A", 1100)
assert "A" not in monitor.eligible
monitor.update_ram("A", 1200)
assert "A" in monitor.eligible
monitor.update_ram("A", 1050)
assert "A" in monitor.eligible
monitor.update_ram("A", 999)
assert "A" not in monitor.eligible
monitor.update_ram("A", 1300)
assert monitor.preferred("A")
monitor.record_failure("A")
monitor.record_failure("A")
assert monitor.preferred("A")
monitor.record_failure("A")
assert monitor.preferred("A") is False
```

Questa classe didattica accetta qualunque ID; in `ServerRing.record_server_failure` va verificata l'esistenza del server e restituito un booleano.

## 8.7 Esercizio B — `VersionedLRU`

Crea una cache con due epoch: gli aggiornamenti RAM invalidano **solo al prossimo accesso**, quelli di topologia svuotano subito tutto. Rispetta la capacità massima LRU.

**Soluzione proposta:**

```python
from collections import OrderedDict

class VersionedLRU:
    def __init__(self, capacity: int = 2):
        self.capacity = capacity
        self.ring_epoch = 0
        self.tele_epoch = 0
        self._cache = OrderedDict()

    def put(self, chat_id: str, server: str) -> None:
        self._cache[chat_id] = (
            server, self.ring_epoch, self.tele_epoch
        )
        self._cache.move_to_end(chat_id)
        if len(self._cache) > self.capacity:
            self._cache.popitem(last=False)

    def get(self, chat_id: str) -> str | None:
        entry = self._cache.get(chat_id)
        if entry is None:
            return None
        server, ring, tele = entry
        if ring != self.ring_epoch or tele != self.tele_epoch:
            del self._cache[chat_id]
            return None
        self._cache.move_to_end(chat_id)
        return server

    def ram_changed(self) -> None:
        self.tele_epoch += 1

    def topology_changed(self) -> None:
        self.ring_epoch += 1
        self._cache.clear()

cache = VersionedLRU()
cache.put("c1", "A")
cache.put("c2", "B")
assert cache.get("c1") == "A"
cache.ram_changed()
assert len(cache._cache) == 2  # non svuotata immediatamente
assert cache.get("c1") is None
assert len(cache._cache) == 1  # c1 eliminata su accesso
cache.topology_changed()
assert len(cache._cache) == 0
```

## 8.8 Implementazione

1. Conservare tutti i metodi pubblici dei livelli precedenti, comprese le opzioni `primary_failed` e `min_free_ram_mb`.
2. Aggiungere `_min_free_ram_mb`, `_hysteresis_mb`, `_ram_eligible`, `_failures`, `_tele_epoch`.
3. Quando cambia la politica globale RAM, **ricalcolare coerentemente** lo stato di eleggibilità dei server con telemetria nota e invalidare le decisioni che dipendono da essa; controllare i test per la semantica esatta.
4. `record_server_failure`: server sconosciuto → `False`; noto → registra e incrementa `_tele_epoch`.
5. Fallimenti e RAM incidono **solo sulle preferenze di fallback**, non sulla scelta del primary.
6. Cache entry registra sia `_epoch` sia `_tele_epoch`; hit valido solo se entrambi coincidono e i parametri sono compatibili.
7. La telemetria cambia **solo** `_tele_epoch`, senza `.clear()` globale.
8. Topologia e health incrementano `_epoch`, con `.clear()` globale.
9. Testare transizioni `soglia-1`, `soglia`, `soglia+isteresi-1`, `soglia+isteresi` e più fallimenti.

> **Semplificazione esplicita:** gli esercizi A e B implementano due sottosistemi isolati.
---

# 9. Big O, errori frequenti e attenzione ai test

## 9.1 Complessità da conoscere

| Operazione | Complessità tipica |
|---|---|
| Accesso per indice a una lista | O(1) |
| `list.append()` | O(1) ammortizzato |
| Ricerca con `x in list` | O(n) |
| `insert`/`del` in mezzo a una lista | O(n) |
| `x in set` / `dict.get(key)` | O(1) medio |
| `sorted(lista)` / `lista.sort()` | O(n log n) |
| `bisect_right(lista_ordinata, x)` | O(log n) |
| `insort(lista_ordinata, x)` | O(n) complessivo (spostamento elementi) |
| `deque.append`, `deque.popleft` | O(1) |
| Scansione circolare di tutti i vnode | O(N) nel caso peggiore |
| LRU get/put con OrderedDict | O(1) medio per le operazioni sul dizionario |

**Attenzione:** non confondere complessità dell'accesso a `_vnodes_by_server` (O(1) medio per la mappa) e complessità di **eliminare** i nodi fisici dalla lista globale (spesso O(N)). N è il numero totale di virtual nodes, non il numero di server fisici.

## 10.2 Snippet scheletro

```python
from __future__ import annotations
from dataclasses import dataclass, field
from bisect import bisect_right, insort
from collections import defaultdict, OrderedDict, deque
from typing import Optional

MASK64 = (1 << 64) - 1

def stable_hash(s: str) -> int:
    h = 0xCBF29CE484222325
    for byte in s.encode("utf-8"):
        h ^= byte
        h = (h * 0x100000001B3) & MASK64
    return h

@dataclass(order=True, frozen=True)
class Server:
    h_str: int
    server_id: str = field(compare=False)

@dataclass(order=True)
class Chat:
    h_str: int
    chat_id: str = field(compare=False)
    server: Optional[Server] = field(default=None, compare=False)

# Ring ordinato:
servers: list[Server] = []
insort(servers, Server(stable_hash("A"), "A"))

# Primo hash strettamente maggiore, con wrap-around:
if servers:
    idx = bisect_right(servers, Server(stable_hash("chat1"), ""))
    assigned = servers[idx % len(servers)]

# Indici e strutture ausiliarie:
server_ids: set[str] = set()
chat_by_id: dict[str, Chat] = {}
vnodes_by_server = defaultdict(list)
unhealthy: set[str] = set()
failures = defaultdict(lambda: deque(maxlen=32))
lru = OrderedDict()

# Aggiornamento utilizzo LRU:
lru["c1"] = ("A", 0)
lru.move_to_end("c1")
lru.popitem(last=False)
```

