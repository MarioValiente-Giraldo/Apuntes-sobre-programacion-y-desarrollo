# Angular — Guía Rápida Definitiva

> 💡 Resumen de los 6 bloques del curso: Control Flow, Directivas, Services/SSOT, Reactive Forms, Interceptors y Testing. Cubre Angular 17/18 con standalone components, Signals y la API funcional moderna.

---

## 1. Control Flow Syntax (Templates)

Reemplaza `*ngIf`, `*ngFor`, `[ngSwitch]`. No requiere importar módulos.

```html
<!-- Condicional -->
@if (isAdmin) {
  <p>Panel admin</p>
} @else if (isUser) {
  <p>Panel usuario</p>
} @else {
  <p>Acceso denegado</p>
}

<!-- Bucle — track es obligatorio -->
@for (user of users; track user.id) {
  <p>{{ user.name }}</p>
} @empty {
  <p>Sin resultados</p>
}

<!-- Variables disponibles en @for -->
@for (item of items; track item.id; let i = $index, isFirst = $first, isLast = $last) { }

<!-- Switch -->
@switch (role) {
  @case ('admin') { <p>Admin</p> }
  @case ('user')  { <p>User</p>  }
  @default        { <p>Otro</p>  }
}
```

### @defer — Carga diferida

```html
<!-- Ciclo: placeholder → loading → contenido ✅ / @error ❌ -->
@defer (on viewport) {
  <app-heavy-chart />
} @loading (after 300ms; minimum 500ms) {
  <p>Cargando...</p>
} @placeholder {
  <div style="height:400px">Haz scroll...</div>
} @error {
  <p>Error al cargar</p>
}
```

| Trigger | Cuándo se activa |
|---|---|
| `when condicion` | Variable booleana |
| `on idle` | Navegador inactivo |
| `on viewport` | Elemento en pantalla |
| `on interaction` | Primer clic del usuario |
| `on hover` | Ratón sobre el placeholder |
| `on timer(Xms)` | Tras X milisegundos |
| `prefetch on idle` | Precarga JS en background |

---

## 2. Directivas

### Estructurales (modifican el DOM)

```typescript
@Directive({ standalone: true, selector: '[appShowOnSize]' })
export class ShowOnSizeDirective implements OnInit {
  @Input() appShowOnSize: 'small' | 'large';
  constructor(
    private templateRef: TemplateRef<any>,   // El contenido del template
    private viewContainer: ViewContainerRef  // Donde se inserta/elimina
  ) {}
  ngOnInit() {
    if (this.shouldShow()) {
      this.viewContainer.createEmbeddedView(this.templateRef); // Añade al DOM
    } else {
      this.viewContainer.clear(); // Elimina del DOM
    }
  }
}
```

### De atributo (modifican estilos/comportamiento)

```typescript
@Directive({ standalone: true, selector: '[appHighlight]' })
export class HighlightDirective {
  constructor(private el: ElementRef, private renderer: Renderer2) {}

  @HostListener('mouseenter') onEnter() {
    this.renderer.setStyle(this.el.nativeElement, 'background-color', 'yellow');
  }
  @HostListener('mouseleave') onLeave() {
    this.renderer.removeStyle(this.el.nativeElement, 'background-color');
  }
}
```

```html
<p appHighlight>Hover me</p>
<div *appShowOnSize="'small'">Solo en móvil</div>
```

> 💡 `Renderer2` es preferible a manipular `nativeElement` directamente: es compatible con SSR y otros entornos no-browser.

### Pipes

```html
{{ nombre | uppercase }}
{{ precio | currency:'EUR' }}
{{ fecha | date:'dd/MM/yyyy' }}
{{ texto | lowercase | capitalize }}  <!-- Se pueden encadenar -->
```

```typescript
@Pipe({ standalone: true, name: 'capitalize' })
export class CapitalizePipe implements PipeTransform {
  transform(value: string): string {
    return value.charAt(0).toUpperCase() + value.slice(1).toLowerCase();
  }
}
```

### Comunicación entre componentes

```typescript
// Padre → Hijo
@Input() nombre: string = '';
// <app-hijo [nombre]="'María'">

// Hijo → Padre
@Output() evento = new EventEmitter<string>();
this.evento.emit('datos');
// <app-hijo (evento)="handler($event)">

// Moderno (Angular 17+)
nombre = input.required<string>(); // Signal, se lee como nombre()
```

### Lifecycle Hooks

