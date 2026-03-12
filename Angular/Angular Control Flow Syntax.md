# Angular: Control Flow Syntax
> 💡 Introducida en Angular 17, la **Control Flow Syntax** es la nueva forma de escribir lógica condicional y de repetición directamente en los templates HTML, usando bloques `@if`, `@for`, `@switch` y `@defer`. Sustituye a las antiguas directivas estructurales (`*ngIf`, `*ngFor`, `ngSwitch`) y elimina la necesidad de importarlas.

---

## @if / @else

Equivalente moderno de `*ngIf`. Muestra u oculta contenido según una condición booleana.

- **Antes (directiva estructural):**
```html
<!-- Necesitaba importar CommonModule o NgIf -->
<div *ngIf="isVisible; else elseBlock">Se ve</div>
<ng-template #elseBlock>Contenido oculto</ng-template>
```

- **Ahora (Control Flow Syntax):**
```html
@if (isVisible) {
    <div>Se ve</div>
} @else {
    <div>Contenido oculto</div>
}
```

```typescript
@Component({
    selector: 'app-if',
    standalone: true,
    imports: [], /* Ya no necesitas importar NgIf */
    templateUrl: './if.component.html',
})
export class IfComponent {
    protected isVisible = true;
    /* 'protected' permite que el template acceda a la propiedad,
       pero no la expone fuera del componente */
}
```

> 💡 También puedes encadenar condiciones con `@else if`:
> ```html
> @if (role === 'admin') {
>     <p>Panel de administración</p>
> } @else if (role === 'user') {
>     <p>Panel de usuario</p>
> } @else {
>     <p>Acceso denegado</p>
> }
> ```


---

## @for / @empty

Equivalente moderno de `*ngFor`. Itera sobre arrays y renderiza un elemento por cada ítem. Incluye `@empty` para cuando el array está vacío.

- **Antes (directiva estructural):**
```html
<!-- *ngFor no tenía forma nativa de manejar arrays vacíos -->
<ul>
    <li *ngFor="let name of names; let i = index">
        {{ name }}
    </li>
</ul>
```

- **Ahora (Control Flow Syntax):**
```html
<ul>
    @for (name of names; track name) {
        <li>{{ name }}</li>
    } @empty {
        <li>No names</li>
    }
</ul>
```

```typescript
@Component({
    selector: 'app-for',
    standalone: true,
    imports: [], /* Ya no necesitas importar NgFor */
    templateUrl: './for.component.html',
})
export class ForComponent {
    names: string[] = ['John', 'Jane', 'Doe', 'Buck'];
}
```

> 💡 **`track` es obligatorio** en `@for`. Le indica a Angular cómo identificar de forma única cada elemento para optimizar las actualizaciones del DOM. Usa la propiedad más estable del elemento (idealmente un `id`):
> ```html
> <!-- Si los ítems tienen ID, úsalo siempre como track -->
> @for (user of users; track user.id) {
>     <p>{{ user.name }}</p>
> }
> ```
> Sin `track`, Angular tendría que re-renderizar todos los elementos al menor cambio.

> 💡 Variables disponibles dentro de `@for`:
> ```html
> @for (item of items; track item.id; let i = $index, isFirst = $first, isLast = $last, isEven = $even) {
>     <p>{{ i }}: {{ item.name }} — {{ isFirst ? 'Primero' : '' }} {{ isLast ? 'Último' : '' }}</p>
> }
> ```


---

## @switch / @case

Equivalente moderno de `[ngSwitch]`. Evalúa una expresión y renderiza el bloque que coincida con el valor.

- **Antes (directiva estructural):**
```html
<div [ngSwitch]="value">
    <p *ngSwitchCase="'option 1'">Option 1 selected</p>
    <p *ngSwitchCase="'option 2'">Option 2 selected</p>
    <p *ngSwitchDefault>Otro valor</p>
</div>
```

- **Ahora (Control Flow Syntax):**
```html
@switch (value) {
    @case ('option 1') {
        <p>Option 1 selected</p>
    }
    @case ('option 2') {
        <p>Option 2 selected</p>
    }
    @case ('option 3') {
        <p>Option 3 selected</p>
    }
    @default {
        <p>Valor no reconocido</p>
    }
}
```

```typescript
@Component({
    selector: 'app-switch',
    standalone: true,
    imports: [],
    templateUrl: './switch.component.html',
})
export class SwitchComponent {
    protected value = 'option 1';
}
```

> 💡 `@switch` no necesita `break` como en JavaScript. Cada `@case` es exclusivo y Angular no ejecuta los siguientes bloques una vez que encuentra una coincidencia.


---

## @defer

`@defer` permite cargar contenido de forma **diferida (lazy loading)**. El contenido dentro del bloque no se descarga ni renderiza hasta que se cumpla la condición o trigger definido. Esto mejora el rendimiento inicial de la aplicación.

