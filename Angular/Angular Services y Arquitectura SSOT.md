# Angular: Services, Signals como Estado y Arquitectura SSOT

> 💡 En esta clase se construye una arquitectura real de servicios en Angular, donde el servicio actúa como **Single Source of Truth** (única fuente de verdad) usando `signal()` como estado interno. Los componentes leen del servicio y reaccionan automáticamente a los cambios.

---

## Single Source of Truth (SSOT)

El patrón **Single Source of Truth** establece que **el estado de la aplicación vive en un único lugar**: el servicio. Los componentes no almacenan datos, solo los leen y muestran.

```
Servicio (SSOT)
  └── signal({ characters: Map })   ← Estado centralizado
        ↑                ↑
  getCharacters()   updateCharacter()   ← Mutations
        ↓
  Componente (solo lee con computed())
```

> 💡 Ventaja: si varios componentes necesitan los mismos datos, todos apuntan al mismo servicio. Cambiar un dato en el servicio actualiza todos los componentes que lo consumen automáticamente, sin pasar props ni emitir eventos.

---

## Promise vs Observable

```typescript
// Promise -> Promete que algo va a pasar en el futuro.
// Se resuelve UNA sola vez (bien o mal) y luego termina.
const promesa = fetch('https://api.com/data')
  .then(res => res.json())  // Solo se ejecuta una vez

// Observable -> Es un canal de comunicación abierto.
// Puede emitir MÚLTIPLES valores a lo largo del tiempo.
// Puedes suscribirte, recibir datos, y cancelar cuando quieras.
const observable = this.http.get('https://api.com/data')
observable.subscribe(data => console.log(data))  // Escucha mientras dure
```

> 💡 En Angular, `HttpClient` devuelve Observables (no Promises). Los observables son más potentes: permiten cancelaciones, transformaciones con RxJS (`map`, `filter`, `catchError`) y manejar streams de datos en tiempo real.

| | Promise | Observable |
|---|---|---|
| Emite valores | Una vez | Múltiples veces |
| Cancelable | No | Sí (`.unsubscribe()`) |
| Operadores | `.then()` / `.catch()` | `map`, `filter`, `catchError`... |
| Lazy | No (empieza al crearse) | Sí (empieza al suscribirse) |

---

## Modelo de datos: Interface

Define la forma de los datos con una `interface` de TypeScript. Esto da tipado estricto en toda la app.

```typescript
// src/app/models/character.model.ts
export interface Character {
    id: number
    name: string;
    status: string
    image: string;
}
```

> 💡 Las interfaces en TypeScript solo existen en tiempo de compilación. No generan código JavaScript. Son una herramienta de tipado para detectar errores antes de ejecutar la app.

---

## Barrel Exports (index.ts)

El patrón **barrel** agrupa todos los exports de una carpeta en un `index.ts`. Esto limpia los imports.

```typescript
// src/app/models/index.ts
export * from './character.model';

// src/app/services/index.ts
export * from './character-service.service';
```

```typescript
// Sin barrel (imports sucios)
import { Character } from '../models/character.model';
import { CharacterServiceService } from './services/character-service.service';

// Con barrel (imports limpios)
import { Character } from '../models';
import { CharacterServiceService } from './services';
```

> 💡 Los barrel exports mejoran la legibilidad y permiten reorganizar archivos internamente sin romper imports externos. Es una convención muy común en proyectos Angular escalables.

---

## `of()` de RxJS — Crear un Observable desde un valor

`of()` convierte cualquier valor (array, objeto, primitivo) en un Observable que emite ese valor inmediatamente y completa.

```typescript
import { of } from 'rxjs';

// Simula una llamada a API que devuelve datos
of(['Rick', 'Morty', 'Summer'])
  .subscribe(result => {
    console.log(result); // ['Rick', 'Morty', 'Summer']
    // Se imprime una vez y el observable completa
  });
```

> 💡 `of()` es muy útil durante el desarrollo para simular respuestas de API sin hacer llamadas HTTP reales. Cuando la API esté lista, solo cambias `of(mockData)` por `this.http.get<T>(url)`.

