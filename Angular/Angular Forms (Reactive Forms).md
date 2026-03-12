# Angular: Reactive Forms

> 💡 Angular ofrece dos enfoques para formularios: **Template-driven** (con `ngModel`, más sencillo) y **Reactive Forms** (con `ReactiveFormsModule`, más potente y escalable). Esta clase se centra en Reactive Forms porque dan tipado estricto, son más fáciles de testear y permiten construir formularios dinámicos complejos.

---

## ¿Por qué Reactive Forms?

| | Template-driven | Reactive Forms |
|---|---|---|
| Tipado TypeScript | Débil | Fuerte (genéricos) |
| Lógica del form | En el HTML | En el componente (.ts) |
| Testabilidad | Difícil | Fácil |
| Formularios dinámicos | Complicado | Natural (FormArray) |
| Validación | Atributos HTML | Funciones TypeScript |

---

## Configuración: ReactiveFormsModule

Para usar Reactive Forms, hay que importar `ReactiveFormsModule` en el componente (o en un módulo compartido).

```typescript
import { ReactiveFormsModule } from '@angular/forms';

@Component({
  standalone: true,
  imports: [ReactiveFormsModule], /* Sin este import, las directivas formGroup, formControlName, etc. no funcionan */
  template: `...`
})
export class AppComponent {}
```

---

## FormControl — El bloque básico

Un `FormControl` representa un campo individual del formulario (un `<input>`, `<select>`, etc.).

```typescript
import { FormControl, Validators } from '@angular/forms';

// FormControl<Tipo> -> tipado fuerte, sabe qué tipo de dato almacena
const nombre = new FormControl<string>('', [Validators.required]);
const edad   = new FormControl<number>(0, [Validators.min(0)]);

// Leer el valor
console.log(nombre.value); // ''

// Cambiar el valor
nombre.setValue('Mario');

// Comprobar estado
nombre.valid;   // false (está vacío y es required)
nombre.invalid; // true
nombre.dirty;   // true si el usuario ya ha escrito algo
nombre.touched; // true si el usuario ha hecho blur (salido del campo)
```

---

## FormGroup — Agrupar controles

Un `FormGroup` agrupa varios `FormControl` relacionados bajo un mismo objeto.

```typescript
import { FormGroup, FormControl, Validators } from '@angular/forms';

const loginForm = new FormGroup({
  email:    new FormControl('', [Validators.required, Validators.email]),
  password: new FormControl('', [Validators.required, Validators.minLength(6)])
});

// Leer todos los valores de golpe
console.log(loginForm.value); // { email: '', password: '' }

// Acceder a un control específico
loginForm.controls.email.setValue('mario@ejemplo.com');

// El grupo es válido solo si TODOS los controles son válidos
loginForm.valid; // false mientras haya campos inválidos
```

```html
<!-- En el template, se vincula el formGroup al <form> y cada control por nombre -->
<form [formGroup]="loginForm" (ngSubmit)="onSubmit()">
  <input formControlName="email" />
  <input formControlName="password" type="password" />
  <button type="submit" [disabled]="loginForm.invalid">Entrar</button>
</form>
```

---

## FormBuilder y NonNullableFormBuilder — Crear forms más fácil

Crear controles con `new FormControl()` puede ser repetitivo. `FormBuilder` es un servicio que simplifica la creación.

```typescript
import { inject } from '@angular/core';
import { FormBuilder, NonNullableFormBuilder } from '@angular/forms';

export class AppComponent {
  fb = inject(FormBuilder)

  form = this.fb.group({
    name:  this.fb.control(''),
    email: this.fb.control('')
  })
}
```

> 💡 **`NonNullableFormBuilder`** es la versión mejorada: garantiza que los controles **nunca tengan valor `null`**. Con el `FormBuilder` normal, si haces `.reset()`, los controles pueden volver a `null`. Con `NonNullableFormBuilder`, vuelven a su valor inicial.

