---
title: Accodare una Serie di Aggiornamenti di Stato
---

<Intro>

Impostando una variabile di stato si accoda un ulteriore render. Tuttavia, a volte si potrebbe desiderare eseguire più operazioni sul valore prima di accodare il prossimo render. Per farlo, è utile comprendere come React raggruppa gli aggiornamenti di stato.

</Intro>

<YouWillLearn>

* Cos'è il "raggruppamento" (batching) e come React lo utilizza per elaborare più aggiornamenti di stato contemporaneamente
* Come applicare diversi aggiornamenti alla stessa variabile di stato in successione

</YouWillLearn>

## React raggruppa gli aggiornamenti di stato {/*react-batches-state-updates*/}

Potresti aspettarti che cliccando il bottone '+3' il contatore viene aumentato tre volte perché chiama setNumber(number + 1) tre volte:

<Sandpack>

```js
import { useState } from 'react';

export default function Counter() {
  const [number, setNumber] = useState(0);

  return (
    <>
      <h1>{number}</h1>
      <button onClick={() => {
        setNumber(number + 1);
        setNumber(number + 1);
        setNumber(number + 1);
      }}>+3</button>
    </>
  )
}
```

```css
button { display: inline-block; margin: 10px; font-size: 20px; }
h1 { display: inline-block; margin: 10px; width: 30px; text-align: center; }
```

</Sandpack>

