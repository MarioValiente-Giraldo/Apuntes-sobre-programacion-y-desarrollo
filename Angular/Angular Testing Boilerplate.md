# Angular: Testing Boilerplate (Jest + Playwright)

> 💡 El **testing en Angular** se divide en dos grandes capas: **tests unitarios** (con Jest, prueban lógica aislada de componentes y servicios) y **tests E2E** (con Playwright, prueban la aplicación completa en un navegador real). Este boilerplate configura las dos capas desde cero.

---

## Pirámide de testing — ¿Qué testear y con qué?

```
          /\
         /E2E\         ← Playwright: prueba la app real en navegador
        /------\
       / Integr \      ← TestBed + HttpTestingController: componente + servicio juntos
      /----------\
     /   Unitario \    ← Jest puro: lógica de negocio aislada (servicios, utils)
    /--------------\
```

| Tipo | Herramienta | Qué prueba | Velocidad |
|---|---|---|---|
| **Unitario** | Jest | Una clase/función aislada | Muy rápido |
| **Integración** | TestBed + Jest | Componente con sus dependencias | Rápido |
| **E2E** | Playwright | La app completa en navegador real | Lento |

> 💡 No todo necesita test E2E. La mayoría del código se cubre con tests unitarios e integración (Jest). Playwright solo para flujos críticos del usuario: login, checkout, formularios principales.

---

## Configuración: Stack de testing

### `package.json` — Dependencias

```json
// devDependencies relevantes para testing
{
  "@jest/globals": "^30.2.0",              // Tipado de Jest
  "@playwright/test": "1.58.2",            // Test E2E

  "@testing-library/angular": "^17.4.0",  // Utilidades de Testing Library para Angular
  "@testing-library/dom": "^10.4.1",      // Queries de DOM (getByRole, getByText...)
  "@testing-library/jest-dom": "^6.9.1",  // Matchers extra: toBeVisible, toHaveValue...
  "@testing-library/user-event": "^14.6.1", // Simular eventos de usuario realistas

  "@types/jest": "^30.0.0",               // Tipos TypeScript de Jest
  "jest": "^29.7.0",                       // Test runner
  "jest-preset-angular": "^14.6.2"        // Preset que conecta Jest con Angular
}
```

```json
// scripts
{
  "test": "jest",                  // Ejecuta todos los tests unitarios/integración
  "test:coverage": "jest --coverage", // Tests + reporte de cobertura
  "e2e": "ng e2e"                  // Tests E2E con Playwright
}
```

> 💡 `jest-preset-angular` es el puente clave: configura Jest para que entienda los decoradores de Angular (`@Component`, `@Injectable`...), el compilador de templates, y el sistema de módulos.

---

## Configuración: `jest.config.js`

```javascript
// jest.config.js — en la raíz del proyecto
module.exports = {
  preset: 'jest-preset-angular',  // Configura Jest para compilar Angular (ts, templates, decoradores)

  setupFilesAfterEnv: ['<rootDir>/setup-jest.ts'],  // Archivo que se ejecuta ANTES de cada test suite

  testPathIgnorePatterns: [
    '<rootDir>/node_modules/',       // Ignorar dependencias
    '.*\\.e2e\\.spec\\.ts$',         // Ignorar tests E2E (los corre Playwright, no Jest)
    '.*\\.functional\\.spec\\.ts$'  // Ignorar tests funcionales si los hay
  ],

  globalSetup: 'jest-preset-angular/global-setup'  // Setup global del entorno Angular+Jest
}
```

> 💡 `testPathIgnorePatterns` es crítico: evita que Jest intente ejecutar los archivos `.e2e.spec.ts` (esos son de Playwright). Sin esto, Jest fallaría al encontrar APIs de Playwright que no existen en su contexto.

---

## Configuración: `setup-jest.ts`