```typescript
fb = inject(NonNullableFormBuilder)
// Ahora form.controls.name.value es siempre string, nunca string | null
```

---

## Interfaces tipadas para formularios

Para tener tipado fuerte en `FormGroup`, se puede definir una interfaz que describa la forma exacta del formulario:

```typescript
import { FormControl, FormGroup } from '@angular/forms';

// Cada propiedad de la interfaz es un FormControl del tipo correspondiente
export interface ItemForm {
  id:    FormControl<number>;  // Campo numérico
  name:  FormControl<string>;  // Campo de texto
  value: FormControl<number>;  // Campo numérico
}

// Tipo alias para el FormGroup tipado. Mejora la legibilidad
export type CustomFormGroup = FormGroup<ItemForm>
```

```typescript
// Ahora el FormGroup conoce exactamente qué controles tiene
const itemForm: CustomFormGroup = fb.group<ItemForm>({
  id:    fb.control(1),
  name:  fb.control('', [Validators.required]),
  value: fb.control(0,  [Validators.required, Validators.min(0)])
})

// TypeScript sabe que .controls.name es FormControl<string>, no any
itemForm.controls.name.value // string ✅
itemForm.controls.name.setValue(42) // ❌ Error en compilación: 42 no es string
```

---

## FormArray — Formularios dinámicos

`FormArray` es un array de controles (pueden ser `FormControl`, `FormGroup`, o incluso otros `FormArray`). Permite añadir y quitar filas de formulario dinámicamente.

```typescript
import { FormArray, FormGroup, FormControl } from '@angular/forms';

// FormArray<CustomFormGroup> = array tipado de grupos de formulario
form: FormGroup<{items: FormArray<CustomFormGroup>}> = this.fb.group({
  items: this.fb.array<CustomFormGroup>([]) // Empieza vacío
})

addItem() {
  const nuevoGrupo = this.fb.group<ItemForm>({
    id:    this.fb.control(1),
    name:  this.fb.control('', [Validators.required]),
    value: this.fb.control(0,  [Validators.required, Validators.min(0)])
  })

  this.form.controls.items.push(nuevoGrupo) // Añade el grupo al array
}

removeItem(index: number) {
  this.form.controls.items.removeAt(index) // Elimina por posición
}
```

> 💡 El `FormArray` tiene los métodos principales:
> - `.push(control)` — Añade al final
> - `.removeAt(index)` — Elimina por posición
> - `.at(index)` — Accede a un elemento por posición
> - `.controls` — Array con todos los controles actuales

---

## Validators — Validaciones integradas

Angular incluye validadores listos para usar en `Validators`:

```typescript
import { Validators } from '@angular/forms';

fb.control('', [
  Validators.required,           // No puede estar vacío
  Validators.minLength(3),       // Mínimo 3 caracteres
  Validators.maxLength(50),      // Máximo 50 caracteres
  Validators.email,              // Debe ser un email válido
  Validators.pattern(/^[0-9]+$/) // Debe coincidir con el patrón regex
])

fb.control(0, [
  Validators.required,
  Validators.min(0),   // Valor mínimo
  Validators.max(100)  // Valor máximo
])
```

> 💡 **Validadores personalizados**: puedes crear tus propias funciones validadoras:
> ```typescript
> function noEspacios(control: AbstractControl): ValidationErrors | null {
>   return control.value?.includes(' ') ? { noEspacios: true } : null;
> }
> // Uso: fb.control('', [noEspacios])
> ```

---

## Signals + Reactive Forms — Reactividad combinada

Los Reactive Forms usan Observables (`valueChanges`), pero podemos integrarlos con Signals para aprovechar la reactividad de Angular 17+.