---

## Map de JavaScript como estructura de datos

En lugar de un `array`, el servicio usa un `Map<number, Character>` para almacenar los personajes. Un `Map` permite acceder a cualquier elemento por clave en O(1) sin iterar todo el array.

```typescript
// Map<Clave, Valor>
const characters = new Map<number, Character>();

// Añadir / actualizar (si la clave ya existe, sobreescribe)
characters.set(1, { id: 1, name: 'Rick', status: 'Alive', image: '...' });
characters.set(2, { id: 2, name: 'Morty', status: 'Alive', image: '...' });

// Obtener por ID — O(1), sin buscar en todo el array
characters.get(1); // { id: 1, name: 'Rick', ... }

// Eliminar por ID
characters.delete(1);

// Convertir a array para iterar o mostrar en el template
Array.from(characters.values()); // [{ id: 1... }, { id: 2... }]
```

> 💡 Usar `Map` en vez de `array` para el estado interno es una decisión de arquitectura: las operaciones de búsqueda, actualización y eliminación por ID son más eficientes. Para renderizar en el template, se convierte a array con `Array.from()`.

---

## Servicio con Signal como Estado (SSOT en práctica)

El servicio centraliza el estado usando `signal()`. Todos los métodos CRUD modifican ese signal.

```typescript
// src/app/services/character-service.service.ts
import { HttpClient } from '@angular/common/http';
import { inject, Injectable, signal } from '@angular/core';
import { of } from 'rxjs';
import { Character } from '../models';

@Injectable({
  providedIn: 'root'  // Singleton: una única instancia en toda la app
})
export class CharacterServiceService {

  // El estado vive aquí. Un signal que contiene un objeto con un Map de personajes.
  // Esto es el "Single Source of Truth" del servicio.
  state = signal({
    characters: new Map<number, Character>()
  })

  constructor(){
    this.getCharacters()  // Se cargan los datos al instanciar el servicio
  }

  // READ — Obtener un personaje por ID
  getCharacterByID(id: number) {
    return this.state().characters.get(id)
  }

  // READ — Obtener todos los personajes como array (para iterar en el template)
  getFormattedCharacters() {
    return Array.from(this.state().characters.values())
  }

  // READ (inicial) — Cargar datos (aquí mock, en producción sería this.http.get())
  getCharacters() {
    const mockCharacters: Character[] = [
      { id: 1, name: 'Rick Sanchez',  status: 'Alive', image: 'https://rickandmortyapi.com/api/character/avatar/1.jpeg' },
      { id: 2, name: 'Morty Smith',   status: 'Alive', image: 'https://rickandmortyapi.com/api/character/avatar/2.jpeg' },
      { id: 3, name: 'Summer Smith',  status: 'Alive', image: 'https://rickandmortyapi.com/api/character/avatar/3.jpeg' },
      { id: 4, name: 'Beth Smith',    status: 'Alive', image: 'https://rickandmortyapi.com/api/character/avatar/4.jpeg' },
    ]

    of(mockCharacters)       // Simula una respuesta de API como Observable
      .subscribe(result => {
        // Cargamos cada personaje en el Map usando su ID como clave
        result.forEach(character => this.state().characters.set(character.id, character))
        // Actualizamos el signal para disparar la reactividad
        this.state.set({ characters: this.state().characters })
      })
  }

  // UPDATE — Actualizar un personaje existente
  updateCharacter(character: Character): void {
    const updatedCharacter = { ...character }; // Spread: crea una copia para no mutar el original

    of(updatedCharacter).subscribe((result) => {
      this.state.update((state) => {
        // state.update() recibe el estado actual y devuelve el nuevo estado
        state.characters.set(result.id, result)
        return ({ characters: state.characters })
      })
    })
  }

  // DELETE — Eliminar un personaje por ID
  deleteCharacter(id: number) {
    of({ status: 200 })  // Simula respuesta 200 OK del servidor
      .subscribe(() => {
        this.state.update((state) => {
          state.characters.delete(id)
          return ({ characters: state.characters })
        })
      })
  }
}
```