```html
@defer (when isImageVisible) {
    <!-- Este contenido solo se carga cuando isImageVisible sea true -->
    <img src="angular-logo.png" alt="Angular Logo">
}

@if (!isImageVisible) {
    <button (click)="showImage()">Mostrar Imagen</button>
} @else {
    <button (click)="hideImage()">Ocultar Imagen</button>
}
```

```typescript
@Component({
    selector: 'app-defer',
    standalone: true,
    imports: [],
    templateUrl: './defer.component.html',
})
export class DeferComponent {
    /* Permite cargar contenido de forma diferida según una condición.
       Mejora el rendimiento: retrasa la carga de partes no críticas de la app */
    isImageVisible = false;

    showImage() {
        this.isImageVisible = true;
    }
    hideImage() {
        this.isImageVisible = false;
    }
}
```

> 💡 `@defer` es especialmente útil para componentes pesados (gráficos, mapas, editores de texto) que no son necesarios en la carga inicial de la página. Solo se descargan del servidor cuando el usuario los necesita.


---

## @placeholder

Define el contenido que se muestra **mientras el bloque `@defer` aún no se ha activado** (antes de que se cumpla la condición). Es el estado inicial visible.

```html
@defer (when isImageVisible) {
    <img src="angular-logo.png" alt="Angular Logo">
} @placeholder {
    <p>Cargando...</p>
    <!-- Esto se ve desde el principio, antes de que isImageVisible sea true -->
}

@if (!isImageVisible) {
    <button (click)="showImage()">Mostrar Imagen</button>
}
```

```typescript
@Component({
    selector: 'app-placeholder',
    standalone: true,
    imports: [],
    templateUrl: './placeholder.component.html',
})
export class PlaceholderComponent {
    isImageVisible = false;

    showImage() {
        setTimeout(() => {
            this.isImageVisible = true;
        }, 4000); /* Simula una carga asíncrona de 4 segundos */
    }
}
```

> 💡 **Diferencia entre `@placeholder` y `@loading`:**
> - `@placeholder` = lo que ves **antes** de activar el defer (estado inicial)
> - `@loading` = lo que ves **mientras** el contenido se está descargando/cargando (estado de transición)


---

## @loading

Define el contenido que se muestra **durante el proceso de carga** del bloque `@defer`. Admite los parámetros `after` y `minimum` para controlar cuándo y cuánto tiempo se muestra.

```html
@defer (when isContentReady) {
    <!-- Se muestra cuando el contenido está listo -->
    <img src="defer-icon.svg" alt="defer icon">
} @loading (after 4000ms; minimum 4000ms) {
    <!-- 'after 4000ms': el loading solo aparece si la carga tarda más de 4 segundos (evita flash) -->
    <!-- 'minimum 4000ms': aunque cargue antes, el loading se mantiene al menos 4 segundos (evita parpadeos) -->
    <p>Cargando...</p>
} @placeholder {
    <div>Preparando el contenido...</div>
}
```

```typescript
@Component({
    selector: 'app-loading',
    standalone: true,
    imports: [],
    templateUrl: './loading.component.html',
})
export class LoadingComponent implements OnInit {
    isContentReady = false;

    ngOnInit(): void {
        setTimeout(() => {
            this.isContentReady = true; /* Simula que los datos llegan después de 4 segundos */
        }, 4000);
    }
}
```

> 💡 Los parámetros de `@loading` son opcionales pero muy útiles para la UX:
> - `after Xms` → Espera X milisegundos antes de mostrar el spinner (evita flashes en cargas rápidas)
> - `minimum Xms` → Muestra el spinner al menos X milisegundos (evita que aparezca y desaparezca muy rápido)


---

## @error

Define el contenido que se muestra si **ocurre un error** durante la carga del bloque `@defer`. Completa el ciclo de vida: placeholder → loading → (contenido o error).

```html
@defer (when isContentReady) {
    <p>Componente cargado correctamente</p>
} @loading (after 100ms; minimum 100ms) {
    <p>Cargando...</p>
} @placeholder {
    <p>Preparando el contenido</p>
} @error {
    <p>Ha ocurrido un error cargando el componente, inténtelo más tarde</p>
}
```

```typescript
@Component({
    selector: 'app-error-component',
    standalone: true,
    imports: [],
    templateUrl: './error-component.component.html',
})
export class ErrorComponentComponent {
    isContentReady = false;
    isContentError = false;

    onInit() {
        setTimeout(() => {
            this.isContentReady = true; /* Contenido listo a los 3s */
        }, 3000);
        setTimeout(() => {
            throw new Error('Error al cargar el componente'); /* Error simulado a los 5s */
        }, 5000);
    }
}
```

> 💡 El ciclo de vida completo de un bloque `@defer`:
> ```
> placeholder → (se activa el trigger) → loading → contenido ✅
>                                                 → @error ❌
> ```


---

## @defer — Triggers avanzados