```typescript
import { signal, computed, effect } from '@angular/core';

export class AppComponent {
  fb = inject(NonNullableFormBuilder)

  form = this.fb.group({
    items: this.fb.array<CustomFormGroup>([])
  })

  // Signal que contiene los controles actuales del FormArray
  // Necesitamos un signal porque @for en el template necesita reactividad
  items = signal(this.form.controls.items.controls)

  // computed() calcula el total automáticamente cada vez que 'items' cambia
  totalValue = computed(() => {
    return this.items().reduce(
      (total, formGroup) => total + formGroup.controls.value.value,
      0
    )
  })

  constructor() {
    // effect() se ejecuta cuando el signal cambia.
    // Aquí: cuando valueChanges emite, actualizamos el signal 'items'
    // para que el template reaccione y @for se re-renderice
    effect(() => {
      this.form.controls.items.valueChanges.subscribe(() => {
        this.items.set([...this.form.controls.items.controls])
        /* Spread [...] crea un nuevo array para cambiar la referencia
           y así que Angular detecte el cambio en el signal */
      })
    })
  }

  addItem() {
    const itemForm = this.fb.group<ItemForm>({ /* ... */ })
    this.form.controls.items.push(itemForm)
    this.items.set([...this.form.controls.items.controls]) // Notifica el cambio
  }
}
```

```html
<!-- El template usa la signal 'items' en lugar del FormArray directamente -->
<button (click)="addItem()">Agregar Item</button>

@for (formGroup of items(); track formGroup.controls.id.value) {
  <app-form-child [formGroup]="formGroup" />
}

<h3>Total: {{ totalValue() }}</h3>
```

> 💡 `track formGroup.controls.id.value` le dice a Angular qué propiedad usar para identificar cada fila. Esto optimiza las actualizaciones del DOM: Angular solo re-renderiza las filas que realmente cambiaron.

---

## Pasar FormGroup entre componentes con input.required()

Para mantener el código organizado, cada fila del `FormArray` se puede extraer a un componente hijo. Se le pasa el `FormGroup` como `input.required()` (la forma moderna de `@Input()`):

```typescript
// form-child.component.ts
import { Component, input } from '@angular/core';
import { FormGroup, ReactiveFormsModule } from '@angular/forms';

@Component({
  selector: 'app-form-child',
  standalone: true,
  imports: [ReactiveFormsModule],
  templateUrl: './form-child.component.html',
})
export class FormChildComponent {
  // input.required<T>() = @Input() obligatorio con tipado fuerte
  // El padre DEBE pasar este valor o Angular lanzará un error
  formGroup = input.required<FormGroup<ItemForm>>()
}
```

```html
<!-- form-child.component.html -->
<!-- [formGroup]="formGroup()" vincula el FormGroup al <div> contenedor -->
<!-- formGroup() -> se llama como función porque input() devuelve un signal -->
<div [formGroup]="formGroup()">
  <app-custom-input [control]="formGroup().controls.name" formControlName="name" />
  <app-custom-input [control]="formGroup().controls.value" formControlName="value" />
</div>
```

```html
<!-- app.component.html — Uso desde el padre -->
@for (formGroup of items(); track formGroup.controls.id.value) {
  <app-form-child [formGroup]="formGroup" />
}
```

> 💡 `input.required<T>()` (Angular 17+) es la versión signal de `@Input({ required: true })`. La diferencia es que al leerlo en el template o en el código TypeScript se llama como función: `formGroup()` en vez de `formGroup`.

---

## ControlValueAccessor — Componentes de Input Personalizados

`ControlValueAccessor` (CVA) es la interfaz que permite que un componente Angular actúe como si fuera un `<input>` nativo para Angular Forms. Es el puente entre el DOM personalizado y el sistema de formularios de Angular.

> 💡 Sin CVA: si creas un `<app-custom-input>`, Angular no sabe cómo conectarlo al `FormControl`. Con CVA: Angular lo trata exactamente igual que un `<input>` nativo.