---

## `signal.set()` vs `signal.update()`

Dos formas de modificar un signal:

```typescript
// signal.set() — Establece un valor nuevo directamente
// Úsalo cuando conoces el valor final completo
this.state.set({ characters: this.state().characters })

// signal.update() — Recibe el estado actual y devuelve el nuevo
// Úsalo cuando el nuevo valor depende del valor anterior
this.state.update((state) => {
  state.characters.delete(id)
  return ({ characters: state.characters })
})
```

> 💡 `update()` es preferible cuando operas sobre el estado actual (añadir, eliminar, modificar un elemento). `set()` es para reemplazar el estado completamente por un valor conocido.

---

## `computed()` en el Componente

El componente no llama directamente al Observable ni gestiona subscripciones. Usa `computed()` para derivar un signal a partir del estado del servicio.

```typescript
// src/app/app.component.ts
import { ChangeDetectionStrategy, Component, computed, inject, Signal } from '@angular/core';
import { CharacterServiceService } from './services';
import { Character } from './models';
import { toSignal } from '@angular/core/rxjs-interop';

@Component({
  selector: 'app-root',
  standalone: true,
  imports: [RouterOutlet],
  templateUrl: './app.component.html',
  styleUrl: './app.component.css',
  changeDetection: ChangeDetectionStrategy.OnPush
  // OnPush: Angular solo comprueba cambios cuando las referencias cambian.
  // Mejora el rendimiento evitando ciclos de detección innecesarios.
})
export class AppComponent {
  characterService = inject(CharacterServiceService)

  // computed() crea un signal derivado. Se recalcula automáticamente
  // cuando el signal del servicio (state) cambia.
  characters: Signal<Character[] | undefined> = computed(
    () => this.characterService.getFormattedCharacters()
  )
}
```

> 💡 `computed()` es reactivo: si el `state` del servicio cambia (nuevo personaje, borrado, actualizado), `characters` se recalcula automáticamente sin necesidad de subscripciones ni `ngOnInit`.

---

## `ChangeDetectionStrategy.OnPush`

```typescript
@Component({
  changeDetection: ChangeDetectionStrategy.OnPush
  // Default: Angular comprueba TODOS los componentes en CADA evento (click, timer, http...)
  // OnPush:  Angular solo comprueba este componente cuando:
  //   1. Cambia una referencia de @Input()
  //   2. El componente emite un evento
  //   3. Un Observable con async pipe emite un valor
  //   4. Un Signal cambia (caso de esta clase)
})
```

> 💡 Con Signals + OnPush, Angular sabe exactamente qué componentes hay que actualizar. El resultado es una aplicación más rápida porque Angular no "revisa todo" en cada interacción del usuario.

---

## `@let` — Variables locales en el template

Nueva sintaxis de Angular (v18+) para declarar variables locales en el template.

```html
<!-- Sin @let: llamamos al signal dos veces (dos lecturas) -->
@for (character of characters(); track character.id) {
  <div>{{ character.name }}</div>
}

<!-- Con @let: leemos el signal una vez y lo guardamos en una variable local -->
@let charactersLocal = characters();
@for (character of charactersLocal; track character.id) {
  <div>
    <h2>{{ character.name }}</h2>
    <img [src]="character.image" alt="{{ character.name }}" />
  </div>
}
```

> 💡 `@let` es útil para evitar llamar al mismo método/signal múltiples veces en el template. También mejora la legibilidad cuando el valor tiene un nombre largo o complejo.

---

## Adapters — Capa de transformación de datos

Un **adapter** es una función pura que transforma datos de la API a la forma que necesita la app (o viceversa). Separa la lógica de transformación del servicio.