```typescript
// setup-jest.ts — se ejecuta una vez antes de cada test suite
import { setupZoneTestEnv } from 'jest-preset-angular/setup-env/zone'
// setupZoneTestEnv() inicializa Zone.js en el entorno de test
// Zone.js es lo que permite que Angular detecte cambios (Change Detection)
// Sin esto, los cambios de estado no se reflejarían en el DOM durante los tests
setupZoneTestEnv

import "@testing-library/jest-dom"
// Añade matchers extra a expect():
// toBeInTheDocument(), toBeVisible(), toHaveValue(), toHaveTextContent()...
```

> 💡 `Zone.js` es necesario en tests porque Angular 18 sin zoneless sigue usando Zone para la detección de cambios en el entorno de pruebas. `jest-preset-angular/setup-env/zone` lo inicializa correctamente en Jest (que no es un navegador real).

---

## Configuración: `tsconfig.spec.json`

```json
// tsconfig.spec.json — TypeScript config solo para los tests
{
  "extends": "./tsconfig.json",  // Hereda la config base del proyecto
  "compilerOptions": {
    "outDir": "./out-tsc/spec",
    "types": [
      "jest",                       // Tipos globales de Jest: describe, it, expect, beforeEach...
      "@testing-library/jest-dom"  // Tipos de los matchers extra (toBeVisible, etc.)
    ]
  },
  "include": [
    "src/**/*.spec.ts",   // Solo compila archivos de test
    "src/**/*.d.ts"       // Y las declaraciones de tipos
  ]
}
```

> 💡 Separar `tsconfig.spec.json` del `tsconfig.app.json` permite tener los tipos de Jest disponibles solo en los tests, sin contaminar el código de producción con `describe`, `it`, `expect` como globales.

---

## Estructura base de un test — `describe` / `it` / `beforeEach`

```typescript
// src/app/services/auth.service.spec.ts
import { TestBed } from '@angular/core/testing';
import { AuthService } from './auth.service';

// describe: agrupa tests relacionados bajo un nombre descriptivo
describe('AuthService', () => {
  let service: AuthService;  // Variable compartida entre todos los tests del bloque

  // beforeEach: se ejecuta ANTES de cada 'it', garantiza estado limpio por test
  beforeEach(() => {
    TestBed.configureTestingModule({});      // Configura el módulo de testing Angular
    service = TestBed.inject(AuthService);  // Obtiene una instancia del servicio del DI
  });

  // it: un test individual. Describe qué DEBE ocurrir
  it('should be created', () => {
    // Patrón Given / When / Then:
    // given: el servicio ya fue creado en beforeEach
    // when: (implícito — no hay acción, solo verificamos existencia)
    // then: el servicio debe existir
    expect(service).toBeTruthy();
  });
});

// Existen dos formas de trabajar los tests:
// 1 - Cada test es independiente (recomendado): beforeEach resetea el estado
// 2 - Cada test depende del anterior: orden importa, más frágil, menos mantenible
```

> 💡 La filosofía de **tests independientes** (opción 1) es la recomendada: cada `it` debe poder ejecutarse solo, en cualquier orden, sin depender del estado dejado por otro test. El `beforeEach` garantiza esto reseteando el entorno cada vez.

---

## `TestBed` — El contenedor de testing de Angular

`TestBed` simula el módulo Angular en el que vive el componente o servicio bajo test. Es equivalente a `AppModule` pero solo para el contexto del test.