| Hook | Cuándo se ejecuta |
|---|---|
| `ngOnChanges` | Cuando cambia un `@Input()` |
| `ngOnInit` | Una vez tras la primera renderización |
| `ngDoCheck` | En cada ciclo de detección de cambios |
| `ngOnDestroy` | Cuando el componente se destruye |

> 💡 Orden: `constructor → ngOnChanges → ngOnInit → ngDoCheck → ngOnDestroy`

### HTTP Client y Routing (esenciales)

```typescript
// app.config.ts
provideHttpClient(withFetch(), withInterceptors([...]))
provideRouter(routes)

// Rutas
{ path: '', component: HomeComponent },
{ path: 'users/:id', component: UserDetailComponent },
{ path: 'dashboard', loadComponent: () => import('./dashboard.component').then(c => c.DashboardComponent) },
{ path: '**', component: NotFoundComponent }
```

```html
<a routerLink="/">Inicio</a>
<a [routerLink]="['/users', userId]">Usuario</a>
<router-outlet></router-outlet>
```

---

## 3. Services y Arquitectura SSOT

### Service con Signal como Single Source of Truth

```typescript
@Injectable({ providedIn: 'root' }) // Singleton: una instancia para toda la app
export class CharacterService {

  // El estado vive aquí — nadie más almacena datos
  state = signal({ characters: new Map<number, Character>() })

  getFormattedCharacters() {
    return Array.from(this.state().characters.values()) // Map → array para el template
  }

  updateCharacter(character: Character) {
    of(character).subscribe(result => { // of() simula HTTP; en prod: this.http.put()
      this.state.update(state => {      // update() = set() pero basado en estado actual
        state.characters.set(result.id, result)
        return { characters: state.characters }
      })
    })
  }

  deleteCharacter(id: number) {
    this.state.update(state => {
      state.characters.delete(id)
      return { characters: state.characters }
    })
  }
}
```

### Componente con computed() + OnPush

```typescript
@Component({
  changeDetection: ChangeDetectionStrategy.OnPush // Solo actualiza cuando el signal cambia
})
export class AppComponent {
  service = inject(CharacterService)

  // computed() = signal derivado; se recalcula cuando service.state cambia
  characters = computed(() => this.service.getFormattedCharacters())
}
```

```html
@let list = characters();  <!-- @let: evita llamar al signal múltiples veces -->
@for (char of list; track char.id) {
  <img [src]="char.image" />
  <h2>{{ char.name }}</h2>
}
```

### Signals — API completa

```typescript
count = signal(0);              // Crear
count();                        // Leer
count.set(5);                   // Escribir (valor directo)
count.update(v => v + 1);       // Escribir (basado en anterior)
double = computed(() => count() * 2); // Derivado (solo lectura)
effect(() => console.log(count())); // Efecto secundario reactivo

// Observable → Signal (gestiona la suscripción automáticamente)
characters = toSignal(this.http.get<Character[]>(url), { initialValue: [] })
```

### Tipos de providers

```typescript
providers: [
  { provide: MockService, useClass: RealService },    // Sustituir clase
  { provide: 'CONFIG',    useValue: { url: '...' } }, // Valor constante
  { provide: DataService, useFactory: myFactory, deps: [] }, // Con lógica de creación
  { provide: AliasService, useExisting: BaseService } // Alias (misma instancia)
]
```

### Promise vs Observable

| | Promise | Observable |
|---|---|---|
| Emite valores | Una vez | Múltiples veces |
| Cancelable | No | Sí (`.unsubscribe()`) |
| Lazy | No | Sí |
| Operadores | `.then/.catch` | `map`, `filter`, `catchError`... |

### Patrones de arquitectura

```typescript
// Barrel exports (index.ts)
export * from './character.model'; // import { Character } from '../models'

// Adapter — transformación de datos de API
export const characterAdapter = (chars: Character[]) =>
  chars.map(c => ({ ...c, name: c.name.toUpperCase() }))
```

---

## 4. Reactive Forms

### Jerarquía de clases

```typescript
// FormControl — un campo individual
const email = new FormControl<string>('', [Validators.required, Validators.email]);
email.value;    // Leer valor
email.setValue('a@b.com');
email.valid / email.invalid / email.dirty / email.touched

// FormGroup — agrupa controles
const loginForm = new FormGroup({
  email:    new FormControl('', [Validators.required, Validators.email]),
  password: new FormControl('', [Validators.required, Validators.minLength(6)])
});
loginForm.value;                   // { email: '', password: '' }
loginForm.controls.email.setValue(...);
loginForm.valid;                   // true solo si todos los controles son válidos

// FormArray — array dinámico
form.controls.items.push(nuevoGrupo);
form.controls.items.removeAt(index);
form.controls.items.at(index);
```