Tuttavia, come potresti ricordare dalla sezione precedente, [i valori di stato dei render sono fissi](/learn/state-as-a-snapshot#rendering-takes-a-snapshot-in-time), quindi il valore di `number` all'interno del gestore di eventi del primo render è sempre `0`, indipendentemente da quante volte chiami `setNumber(1)`:

```js
setNumber(0 + 1);
setNumber(0 + 1);
setNumber(0 + 1);
```

Ma c'è un altro fattore che gioca un ruolo qui. **React attende fino a quando *tutto* il codice negli handler degli eventi è stato eseguito prima di elaborare gli aggiornamenti di stato.** È per questo che il re-render avviene solo  *dopo* tutte queste chiamate a `setNumber()`.

Questo potrebbe ricordarti un cameriere che prende un ordine al ristorante. Un cameriere non corre in cucina quando menzioni il primo piatto! Invece, ti lascia finire il tuo ordine, permettono di apportare modifiche e prendono anche ordini da altre persone al tavolo.

<Illustration src="/images/docs/illustrations/i_react-batching.png"  alt="Un elegante cursore in un ristorante effettua un ordine più volte con React, interpretando il ruolo del cameriere. Dopo che chiama setState() più volte, il cameriere annota l'ultimo ordine richiesto come il suo ordine finale." />

Questo ti consente di aggiornare multiple variabili di state--anche da più componenti--senza attivare troppi [re-renders.](/learn/render-and-commit#re-renders-when-state-updates) Ma ciò significa anche che l'UI non viene aggiornata fino a _dopo che_ l'event handler, e qualsiasi codice al suo interno, terminano. Questo comportamento, noto anche come **batching,** rende l'esecuzione della tua React app molto più veloce. Evita anche di dover gestire renders confusionari "eseguiti a metà" dove solo alcune delle variabili sono state aggiornate.

**React non raggruppa gli eventi *multipli* intenzionali come i click**--ogni click viene gestito separatamente. Stai tranquillo che React di solito raggruppa solo quando è sicuro farlo. Questo assicura che, per esempio, che se il primo clic del pulsante disabilita un form, il secondo click non lo invierà di nuovo.

## Aggiornamento dello stesso stato più volte prima del successivo render {/*updating-the-same-state-multiple-times-before-the-next-render*/}

Si tratta di uno use case non comune, but if you would like to update the same state variable multiple times before the next render, instead of passing the *next state value* like `setNumber(number + 1)`, you can pass a *function* that calculates the next state based on the previous one in the queue, like `setNumber(n => n + 1)`. It is a way to tell React to "do something with the state value" instead of just replacing it.

Prova ad incrementare il contatore adesso:

<Sandpack>

```js
import { useState } from 'react';

export default function Counter() {
  const [number, setNumber] = useState(0);

  return (
    <>
      <h1>{number}</h1>
      <button onClick={() => {
        setNumber(n => n + 1);
        setNumber(n => n + 1);
        setNumber(n => n + 1);
      }}>+3</button>
    </>
  )
}
```

```css
button { display: inline-block; margin: 10px; font-size: 20px; }
h1 { display: inline-block; margin: 10px; width: 30px; text-align: center; }
```

</Sandpack>

Qui, `n => n + 1` viene chiamata una **updater function.** Quando la passi ad uno state setter:

1. React mette in coda questa funzione per essere elaborata dopo che tutto il resto del codice nel event handler è stato eseguito.
2. Durante il successivo render, React passa in rassegna la coda e ti fornisce lo stato aggiornato finale.

```js
setNumber(n => n + 1);
setNumber(n => n + 1);
setNumber(n => n + 1);
```

Ecco come React elabora queste righe di codice durante l'esecuzione del event handler:

1. `setNumber(n => n + 1)`: `n => n + 1` è una funzione. React la aggiunge a una coda.
1. `setNumber(n => n + 1)`: `n => n + 1` è una funzione. React la aggiunge a una coda.
1. `setNumber(n => n + 1)`: `n => n + 1` è una funzione. React la aggiunge a una coda.

Quando chiami useState durante il successivo render, React scansiona la coda.  Il precedente `numero` di stato era `0`, quindi è quello che React passa alla prima funzione di aggiornamento come argomento `n`. In seguito React  prende il valore restituito dalla precedente funzione di aggiornamento e lo passa alla successiva come parametro `n`, e cosi via:

|  queued update | `n` | ritorna |
|--------------|---------|-----|
| `n => n + 1` | `0` | `0 + 1 = 1` |
| `n => n + 1` | `1` | `1 + 1 = 2` |
| `n => n + 1` | `2` | `2 + 1 = 3` |

React memorizza 3 come risultato finale e lo restituisce dallo `useState`.

Ecco perché cliccare "+3" nell'esempio sopra incrementa correttamente il valore di 3.
### Cosa succede se aggiorni lo stato dopo averlo sostituito {/*what-happens-if-you-update-state-after-replacing-it*/}

Cosa possiamo dire riguardo questo event handler? Che valore ha `number` al prossimo render?

```js
<button onClick={() => {
  setNumber(number + 5);
  setNumber(n => n + 1);
}}>
```

<Sandpack>

```js
import { useState } from 'react';

export default function Counter() {
  const [number, setNumber] = useState(0);

  return (
    <>
      <h1>{number}</h1>
      <button onClick={() => {
        setNumber(number + 5);
        setNumber(n => n + 1);
      }}>Increase the number</button>
    </>
  )
}
```

```css
button { display: inline-block; margin: 10px; font-size: 20px; }
h1 { display: inline-block; margin: 10px; width: 30px; text-align: center; }
```

</Sandpack>

Ecco cosa l'event handler dice di elaborare a React:

1. `setNumber(number + 5)`: `number` is `0`, quindi  `setNumber(0 + 5)`. React aggiunge *"sostituisci con 5"* alla sua coda.
2. `setNumber(n => n + 1)`: `n => n + 1` è una funzione di aggiornamento. React aggiunge *quella funzione* alla sua coda.

Durante il prossimo render, React va attraverso la state queue:

|   queued update       | `n` | ritorna |
|--------------|---------|-----|
| "replace with `5`" | `0` (unused) | `5` |
| `n => n + 1` | `5` | `5 + 1 = 6` |

React memorizza `6` come risultato finale e lo restituisce da `useState`. 

<Note>

Potresti aver notato che `setState(5)` in realtà funziona come `setState(n => 5)`, ma `n` non viene utilizzato!

</Note>

### osa succede se sostituisci lo stato dopo averlo aggiornato{/*what-happens-if-you-replace-state-after-updating-it*/}

Proviamo un altro esempio. Che valore pensi `number` ha nel prossimo render?

```js
<button onClick={() => {
  setNumber(number + 5);
  setNumber(n => n + 1);
  setNumber(42);
}}>
```

<Sandpack>

```js
import { useState } from 'react';

export default function Counter() {
  const [number, setNumber] = useState(0);

  return (
    <>
      <h1>{number}</h1>
      <button onClick={() => {
        setNumber(number + 5);
        setNumber(n => n + 1);
        setNumber(42);
      }}>Increase the number</button>
    </>
  )
}
```

```css
button { display: inline-block; margin: 10px; font-size: 20px; }
h1 { display: inline-block; margin: 10px; width: 30px; text-align: center; }
```

</Sandpack>

Ecco come React elabora queste righe di codice durante l'esecuzione di questo event handler:

1. `setNumber(number + 5)`: `number` e `0`, quindi `setNumber(0 + 5)`. React aggiunge *"sostituisci con `5`"* nella sua coda.
2. `setNumber(n => n + 1)`: `n => n + 1` è una funzione di aggiornamento. React aggiunge *quella funzione* alla sua coda.
3. `setNumber(42)`: React aggiunge *"rimpiazza con `42`"* alla sua coda.

Durante il prossimo render, React attraversa la coda degli stati:

|   queued update       | `n` | ritorna |
|--------------|---------|-----|
| "replace with `5`" | `0` (unused) | `5` |
| `n => n + 1` | `5` | `5 + 1 = 6` |
| "replace with `42`" | `6` (unused) | `42` |

Then React deposita`42` come risultato finale e lo ritorna da `useState`.

Per riassumere, ecco come puoi pensare a ciò che stai passando al setter di state `setNumber`:

* **Una funzione di aggiornamento** (ad esempio `n => n + 1`) viene aggiunta alla coda.
* **Qualsiasi altro valore** (ad esempio il numero `5`) aggiunge "sostituisci con 5" alla coda, ignorando ciò che è già in coda.

Dopo il completamento dell'handler evento, React attiverà un nuovo rendering. Durante il rendering, React elaborerà la coda. Le funzioni dell'updater vengono eseguite durante il rendering, quindi **le funzioni di aggiornamento devono essere [pure](/learn/keeping-components-pure)** e devono restituire solo il risultato. Non provare a impostare lo state da all'interno di esse o eseguire altri side effects. In Strict Mode, React eseguirà ciascuna funzione dell'updater due volte (ma scarterà il secondo risultato)  per aiutarti a individuare errori.

### Convenzioni di denominazione {/*naming-conventions*/}

È comune nominare l'argomento della funzione updater con le prime lettere della variabile di stato corrispondente:

```js
setEnabled(e => !e);
setLastName(ln => ln.reverse());
setFriendCount(fc => fc * 2);
```

If you prefer more verbose code, another common convention is to repeat the full state variable name, like `setEnabled(enabled => !enabled)`, or to use a prefix like `setEnabled(prevEnabled => !prevEnabled)`.

<Recap>

* Setting state does not change the variable in the existing render, but it requests a new render.
* React processes state updates after event handlers have finished running. This is called batching.
* To update some state multiple times in one event, you can use `setNumber(n => n + 1)` updater function.

</Recap>



<Challenges>

#### Fix a request counter {/*fix-a-request-counter*/}

You're working on an art marketplace app that lets the user submit multiple orders for an art item at the same time. Each time the user presses the "Buy" button, the "Pending" counter should increase by one. After three seconds, the "Pending" counter should decrease, and the "Completed" counter should increase.

However, the "Pending" counter does not behave as intended. When you press "Buy", it decreases to `-1` (which should not be possible!). And if you click fast twice, both counters seem to behave unpredictably.

Why does this happen? Fix both counters.

<Sandpack>

```js
import { useState } from 'react';

export default function RequestTracker() {
  const [pending, setPending] = useState(0);
  const [completed, setCompleted] = useState(0);

  async function handleClick() {
    setPending(pending + 1);
    await delay(3000);
    setPending(pending - 1);
    setCompleted(completed + 1);
  }

  return (
    <>
      <h3>
        Pending: {pending}
      </h3>
      <h3>
        Completed: {completed}
      </h3>
      <button onClick={handleClick}>
        Buy     
      </button>
    </>
  );
}

function delay(ms) {
  return new Promise(resolve => {
    setTimeout(resolve, ms);
  });
}
```

</Sandpack>

<Solution>

Inside the `handleClick` event handler, the values of `pending` and `completed` correspond to what they were at the time of the click event. For the first render, `pending` was `0`, so `setPending(pending - 1)` becomes `setPending(-1)`, which is wrong. Since you want to *increment* or *decrement* the counters, rather than set them to a concrete value determined during the click, you can instead pass the updater functions:

<Sandpack>

```js
import { useState } from 'react';

export default function RequestTracker() {
  const [pending, setPending] = useState(0);
  const [completed, setCompleted] = useState(0);

  async function handleClick() {
    setPending(p => p + 1);
    await delay(3000);
    setPending(p => p - 1);
    setCompleted(c => c + 1);
  }

  return (
    <>
      <h3>
        Pending: {pending}
      </h3>
      <h3>
        Completed: {completed}
      </h3>
      <button onClick={handleClick}>
        Buy     
      </button>
    </>
  );
}

function delay(ms) {
  return new Promise(resolve => {
    setTimeout(resolve, ms);
  });
}
```

</Sandpack>

This ensures that when you increment or decrement a counter, you do it in relation to its *latest* state rather than what the state was at the time of the click.

</Solution>

#### Implement the state queue yourself {/*implement-the-state-queue-yourself*/}

In this challenge, you will reimplement a tiny part of React from scratch! It's not as hard as it sounds.

Scroll through the sandbox preview. Notice that it shows **four test cases.** They correspond to the examples you've seen earlier on this page. Your task is to implement the `getFinalState` function so that it returns the correct result for each of those cases. If you implement it correctly, all four tests should pass.

You will receive two arguments: `baseState` is the initial state (like `0`), and the `queue` is an array which contains a mix of numbers (like `5`) and updater functions (like `n => n + 1`) in the order they were added.

Your task is to return the final state, just like the tables on this page show!

<Hint>

If you're feeling stuck, start with this code structure:

```js
export function getFinalState(baseState, queue) {
  let finalState = baseState;

  for (let update of queue) {
    if (typeof update === 'function') {
      // TODO: apply the updater function
    } else {
      // TODO: replace the state
    }
  }

  return finalState;
}
```

Fill out the missing lines!

</Hint>

<Sandpack>

```js src/processQueue.js active
export function getFinalState(baseState, queue) {
  let finalState = baseState;

  // TODO: do something with the queue...

  return finalState;
}
```

```js src/App.js
import { getFinalState } from './processQueue.js';

function increment(n) {
  return n + 1;
}
increment.toString = () => 'n => n+1';

export default function App() {
  return (
    <>
      <TestCase
        baseState={0}
        queue={[1, 1, 1]}
        expected={1}
      />
      <hr />
      <TestCase
        baseState={0}
        queue={[
          increment,
          increment,
          increment
        ]}
        expected={3}
      />
      <hr />
      <TestCase
        baseState={0}
        queue={[
          5,
          increment,
        ]}
        expected={6}
      />
      <hr />
      <TestCase
        baseState={0}
        queue={[
          5,
          increment,
          42,
        ]}
        expected={42}
      />
    </>
  );
}

function TestCase({
  baseState,
  queue,
  expected
}) {
  const actual = getFinalState(baseState, queue);
  return (
    <>
      <p>Base state: <b>{baseState}</b></p>
      <p>Queue: <b>[{queue.join(', ')}]</b></p>
      <p>Expected result: <b>{expected}</b></p>
      <p style={{
        color: actual === expected ?
          'green' :
          'red'
      }}>
        Your result: <b>{actual}</b>
        {' '}
        ({actual === expected ?
          'correct' :
          'wrong'
        })
      </p>
    </>
  );
}
```

</Sandpack>

<Solution>

This is the exact algorithm described on this page that React uses to calculate the final state:

<Sandpack>

```js src/processQueue.js active
export function getFinalState(baseState, queue) {
  let finalState = baseState;

  for (let update of queue) {
    if (typeof update === 'function') {
      // Apply the updater function.
      finalState = update(finalState);
    } else {
      // Replace the next state.
      finalState = update;
    }
  }

  return finalState;
}
```

```js src/App.js
import { getFinalState } from './processQueue.js';

function increment(n) {
  return n + 1;
}
increment.toString = () => 'n => n+1';

export default function App() {
  return (
    <>
      <TestCase
        baseState={0}
        queue={[1, 1, 1]}
        expected={1}
      />
      <hr />
      <TestCase
        baseState={0}
        queue={[
          increment,
          increment,
          increment
        ]}
        expected={3}
      />
      <hr />
      <TestCase
        baseState={0}
        queue={[
          5,
          increment,
        ]}
        expected={6}
      />
      <hr />
      <TestCase
        baseState={0}
        queue={[
          5,
          increment,
          42,
        ]}
        expected={42}
      />
    </>
  );
}

function TestCase({
  baseState,
  queue,
  expected
}) {
  const actual = getFinalState(baseState, queue);
  return (
    <>
      <p>Base state: <b>{baseState}</b></p>
      <p>Queue: <b>[{queue.join(', ')}]</b></p>
      <p>Expected result: <b>{expected}</b></p>
      <p style={{
        color: actual === expected ?
          'green' :
          'red'
      }}>
        Your result: <b>{actual}</b>
        {' '}
        ({actual === expected ?
          'correct' :
          'wrong'
        })
      </p>
    </>
  );
}
```

</Sandpack>

Now you know how this part of React works!

</Solution>

</Challenges>