```typescript
// src/app/adapters/character-adapter.ts
import { Character } from "../models"

// Función pura: recibe un array de Character y devuelve uno nuevo transformado
// En este caso, convierte todos los nombres a MAYÚSCULAS
export const characterAdapter = (characters: Character[]) =>
  characters.map(character => ({
    ...character,               // Spread: copia todas las propiedades
    name: character.name.toUpperCase()  // Solo modifica 'name'
  }))
```

```typescript
// Uso en el servicio:
import { characterAdapter } from '../adapters/character-adapter';

// Al recibir datos de la API, los pasamos por el adapter antes de guardarlos
const adapted = characterAdapter(rawCharacters);
adapted.forEach(char => this.state().characters.set(char.id, char));
```

> 💡 El patrón Adapter es especialmente útil cuando la API devuelve datos en un formato diferente al que usa la app. Por ejemplo: la API devuelve `first_name` y `last_name`, pero la app necesita `fullName`. El adapter hace esa conversión en un único lugar.

---

## `toSignal` — Convertir Observables en Signals

`toSignal` del paquete `@angular/core/rxjs-interop` convierte un Observable en un Signal. Gestiona la suscripción automáticamente (sin necesidad de `.subscribe()` ni `ngOnDestroy`).

```typescript
import { toSignal } from '@angular/core/rxjs-interop';
import { HttpClient } from '@angular/common/http';

@Injectable({ providedIn: 'root' })
export class CharacterService {
  private http = inject(HttpClient);

  // El Observable de HttpClient se convierte en un Signal
  // Angular gestiona la suscripción y la cancela automáticamente
  characters = toSignal(
    this.http.get<Character[]>('https://rickandmortyapi.com/api/character'),
    { initialValue: [] }  // Valor inicial mientras llegan los datos
  )
}

// En el componente:
export class AppComponent {
  service = inject(CharacterService);
  // characters es un Signal<Character[]>, no un Observable
  // Se usa directamente en el template: {{ service.characters() }}
}
```

> 💡 **`toSignal` vs `.subscribe()`**: Con `toSignal` no necesitas gestionar la subscripción ni preocuparte por memory leaks. Angular crea y destruye la suscripción automáticamente según el ciclo de vida del contexto de inyección.

---

## Flujo completo de la arquitectura

```
API / Mock Data
      ↓
  of() / http.get()   ← Observable
      ↓
  .subscribe()        ← Recibimos los datos
      ↓
  state (signal)      ← Guardamos en el Map (SSOT)
      ↓
  computed()          ← El componente deriva el array del signal
      ↓
  @let + @for         ← El template renderiza la lista
```

```
Acción del usuario (delete, update)
      ↓
  Método del servicio (deleteCharacter, updateCharacter)
      ↓
  state.update()      ← Mutamos el signal
      ↓
  computed() se recalcula automáticamente
      ↓
  Template se actualiza (OnPush detecta el cambio del signal)
```

---

## Resumen de conceptos clave

| Concepto | Qué es | Para qué sirve |
|---|---|---|
| **SSOT** | Estado centralizado en el servicio | Fuente única de verdad para todos los componentes |
| **`signal()`** | Estado reactivo en el servicio | Almacenar y actualizar el estado del servicio |
| **`computed()`** | Signal derivado en el componente | Leer y transformar el estado del servicio |
| **`Map<K,V>`** | Estructura de datos clave-valor | Acceso O(1) por ID, sin iterar arrays |
| **`of()`** | Crea un Observable desde un valor | Simular APIs durante el desarrollo |
| **`state.update()`** | Modifica el signal basándose en su valor actual | CRUD: añadir, eliminar, actualizar elementos |
| **`toSignal()`** | Convierte Observable → Signal | Integrar RxJS con la reactividad de Signals |
| **`OnPush`** | Estrategia de detección de cambios | Mejorar rendimiento: solo actualiza cuando hay cambio real |
| **`@let`** | Variable local en el template | Evitar lecturas duplicadas de signals/métodos |
| **Adapter** | Función pura de transformación | Separar lógica de transformación del servicio |
| **Barrel (index.ts)** | Agrupa exports de una carpeta | Imports limpios y refactorización más fácil |