```typescript
import { Component, forwardRef, input } from '@angular/core';
import {
  ControlValueAccessor,
  FormControl,
  NG_VALUE_ACCESSOR,
  ReactiveFormsModule
} from '@angular/forms';

@Component({
  selector: 'app-custom-input',
  standalone: true,
  imports: [ReactiveFormsModule],
  templateUrl: './custom-input.component.html',
  providers: [
    {
      provide: NG_VALUE_ACCESSOR,
      /* forwardRef(() => CustomInputComponent) resuelve la referencia circular:
         el proveedor se registra ANTES de que la clase esté definida.
         Sin forwardRef, JavaScript lanzaría un error porque la clase aún no existe */
      useExisting: forwardRef(() => CustomInputComponent),
      /* multi: true permite que haya MÚLTIPLES ControlValueAccessors registrados
         para el mismo token (NG_VALUE_ACCESSOR). Siempre debe ser true aquí */
      multi: true
    }
  ]
})
export class CustomInputComponent implements ControlValueAccessor {
  // Recibe el FormControl del padre para poder acceder a su estado (valid, dirty, errors...)
  control = input.required<FormControl<any>>();

  // Callbacks que Angular registra automáticamente al conectar el CVA al formulario
  onTouched = () => {}                   // Se llama cuando el campo pierde el foco
  onChange  = (_value: any) => {}        // Se llama cuando el valor cambia

  // Angular llama a este método para escribir un valor en el componente
  // (p. ej. al hacer form.patchValue() o form.reset())
  writeValue(value: any): void {
    if (value !== this.control().value) {
      this.control().setValue(value, { emitEvent: false });
      /* emitEvent: false evita bucles infinitos: si setValue disparara valueChanges,
         y valueChanges llamara onChange, y onChange llamara writeValue... */
    }
  }

  // Angular llama a este método para registrar la función que debe ejecutarse cuando el valor cambia
  registerOnChange(fn: any): void {
    this.onChange = fn;
  }

  // Angular llama a este método para registrar la función que se ejecuta al hacer blur
  registerOnTouched(fn: any): void {
    this.onTouched = fn;
  }

  // Angular llama a este método cuando el FormControl se habilita o deshabilita
  setDisabledState(isDisabled: boolean): void {
    isDisabled ? this.control().disable() : this.control().enable()
  }
}
```

> 💡 **Los 4 métodos obligatorios de ControlValueAccessor:**
>
> | Método | Cuándo lo llama Angular |
> |---|---|
> | `writeValue(value)` | Cuando el formulario escribe un valor en el input |
> | `registerOnChange(fn)` | Para registrar el callback a llamar cuando el usuario cambia el valor |
> | `registerOnTouched(fn)` | Para registrar el callback a llamar cuando el input pierde el foco |
> | `setDisabledState(isDisabled)` | Cuando el FormControl se habilita o deshabilita |

---

## NG_VALUE_ACCESSOR y forwardRef

`NG_VALUE_ACCESSOR` es el token de inyección de dependencias que Angular usa para encontrar el `ControlValueAccessor` de un elemento del formulario.

```typescript
providers: [
  {
    provide: NG_VALUE_ACCESSOR,
    useExisting: forwardRef(() => CustomInputComponent),
    multi: true
  }
]

/* Desglose:
   - provide: NG_VALUE_ACCESSOR   → Token que Angular busca para los CVA
   - useExisting: forwardRef(...) → "Usa esta clase como CVA"
   - forwardRef(() => Clase)      → Evita error de "Clase no definida aún"
   - multi: true                  → "Añade este CVA a la lista, no reemplaces los demás"
*/
```

---

## Template del Custom Input con validación

```html
<!-- custom-input.component.html -->

<!-- @let guarda el valor del signal en una variable local para no llamarlo múltiples veces -->
@let localControl = control();

<input [formControl]="localControl" (blur)="onTouched()" />
<!-- [formControl] conecta el FormControl directamente al <input> -->
<!-- (blur) llama a onTouched() para marcar el control como "tocado" -->

<!-- Mostrar errores solo si el campo es inválido Y (está sucio O fue tocado) -->
@if ((localControl.invalid && localControl.dirty) || localControl.touched) {
  <div class="error-messages">
    @if (localControl.errors?.['required']) {
      <span>Este campo es obligatorio</span>
    }
    @if (localControl.errors?.['min']) {
      <span>El valor mínimo es {{ localControl.errors?.['min'].min }}</span>
    }
    @if (localControl.errors?.['email']) {
      <span>Introduce un email válido</span>
    }
  </div>
}
```