Además de `when` (condición booleana), `@defer` ofrece múltiples **triggers** con la palabra clave `on` para controlar cuándo se activa la carga diferida.

### `on idle` — Cuando el navegador está inactivo
```html
@defer (on idle) {
    <p>Componente pesado cargado cuando el navegador está libre</p>
} @placeholder {
    <p>Esperando a que el navegador esté inactivo...</p>
}
/* Se activa cuando el navegador termina sus tareas y entra en estado idle.
   Ideal para cargar componentes no críticos sin afectar el rendimiento inicial */
```

### `on viewport` — Cuando el elemento entra en pantalla
```html
<!-- Se activa cuando el @placeholder entra en el viewport del usuario -->
@defer (on viewport) {
    <p>Componente cargado al hacer scroll hasta aquí</p>
} @placeholder {
    <!-- El placeholder debe tener el tamaño del elemento final para evitar saltos de layout -->
    <div style="height: 200px">Scroll hasta aquí...</div>
}

<!-- Con un elemento de referencia específico -->
<div #triggerElement>Elemento de referencia</div>
@defer (on viewport(triggerElement)) {
    <p>Cargado cuando triggerElement entra en pantalla</p>
} @placeholder {
    <p>Esperando...</p>
}
```

### `on interaction` — Cuando el usuario interactúa
```html
<!-- Se activa al primer clic o interacción del usuario en la página -->
@defer (on interaction) {
    <p>Componente cargado tras la primera interacción</p>
} @placeholder {
    <p>Interactúa con la página para cargar</p>
}

<!-- Con un elemento específico -->
<div #interactionElement>Haz clic aquí</div>
@defer (on interaction(interactionElement)) {
    <p>Cargado al interactuar con el elemento</p>
} @placeholder {
    <p>Haz clic en "Haz clic aquí"</p>
}
```

### `on hover` — Cuando el usuario pasa el ratón
```html
<!-- Se activa al pasar el ratón sobre el @placeholder -->
@defer (on hover) {
    <p>Componente cargado al pasar el ratón</p>
} @placeholder {
    <p>Pasa el ratón por aquí</p>
}

<!-- Con un elemento específico -->
<div #specificHover>Pásame por encima</div>
@defer (on hover(specificHover)) {
    <p>Componente cargado al pasar el ratón sobre specificHover</p>
} @placeholder {
    <p>Esperando hover...</p>
}
```

### `on timer` — Tras un tiempo determinado
```html
<!-- Se activa automáticamente después de 5 segundos -->
@defer (on timer(5000ms)) {
    <p>Componente cargado tras 5 segundos</p>
} @placeholder {
    <p>Cargando en 5 segundos...</p>
}
```

### `prefetch` — Precarga en segundo plano
```html
<!-- Se muestra al interactuar, pero se precarga en segundo plano cuando el navegador está idle -->
@defer (on interaction; prefetch on idle) {
    <p>Componente que se muestra al interactuar, pero que ya estaba precargado</p>
} @placeholder {
    <p>Interactúa conmigo</p>
}
/* prefetch on idle: descarga el JS del componente cuando el navegador está libre,
   así cuando el usuario interactúe, la carga será instantánea */
```

> 💡 **Resumen de todos los triggers de `@defer`:**
>
> | Trigger | Cuándo se activa |
> |---|---|
> | `when condicion` | Cuando una variable booleana es `true` |
> | `on idle` | Cuando el navegador está inactivo |
> | `on viewport` | Cuando el elemento entra en la pantalla |
> | `on viewport(ref)` | Cuando un elemento de referencia entra en la pantalla |
> | `on interaction` | Al primer clic/interacción del usuario |
> | `on interaction(ref)` | Al interactuar con un elemento específico |
> | `on hover` | Al pasar el ratón sobre el placeholder |
> | `on hover(ref)` | Al pasar el ratón sobre un elemento específico |
> | `on timer(Xms)` | Automáticamente tras X milisegundos |
> | `prefetch on idle` | Precarga el JS en segundo plano mientras el navegador está libre |


---

## Comparativa: Sintaxis antigua vs Control Flow Syntax

> 💡 La nueva sintaxis NO requiere importar módulos en el componente (`NgIf`, `NgFor`, `NgSwitch`). Angular la reconoce de forma nativa en cualquier template.

| Funcionalidad | Sintaxis antigua | Control Flow Syntax |
|---|---|---|
| Condicional | `*ngIf="cond"` | `@if (cond) { }` |
| Condicional con else | `*ngIf="cond; else ref"` + `<ng-template #ref>` | `@if (cond) { } @else { }` |
| Bucle | `*ngFor="let x of arr"` | `@for (x of arr; track x) { }` |
| Array vacío | *(no existía nativamente)* | `@empty { }` dentro de `@for` |
| Switch | `[ngSwitch]` + `*ngSwitchCase` | `@switch (val) { @case (x) { } }` |
| Carga diferida | *(no existía)* | `@defer` + triggers |