### NonNullableFormBuilder (recomendado)

```typescript
fb = inject(NonNullableFormBuilder) // Los controles NUNCA son null al hacer reset()

form = this.fb.group<ItemForm>({
  id:    this.fb.control(0),
  name:  this.fb.control('', [Validators.required]),
  value: this.fb.control(0,  [Validators.required, Validators.min(0)])
})
```

### Validadores

```typescript
Validators.required
Validators.minLength(3) / Validators.maxLength(50)
Validators.min(0) / Validators.max(100)
Validators.email
Validators.pattern(/^[0-9]+$/)

// Validador personalizado
function noEspacios(ctrl: AbstractControl): ValidationErrors | null {
  return ctrl.value?.includes(' ') ? { noEspacios: true } : null;
}
```

### Template con validación

```html
<form [formGroup]="form" (ngSubmit)="onSubmit()">
  <input formControlName="email" />
  @if (form.controls.email.invalid && form.controls.email.touched) {
    @if (form.controls.email.errors?.['required']) {
      <span>Campo obligatorio</span>
    }
    @if (form.controls.email.errors?.['email']) {
      <span>Email inválido</span>
    }
  }
  <button [disabled]="form.invalid">Enviar</button>
</form>
```

### Signals + FormArray (combinado)

```typescript
items = signal(this.form.controls.items.controls) // Signal del array de controles
totalValue = computed(() =>
  this.items().reduce((sum, fg) => sum + fg.controls.value.value, 0)
)
constructor() {
  effect(() => {
    this.form.controls.items.valueChanges.subscribe(() => {
      this.items.set([...this.form.controls.items.controls]) // Spread cambia referencia → detecta cambio
    })
  })
}
```

### ControlValueAccessor — Input personalizado

```typescript
@Component({
  providers: [{
    provide: NG_VALUE_ACCESSOR,
    useExisting: forwardRef(() => CustomInputComponent), // forwardRef evita referencia circular
    multi: true                                          // Permite múltiples CVA en el pipeline
  }]
})
export class CustomInputComponent implements ControlValueAccessor {
  control = input.required<FormControl<any>>();
  onTouched = () => {};
  onChange  = (_: any) => {};

  writeValue(value: any)           { this.control().setValue(value, { emitEvent: false }) }
  registerOnChange(fn: any)        { this.onChange = fn }
  registerOnTouched(fn: any)       { this.onTouched = fn }
  setDisabledState(disabled: boolean) { disabled ? this.control().disable() : this.control().enable() }
}
```

| Estado | Significado |
|---|---|
| `pristine` | No modificado por el usuario |
| `dirty` | El usuario escribió algo |
| `untouched` | El usuario no hizo blur |
| `touched` | El usuario salió del campo |
| `valid/invalid` | Según los validadores |
| `errors` | `null` si válido, objeto si no |

---

## 5. HTTP Interceptors

### Interceptor funcional (Angular 17+)

```typescript
// src/app/interceptors/auth.interceptor.ts
export const authInterceptor: HttpInterceptorFn = (req, next) => {
  const platformId = inject(PLATFORM_ID)

  // SSR: localStorage no existe en Node.js — pasar sin modificar
  if (isPlatformServer(platformId)) return next(req)

  const token = localStorage.getItem('token')

  // Las peticiones son inmutables — siempre clonar
  let headers = req.headers.set('Content-Type', 'application/json')
  if (token) headers = headers.set('Authorization', `Bearer ${token}`)

  const authReq = req.clone({ headers })

  return next(authReq).pipe(
    catchError((error: HttpErrorResponse) => {
      if (error.status === 401 || error.status === 403) {
        // Token expirado → renovar y reintentar automáticamente
        return inject(AuthService).refreshToken().pipe(
          switchMap(newToken => {
            localStorage.setItem('token', newToken)
            const retryReq = req.clone({
              headers: req.headers.set('Authorization', `Bearer ${newToken}`)
            })
            return next(retryReq) // Reintento transparente para el servicio
          })
        )
      }
      return throwError(() => error) // Propagar errores no gestionados
    })
  )
}
```

### Registro en app.config.ts

```typescript
export const appConfig: ApplicationConfig = {
  providers: [
    provideHttpClient(
      withFetch(),                         // Fetch API en lugar de XHR (compatible SSR)
      withInterceptors([authInterceptor])  // Orden del array = orden de ejecución
    ),
    provideRouter(routes)
  ]
}
```

### Operadores RxJS clave