> 💡 **¿Cuándo mostrar errores?** La convención es:
> - `dirty` = el usuario ya escribió algo (campo modificado desde su valor inicial)
> - `touched` = el usuario hizo blur (salió del campo sin necesariamente modificarlo)
> - Mostrar errores cuando `(invalid && dirty) || touched` cubre ambos casos y evita mostrar errores antes de que el usuario interactúe con el campo.

---

## Estados de un FormControl

```typescript
const ctrl = new FormControl('');

ctrl.pristine  // true  → No ha sido modificado por el usuario
ctrl.dirty     // false → Ha sido modificado por el usuario
ctrl.untouched // true  → El usuario no ha hecho blur
ctrl.touched   // false → El usuario ha hecho blur (salido del campo)
ctrl.valid     // true/false dependiendo de los validadores
ctrl.invalid   // lo contrario de valid
ctrl.disabled  // true si el control está deshabilitado
ctrl.enabled   // lo contrario de disabled
ctrl.errors    // null si es válido, o un objeto { required: true, min: {...} } si no
```

---

## Flujo completo de la arquitectura del proyecto

```
AppComponent (formulario raíz)
│
│  form: FormGroup<{items: FormArray<CustomFormGroup>}>
│  items: signal([...controls])
│  totalValue: computed(() => suma de values)
│
├── [Botón "Agregar Item"] → addItem() → form.controls.items.push(nuevoGrupo)
│                                      → items.set([...controls]) (notifica signal)
│
└── @for (formGroup of items())
      └── <app-form-child [formGroup]="formGroup">
            │
            │  formGroup = input.required<FormGroup<ItemForm>>()
            │  [formGroup]="formGroup()" → vincula el grupo al <div>
            │
            ├── <app-custom-input formControlName="name">
            │     └── ControlValueAccessor → puente con FormControl<string>
            │         Template: <input [formControl]="..."> + errores
            │
            └── <app-custom-input formControlName="value">
                  └── ControlValueAccessor → puente con FormControl<number>
```

---

## Resumen de conceptos clave

| Concepto | Qué es | Para qué sirve |
|---|---|---|
| **`ReactiveFormsModule`** | Módulo de Angular | Habilita las directivas de Reactive Forms en el template |
| **`FormControl<T>`** | Control individual tipado | Representa un campo del formulario con su valor y estado |
| **`FormGroup<T>`** | Agrupador de controles | Agrupa controles relacionados bajo un objeto tipado |
| **`FormArray<T>`** | Array dinámico de controles | Permite añadir/quitar filas de formulario en tiempo de ejecución |
| **`NonNullableFormBuilder`** | Servicio de creación | Crea controles que nunca son `null` al hacer reset |
| **`Validators`** | Funciones de validación | `required`, `min`, `max`, `email`, `minLength`... |
| **`interface ItemForm`** | Interfaz TypeScript | Da tipado estricto a los controles de un `FormGroup` |
| **`ControlValueAccessor`** | Interfaz de Angular | Permite que componentes custom actúen como inputs de formulario |
| **`NG_VALUE_ACCESSOR`** | Token de inyección | Angular lo usa para encontrar el CVA de un elemento |
| **`forwardRef()`** | Utilidad de Angular | Resuelve referencias circulares al registrar providers |
| **`input.required<T>()`** | Input signal moderno | `@Input()` obligatorio con tipado fuerte (Angular 17+) |
| **`signal` + `valueChanges`** | Combinación reactividad | Integra Observables de formularios con el sistema de Signals |
| **`computed()`** | Signal derivado | Calcula valores derivados del estado del formulario automáticamente |