```typescript
import { TestBed } from '@angular/core/testing';
import { AppComponent } from './app.component';

describe('AppComponent', () => {
  beforeEach(async () => {
    // configureTestingModule: declara qué imports/providers necesita el test
    await TestBed.configureTestingModule({
      imports: [AppComponent],  // Para componentes standalone, se importan directamente
      // providers: []           // Para servicios, interceptores, etc.
    }).compileComponents();     // Compila los templates HTML + CSS del componente
  });

  it('should create the app', () => {
    // createComponent: crea una instancia del componente en el DOM de test
    const fixture = TestBed.createComponent(AppComponent);

    // componentInstance: accede a la CLASE del componente (la instancia TypeScript)
    const app = fixture.componentInstance;

    expect(app).toBeTruthy();  // El componente existe
  });

  it(`should have the 'angular-testing-boilerplate' title`, () => {
    const fixture = TestBed.createComponent(AppComponent);
    const app = fixture.componentInstance;

    // Accedemos directamente a las propiedades de la clase del componente
    expect(app.title).toEqual('angular-testing-boilerplate');
  });

  it('should render title', () => {
    const fixture = TestBed.createComponent(AppComponent);

    // detectChanges(): dispara el ciclo de Change Detection
    // IMPORTANTE: sin esto, el template no se renderiza y el DOM está vacío
    fixture.detectChanges();

    // nativeElement: el elemento DOM real del componente (como document.querySelector)
    const compiled = fixture.nativeElement as HTMLElement;

    // Verificamos que el HTML contiene el texto esperado
    expect(compiled.querySelector('h1')?.textContent).toContain('Hello, angular-testing-boilerplate');
  });
});
```

> 💡 `fixture.detectChanges()` es equivalente a "aplicar los cambios al DOM". Si no se llama, el template no se procesa y cualquier query al DOM (`querySelector`) devolverá vacío. Siempre llamarlo antes de verificar el HTML renderizado.

---

## `ComponentFixture` — Las tres formas de acceder al componente

```typescript
const fixture = TestBed.createComponent(MiComponente);

// 1. componentInstance → la CLASE TypeScript
// Acceder a propiedades, métodos, signals, etc.
const instancia = fixture.componentInstance;
instancia.titulo = 'nuevo titulo';
instancia.cargarDatos();

// 2. nativeElement → el ELEMENTO DOM (HTMLElement)
// Acceder al HTML renderizado, queries CSS
const dom = fixture.nativeElement as HTMLElement;
dom.querySelector('button')?.click();
dom.querySelector('h1')?.textContent;

// 3. detectChanges() → ACTUALIZAR el DOM
// Necesario después de cualquier cambio en la instancia
// para que el template refleje el nuevo estado
fixture.detectChanges();
```

| Accessor | Qué devuelve | Uso |
|---|---|---|
| `fixture.componentInstance` | La clase del componente | Acceder a propiedades y métodos |
| `fixture.nativeElement` | El HTMLElement del componente | Queries al DOM renderizado |
| `fixture.detectChanges()` | `void` | Sincronizar estado → DOM |

---

## Testing de HTTP — `HttpTestingController`

Para testear servicios que hacen llamadas HTTP, Angular provee `HttpTestingController`: intercepta las peticiones HTTP antes de que salgan a la red y permite controlar manualmente qué respuesta devuelven.