| Operador | Modifica valor | Uso típico |
|---|---|---|
| `map` | Sí | Transformar respuesta: `{ data: [...] }` → `[...]` |
| `tap` | No | Logging, guardar en localStorage |
| `catchError` | Depende | Capturar y gestionar errores HTTP |
| `switchMap` | Sí (cambia Observable) | Encadenar llamadas dependientes (token refresh) |
| `throwError` | — | Propagar error como Observable |

```
Flujo de token refresh:
petición → interceptor añade token → 401 → catchError → refreshToken()
→ switchMap recibe nuevo token → req.clone con nuevo token → next(retryReq)
→ el servicio recibe la respuesta como si nada hubiera pasado
```

---

## 6. Testing (Jest + Playwright)

### Pirámide de testing

```
     /E2E\           Playwright: app real en navegador, flujos críticos
    /------\
   /Integra \        TestBed + HttpTestingController: componente + servicio
  /----------\
 / Unitario   \      Jest puro: lógica de negocio aislada, servicios, utils
/--------------\
```

### Estructura base de un test

```typescript
describe('AuthService', () => {
  let service: AuthService;

  beforeEach(() => {           // Se ejecuta ANTES de cada 'it' — estado limpio
    TestBed.configureTestingModule({});
    service = TestBed.inject(AuthService);
  });

  afterEach(() => {            // Limpieza después de cada test
    httpTesting.verify();      // Verifica peticiones HTTP sin responder
  });

  it('should be created', () => {
    // GIVEN — contexto previo (lo hace beforeEach)
    // WHEN  — acción (en este caso, solo verificamos existencia)
    // THEN  — verificación del resultado
    expect(service).toBeTruthy();
  });
});
```

### TestBed — acceder al componente

```typescript
beforeEach(async () => {
  await TestBed.configureTestingModule({
    imports: [AppComponent],    // Componentes standalone se importan directamente
    providers: [AuthService]
  }).compileComponents();       // Compila templates HTML + CSS
});

it('renders correctly', () => {
  const fixture = TestBed.createComponent(AppComponent);
  const app     = fixture.componentInstance; // La CLASE TypeScript
  const dom     = fixture.nativeElement as HTMLElement; // El DOM

  fixture.detectChanges(); // CRÍTICO: sin esto el template no se renderiza

  expect(app.title).toEqual('mi-app');
  expect(dom.querySelector('h1')?.textContent).toContain('Hello');
});
```

### Testing de HTTP con HttpTestingController

```typescript
beforeEach(async () => {
  await TestBed.configureTestingModule({
    providers: [
      provideHttpClient(withFetch()),
      provideHttpClientTesting(), // Sustituye HttpClient real por versión de test
      AuthService
    ]
  });
  service    = TestBed.inject(AuthService);
  httpTesting = TestBed.inject(HttpTestingController);
});

it('should login', () => {
  const mockResponse = { token: 'fake-jwt' };

  // 1. Disparar la llamada
  service.login('a@b.com', '123').subscribe({
    next: (res) => expect(res.token).toBe('fake-jwt') // 4. Verificar respuesta
  });

  // 2. Interceptar la petición (no salió a la red)
  const req = httpTesting.expectOne('/api/login');

  // 3. Verificar detalles de la petición
  expect(req.request.method).toBe('POST');
  expect(req.request.body).toEqual({ email: 'a@b.com', password: '123' });

  // 4. Responder manualmente → dispara el subscribe()
  req.flush(mockResponse);
});

it('should handle 401', () => {
  service.login('a@b.com', 'bad').subscribe({
    next: () => fail('debería haber fallado'),
    error: (err) => expect(err.status).toBe(401)
  });
  httpTesting.expectOne('/api/login').flush('Unauthorized', { status: 401, statusText: 'Unauthorized' });
});
```

### Tests E2E con Playwright

```typescript
// e2e/login.e2e.spec.ts
import { test, expect } from '@playwright/test';

test('login flow', async ({ page }) => {
  await page.goto('/login');
  await page.fill('[data-testid="email"]', 'user@example.com');
  await page.fill('[data-testid="password"]', 'password123');
  await page.click('[data-testid="submit"]');
  await expect(page).toHaveURL('/dashboard');
  await expect(page.locator('h1')).toHaveText('Bienvenido');
});
```

```typescript
// playwright.config.ts — configuración clave
export default defineConfig({
  testDir: './e2e',
  use: { baseURL: 'http://localhost:4200', trace: 'on-first-retry' },
  projects: [
    { name: 'chromium', use: { ...devices['Desktop Chrome'] } },
    { name: 'firefox',  use: { ...devices['Desktop Firefox'] } },
  ]
})
```

```javascript
// jest.config.js — excluir archivos de Playwright de Jest
module.exports = {
  preset: 'jest-preset-angular',
  setupFilesAfterEnv: ['<rootDir>/setup-jest.ts'],
  testPathIgnorePatterns: [
    '<rootDir>/node_modules/',
    '.*\\.e2e\\.spec\\.ts$' // Playwright los toma del /e2e, Jest los ignora
  ],
  globalSetup: 'jest-preset-angular/global-setup'
}
```

---

## Resumen general — todo Angular en una tabla

| Tema | Concepto clave | Para qué sirve |
|---|---|---|
| **Control Flow** | `@if / @for / @switch` | Lógica en templates sin importar módulos |
| **Control Flow** | `@defer + triggers` | Lazy loading de componentes en el template |
| **Control Flow** | `track` en `@for` | Identificar elementos únicos para optimizar el DOM |
| **Directivas** | `@Directive` estructural | Añadir/eliminar elementos del DOM dinámicamente |
| **Directivas** | `@Directive` de atributo | Modificar estilos/comportamiento sin tocar el DOM |
| **Directivas** | `@HostListener` | Escuchar eventos del elemento host declarativamente |
| **Directivas** | `Renderer2` | Manipular el DOM de forma segura (compatible SSR) |
| **Pipes** | `@Pipe` | Transformar datos en el template sin modificar el original |
| **Comunicación** | `@Input() / @Output()` | Pasar datos entre componentes padre-hijo |
| **Comunicación** | `input.required<T>()` | `@Input` obligatorio con tipado Signal (Angular 17+) |
| **Services** | `@Injectable({ providedIn: 'root' })` | Singleton compartido en toda la app |
| **SSOT** | `signal()` en servicio | Estado centralizado reactivo (Single Source of Truth) |
| **SSOT** | `computed()` en componente | Derivar datos del estado del servicio automáticamente |
| **SSOT** | `OnPush` | Solo re-renderizar cuando el Signal cambia (rendimiento) |
| **SSOT** | `toSignal()` | Convertir Observable en Signal (gestiona suscripción) |
| **SSOT** | `@let` | Variable local en el template (evita lecturas duplicadas) |
| **SSOT** | Barrel exports | Imports limpios agrupados en `index.ts` |
| **SSOT** | Adapter | Función pura que transforma datos de API al modelo interno |
| **Providers** | `useClass` | Sustituir una implementación por otra |
| **Providers** | `useValue` | Inyectar constantes o configuración |
| **Providers** | `useFactory` | Crear servicio con lógica de inicialización |
| **Providers** | `useExisting` | Alias — reutiliza una instancia ya existente |
| **Forms** | `FormControl<T>` | Campo individual con tipado fuerte |
| **Forms** | `FormGroup<T>` | Agrupa controles bajo un objeto tipado |
| **Forms** | `FormArray<T>` | Array dinámico (añadir/quitar filas en runtime) |
| **Forms** | `NonNullableFormBuilder` | Controles que nunca son `null` al hacer `reset()` |
| **Forms** | `ControlValueAccessor` | Componente custom que actúa como `<input>` nativo |
| **Forms** | `NG_VALUE_ACCESSOR + forwardRef` | Registrar el CVA en el pipeline de Angular Forms |
| **Interceptors** | `HttpInterceptorFn` | Función que intercepta todas las peticiones HTTP |
| **Interceptors** | `req.clone()` | Modificar headers sin mutar la petición original |
| **Interceptors** | `withInterceptors([])` | Registrar interceptores en `app.config.ts` |
| **Interceptors** | `catchError + switchMap` | Token refresh: renovar sesión automáticamente |
| **Interceptors** | `PLATFORM_ID + isPlatformServer` | Detectar SSR para evitar usar `localStorage` en Node |
| **Testing** | `TestBed` | Módulo Angular simulado para el entorno de test |
| **Testing** | `fixture.detectChanges()` | Aplicar cambios de estado al DOM (Change Detection) |
| **Testing** | `provideHttpClientTesting()` | Sustituir HttpClient por versión que no sale a la red |
| **Testing** | `HttpTestingController` | Interceptar, verificar y responder peticiones HTTP |
| **Testing** | `req.flush(data)` | Responder manualmente una petición HTTP en tests |
| **Testing** | `httpTesting.verify()` | Detectar peticiones sin responder tras cada test |
| **Testing** | Playwright `page.goto/fill/click` | Simular acciones de usuario en navegador real |
| **Testing** | Given / When / Then | Patrón para estructurar tests legibles |