```typescript
// src/app/Login/login/login.component.spec.ts
import { TestBed } from '@angular/core/testing';
import { provideHttpClient, withFetch } from '@angular/common/http';
import { HttpTestingController, provideHttpClientTesting } from '@angular/common/http/testing';
import { AuthService } from '../../services/auth.service';

describe('LoginComponent', () => {
  let service: AuthService;
  let httpTesting: HttpTestingController;  // Controlador que intercepta las llamadas HTTP

  beforeEach(async () => {
    await TestBed.configureTestingModule({
      providers: [
        provideHttpClient(withFetch()),  // Activa HttpClient con Fetch API
        provideHttpClientTesting(),      // SUSTITUYE el HttpClient real por uno de testing
        AuthService                      // El servicio bajo test
      ],
      imports: [LoginComponent]
    });

    service = TestBed.inject(AuthService);
    httpTesting = TestBed.inject(HttpTestingController);  // Obtener el controlador
  });

  // afterEach: verificar que no quedaron peticiones sin responder
  afterEach(() => {
    httpTesting.verify();  // Falla el test si hay peticiones HTTP que no se procesaron
    // Esto detecta llamadas HTTP inesperadas o que se olvidó responder con flush()
  });

  it('should login with valid credentials', () => {
    const email = 'test@example.com';
    const password = 'password123';
    const mockResponse = { token: 'fake-jwt-token' };  // Respuesta falsa que devolveremos

    // 1. DISPARO la llamada HTTP (el servicio llama a this.http.post())
    service.login(email, password).subscribe({
      next: (response) => {
        // 4. Verifico que la respuesta que recibió el servicio es la correcta
        expect(response.token).toBe('fake-jwt-token');
      }
    });

    // 2. INTERCEPTO la petición antes de que salga a la red
    // expectOne(url): verifica que se hizo exactamente UNA petición a esa URL
    const req = httpTesting.expectOne('/api/login');

    // 3. VERIFICO los detalles de la petición (método, body, headers)
    expect(req.request.method).toBe('POST');
    expect(req.request.body).toEqual({ email, password });

    // 4. RESPONDO manualmente con los datos mock → esto dispara el callback del subscribe
    req.flush(mockResponse);
  });

  it('should handle login error', () => {
    const email = 'test@example.com';
    const password = 'wrong-password';

    service.login(email, password).subscribe({
      next: () => fail('should have failed'),  // fail(): hace fallar el test explícitamente
      error: (error) => {
        expect(error.status).toBe(401);  // Verifico que el error llega correctamente
      }
    });

    const req = httpTesting.expectOne('/api/login');
    // flush() con status de error → simula una respuesta de error del servidor
    req.flush('Unauthorized', { status: 401, statusText: 'Unauthorized' });
  });
});
```

> 💡 `provideHttpClientTesting()` es un **sustituto** de `provideHttpClient()` para tests. Hace que todas las llamadas HTTP queden en cola en lugar de salir a la red. `httpTesting.verify()` en `afterEach` es como una red de seguridad: detecta peticiones no respondidas o inesperadas.

---

## Flujo de un test HTTP — paso a paso

```
1. service.login(email, password).subscribe(...)
   → El servicio llama a this.http.post('/api/login', body)
   → HttpTestingController INTERCEPTA la petición (no sale a la red)

2. const req = httpTesting.expectOne('/api/login')
   → Verifica que EXACTAMENTE una petición fue a '/api/login'
   → 'req' contiene los detalles de la petición (método, body, headers)

3. expect(req.request.method).toBe('POST')
   expect(req.request.body).toEqual({ email, password })
   → Verificamos que la petición tiene los datos correctos

4. req.flush(mockResponse)           ← respuesta exitosa
   req.flush('msg', { status: 401 }) ← respuesta de error
   → Responder manualmente dispara el callback del subscribe()

5. afterEach → httpTesting.verify()
   → Falla si quedó alguna petición sin responder (detecta errores de lógica)
```

---

## El servicio bajo test — `AuthService`

```typescript
// src/app/services/auth.service.ts
import { HttpClient } from '@angular/common/http';
import { inject, Injectable } from '@angular/core';
import { Observable } from 'rxjs';

@Injectable({
  providedIn: 'root'
})
export class AuthService {
  http = inject(HttpClient)  // Se inyecta con inject() en lugar de constructor

  // El método que vamos a testear:
  // Recibe email y password, hace POST a /api/login, devuelve Observable con el token
  login(email: string, password: string): Observable<{ token: string }> {
    return this.http.post<{ token: string }>('/api/login', { email, password })
  }
}
```

> 💡 Al usar `provideHttpClientTesting()` en el test, cuando `AuthService` llama a `this.http.post(...)`, la petición no sale a la red: queda interceptada en el `HttpTestingController`. Desde el test controlamos exactamente qué responde el "servidor".

---

## Tests E2E con Playwright — `playwright.config.ts`

Playwright prueba la aplicación real en un navegador. No mockea nada: arranca la app, abre un browser y simula acciones reales del usuario.

```typescript
// playwright.config.ts
import { defineConfig, devices } from '@playwright/test';

export default defineConfig({
  testDir: './e2e',           // Carpeta donde están los tests E2E
  fullyParallel: true,        // Ejecutar todos los tests en paralelo
  forbidOnly: !!process.env['CI'],  // En CI: fallar si hay test.only accidentales
  retries: process.env['CI'] ? 2 : 0,  // En CI: reintentar tests fallidos 2 veces
  workers: process.env['CI'] ? 1 : undefined,  // En CI: un solo worker (más estable)
  reporter: 'html',           // Generar reporte HTML con capturas y trazas

  use: {
    // URL base — en tests: await page.goto('/') irá a 'http://localhost:4200/'
    baseURL: process.env['PLAYWRIGHT_TEST_BASE_URL'] ?? 'http://localhost:4200',
    trace: 'on-first-retry',  // Grabar trazas solo cuando un test falla y se reintenta
  },

  // Ejecutar los tests en múltiples navegadores
  projects: [
    { name: 'chromium', use: { ...devices['Desktop Chrome'] } },
    { name: 'firefox',  use: { ...devices['Desktop Firefox'] } },
    { name: 'webkit',   use: { ...devices['Desktop Safari'] } },
  ],
});
```

> 💡 La variable `PLAYWRIGHT_TEST_BASE_URL` permite apuntar los tests E2E a diferentes entornos (local, staging, producción) sin cambiar código. En CI se suele configurar para apuntar al servidor de staging.

---

## Tests E2E — estructura y API de Playwright

```typescript
// e2e/example.e2e.spec.ts
import { expect, test } from '@playwright/test';

// test() es el equivalente de 'it()' en Jest
test('has title', async ({ page }) => {
  // page: API del navegador — navegar, hacer click, rellenar forms...

  // Navegar a la raíz de la app (usa baseURL del playwright.config.ts)
  await page.goto('/');

  // Verificar el título del documento HTML (<title>)
  // /MyApp/ es una expresión regular → el título debe CONTENER "MyApp"
  await expect(page).toHaveTitle(/MyApp/);
});
```

```typescript
// Ejemplos de API de Playwright
await page.goto('/login');                          // Navegar
await page.fill('[data-testid="email"]', 'a@b.com'); // Rellenar input
await page.click('[data-testid="submit"]');          // Hacer click
await expect(page).toHaveURL('/dashboard');          // Verificar URL
await expect(page.locator('h1')).toHaveText('Bienvenido'); // Verificar texto
await page.screenshot({ path: 'screenshot.png' });  // Captura de pantalla
```

> 💡 En Playwright, todas las acciones son `async/await`. El navegador es asíncrono por naturaleza. A diferencia de Jest donde `expect` es síncrono, aquí `expect(page).toHaveTitle()` también devuelve una Promise y debe esperarse con `await`.

---

## Diferencia entre `testPathIgnorePatterns` y `projects` en Playwright

```
jest.config.js → testPathIgnorePatterns: ['.*\\.e2e\\.spec\\.ts$']
  → Jest IGNORA los archivos *.e2e.spec.ts (son de Playwright)

playwright.config.ts → testDir: './e2e'
  → Playwright SOLO busca tests en la carpeta ./e2e
```

La convención de nombre `*.e2e.spec.ts` actúa como barrera:
- Jest ve el patrón `.e2e.spec.ts` → lo excluye
- Playwright busca en `./e2e/` → los encuentra

> 💡 Esta separación por convención de nombre y carpeta evita que los dos test runners choquen. Sin `testPathIgnorePatterns`, Jest intentaría ejecutar los tests de Playwright y fallaría porque `@playwright/test` no está disponible en el entorno de Jest.

---

## Patrón Given / When / Then

Estructura estándar para escribir tests legibles. Hace explícito qué es setup, qué es la acción y qué es la verificación:

```typescript
it('should login with valid credentials', () => {
  // GIVEN (dado que...) — el contexto previo
  const email = 'test@example.com';
  const password = 'password123';
  const mockResponse = { token: 'fake-jwt-token' };

  // WHEN (cuando...) — la acción que disparamos
  service.login(email, password).subscribe({
    next: (response) => {
      // THEN (entonces...) — la verificación del resultado
      expect(response.token).toBe('fake-jwt-token');
    }
  });

  // (parte del when: respondemos la petición HTTP)
  const req = httpTesting.expectOne('/api/login');
  req.flush(mockResponse);
});
```

> 💡 Los comentarios `// given`, `// when`, `// then` son opcionales pero muy útiles cuando el test tiene mucho setup. En tests simples basta con seguir la estructura mentalmente.

---

## `TestBed.inject()` — Obtener instancias del DI

```typescript
// Equivalente a usar el DI de Angular en tests
const service = TestBed.inject(AuthService);
const httpTesting = TestBed.inject(HttpTestingController);
const router = TestBed.inject(Router);

// TestBed.inject() es la forma de "inyectar" en tests
// sin necesidad de declararlos en el constructor del componente
```

> 💡 `TestBed.inject(Token)` funciona exactamente como la inyección de dependencias en Angular: busca el proveedor registrado en `configureTestingModule()` y devuelve la instancia. Si el servicio tiene `providedIn: 'root'`, no necesita declararse en `providers[]`.

---

## Resumen de conceptos clave

| Concepto | Herramienta | Qué hace |
|---|---|---|
| **`jest-preset-angular`** | Jest | Preset que permite a Jest compilar y entender Angular |
| **`setupZoneTestEnv`** | setup-jest.ts | Inicializa Zone.js para Change Detection en tests |
| **`@testing-library/jest-dom`** | setup-jest.ts | Matchers extra: `toBeVisible`, `toHaveValue`... |
| **`TestBed.configureTestingModule()`** | `@angular/core/testing` | Declara el módulo de test (imports + providers) |
| **`TestBed.createComponent()`** | `@angular/core/testing` | Crea una instancia del componente para testear |
| **`TestBed.inject()`** | `@angular/core/testing` | Obtiene instancias del DI en el contexto del test |
| **`fixture.componentInstance`** | ComponentFixture | Accede a la clase TypeScript del componente |
| **`fixture.nativeElement`** | ComponentFixture | Accede al DOM HTML del componente |
| **`fixture.detectChanges()`** | ComponentFixture | Aplica cambios de estado al DOM (Change Detection) |
| **`provideHttpClientTesting()`** | `@angular/common/http/testing` | Sustituye HttpClient por versión de test (no sale a red) |
| **`HttpTestingController`** | `@angular/common/http/testing` | Intercepta, verifica y responde peticiones HTTP en tests |
| **`httpTesting.expectOne(url)`** | HttpTestingController | Verifica que se hizo UNA petición a esa URL |
| **`req.flush(data)`** | TestRequest | Responde manualmente la petición con datos mock |
| **`httpTesting.verify()`** | HttpTestingController | Verifica que no hay peticiones sin responder |
| **`describe` / `it`** | Jest | Agrupar tests / definir un test individual |
| **`beforeEach` / `afterEach`** | Jest | Setup/teardown antes/después de cada test |
| **`fail()`** | Jest | Hacer fallar el test explícitamente |
| **`defineConfig()`** | Playwright | Configura el runner E2E (navegadores, URL, reintentos) |
| **`page.goto(url)`** | Playwright | Navegar a una URL en el navegador real |
| **`expect(page).toHaveTitle()`** | Playwright | Verificar el título del documento |
| **`testPathIgnorePatterns`** | jest.config.js | Excluir archivos que no debe correr Jest |
| **Given/When/Then** | Patrón | Estructura semántica de un test legible |
