# Angular: Interceptors HTTP

> 💡 Un **interceptor** es un middleware que se ejecuta automáticamente en **cada petición HTTP** que hace la app (y en cada respuesta que recibe). Sirve para añadir headers, gestionar tokens, manejar errores globales y refrescar sesiones sin repetir código en cada servicio. Angular 17+ usa **interceptores funcionales** (`HttpInterceptorFn`) en lugar de clases.

---

## ¿Qué problema resuelven los interceptors?

Sin interceptor, tendrías que añadir el token de autenticación manualmente en cada llamada HTTP:

```typescript
// Sin interceptor: repetición en cada servicio
this.http.get('/api/users', {
  headers: { Authorization: `Bearer ${localStorage.getItem('token')}` }
})

this.http.get('/api/products', {
  headers: { Authorization: `Bearer ${localStorage.getItem('token')}` }
})

// Con interceptor: se añade automáticamente a TODAS las peticiones
this.http.get('/api/users')    // el interceptor añade el token
this.http.get('/api/products') // el interceptor añade el token
```

> 💡 Los interceptores aplican lógica transversal (cross-cutting concerns): autenticación, logging, gestión de errores, caché. Todo en un único lugar, sin tocar los servicios individuales.

---

## Interceptor Funcional — `HttpInterceptorFn`

Angular 17+ introduce los interceptores como **funciones puras** en lugar de clases. La firma siempre tiene dos parámetros: la petición (`req`) y el handler (`next`) que ejecuta la siguiente etapa del pipeline.

```typescript
// src/app/interceptors/auth.interceptor.ts
import { HttpInterceptorFn } from '@angular/common/http';

// HttpInterceptorFn es un tipo: (req, next) => Observable<HttpEvent<unknown>>
export const authInterceptor: HttpInterceptorFn = (req, next) => {
  // 'req'  → la petición HTTP original (inmutable)
  // 'next' → función que pasa la petición al siguiente interceptor o al servidor

  // Para pasar la petición sin modificar:
  return next(req);

  // Para modificar la petición: hay que clonarla (ver req.clone() más abajo)
};
```

> 💡 La función se declara como una `const` con tipo `HttpInterceptorFn`. Angular se encarga de inyectarla en el pipeline HTTP automáticamente al registrarla en `app.config.ts`.

---

## `req.clone()` — La inmutabilidad de las peticiones

Las peticiones HTTP en Angular son **inmutables**. No se puede modificar `req.headers` directamente. Hay que crear una copia con `req.clone()` pasando los cambios.

```typescript
export const authInterceptor: HttpInterceptorFn = (req, next) => {
  // INCORRECTO: las peticiones son inmutables, esto falla en tiempo de ejecución
  // req.headers.set('Content-Type', 'application/json') // ← No muta el original

  // CORRECTO: construir el nuevo header a partir del existente
  let headers = req.headers.set('Content-Type', 'application/json')
  // headers.set() devuelve un NUEVO objeto HttpHeaders, no muta el original

  // req.clone() crea una nueva petición con los cambios indicados
  const newReq = req.clone({ headers })

  // Pasamos la nueva petición (con headers modificados) al siguiente eslabón
  return next(newReq)
};
```

> 💡 `req.headers.set()` devuelve un **nuevo** `HttpHeaders` sin mutar el original. `req.clone()` devuelve una **nueva** `HttpRequest` aplicando solo los campos que se le pasen. Esto garantiza que cada interceptor trabaja con copias limpias.

---

## Añadir el token de autenticación

El caso de uso más habitual: leer el token de `localStorage` y añadirlo como header `Authorization` en cada petición.

```typescript
export const authInterceptor: HttpInterceptorFn = (req, next) => {
  const token = localStorage.getItem('token') // Leemos el token guardado tras el login

  // Empezamos siempre con el header de Content-Type
  let headers = req.headers.set('Content-Type', 'application/json')

  // Solo añadimos Authorization si hay token (no todas las rutas lo necesitan)
  if (token) {
    headers = headers.set('Authorization', `Bearer ${token}`)
    // Bearer <token> es el estándar para autenticación JWT
  }

  // Clonamos la petición con los headers actualizados y la enviamos
  const authReq = req.clone({ headers })
  return next(authReq)
};
```

> 💡 Separar `Content-Type` del token es importante: hay rutas públicas (login, registro) que necesitan `Content-Type` pero NO el token. El condicional `if(token)` lo gestiona automáticamente.

---

## SSR Awareness — `isPlatformServer` / `isPlatformBrowser`

Cuando la app usa **Server-Side Rendering** (SSR), el interceptor también se ejecuta en el servidor, donde `localStorage` **no existe**. Hay que detectar la plataforma antes de acceder.

```typescript
import { isPlatformBrowser, isPlatformServer } from '@angular/common';
import { inject, PLATFORM_ID } from '@angular/core';

export const authInterceptor: HttpInterceptorFn = (req, next) => {
  // PLATFORM_ID es un token de inyección que indica si estamos en browser o server
  const platformId = inject(PLATFORM_ID)

  // Si estamos en el servidor, saltamos toda la lógica del interceptor
  // (localStorage no existe en Node.js, esto evitaría errores de runtime)
  if (isPlatformServer(platformId)) {
    return next(req) // Dejamos pasar la petición sin modificar
  }

  // A partir de aquí sabemos que estamos en el navegador → localStorage disponible
  const token = localStorage.getItem('token')
  // ... resto de la lógica
};
```

> 💡 `isPlatformServer()` devuelve `true` cuando Angular se renderiza en Node.js (SSR). `isPlatformBrowser()` devuelve `true` en el navegador. Usar `inject(PLATFORM_ID)` dentro del interceptor funcional es posible gracias al contexto de inyección que Angular proporciona automáticamente.

---

## Manejo de errores con `catchError`

Los interceptores pueden interceptar también las **respuestas de error**. Con `catchError` del pipeline RxJS, se captura el error antes de que llegue al servicio.

```typescript
import { HttpErrorResponse, HttpInterceptorFn } from '@angular/common/http';
import { catchError, throwError } from 'rxjs';

export const authInterceptor: HttpInterceptorFn = (req, next) => {
  // ...lógica de headers...
  const authReq = req.clone({ headers })

  return next(authReq).pipe(
    // catchError intercepta cualquier error HTTP en la respuesta
    catchError((error: HttpErrorResponse) => {
      if (error.status === 401 || error.status === 403) {
        // 401 Unauthorized: token inválido
        // 403 Forbidden: token expirado o sin permisos
        // → Aquí tratamos de renovar el token (ver siguiente sección)
      }

      // Para cualquier otro error, lo relanzamos hacia el servicio que hizo la petición
      return throwError(() => error)
      // throwError() crea un Observable que emite un error, terminando el stream
    })
  );
};
```

> 💡 `catchError` recibe el error y debe devolver siempre un Observable. Si no queremos manejar el error (solo queremos observarlo), usamos `throwError(() => error)` para propagarlo hacia abajo. Si queremos recuperarnos del error, devolvemos un Observable con datos alternativos.

---

## Token Refresh Flow — `switchMap`

El flujo más complejo: cuando un token expira (error 401/403), el interceptor pide automáticamente un nuevo token y **reintenta la petición original** con él. Esto es transparente para el servicio que hizo la llamada.

```typescript
import { catchError, switchMap, throwError } from 'rxjs';
import { AuthService } from '../services/auth.service';

export const authInterceptor: HttpInterceptorFn = (req, next) => {
  const authService = inject(AuthService)
  const token = localStorage.getItem('token')

  let headers = req.headers.set('Content-Type', 'application/json')
  if (token) { headers = headers.set('Authorization', `Bearer ${token}`) }

  const authReq = req.clone({ headers })

  return next(authReq).pipe(
    catchError((error: HttpErrorResponse) => {
      if (error.status === 401 || error.status === 403) {

        // 1. Pedimos al servidor un nuevo token usando el refreshToken
        return authService.refreshToken().pipe(

          // switchMap: cuando refreshToken() emite el nuevo token,
          // cancelamos el observable anterior y creamos uno nuevo (el reintento)
          switchMap((newToken) => {
            // 2. Guardamos el nuevo token en localStorage
            localStorage.setItem('token', newToken)

            // 3. Construimos la petición original con el nuevo token
            const updatedHeaders = req.headers.set('Authorization', `Bearer ${newToken}`)
            const newRequest = req.clone({ headers: updatedHeaders })

            // 4. Reintentamos la petición original con el token renovado
            return next(newRequest)
          })
        )
      }

      // Cualquier otro error → lo propagamos sin intervenir
      return throwError(() => error)
    })
  );
};
```

```
Flujo del token refresh:

Petición del servicio
        ↓
  authInterceptor añade token
        ↓
  Servidor responde 401/403 (token expirado)
        ↓
  catchError intercepta el error
        ↓
  authService.refreshToken() → POST /token con refreshToken
        ↓
  switchMap recibe el nuevo token
        ↓
  Se clona la petición original con el nuevo token
        ↓
  next(newRequest) → reintento transparente
        ↓
  El servicio recibe la respuesta como si nada hubiera pasado
```

> 💡 `switchMap` es perfecto aquí porque cancela el Observable anterior (el error) y genera un nuevo Observable (el reintento). Sin `switchMap`, no podríamos encadenar el refresh con el reintento dentro de un `catchError`.

---

## AuthService — Gestión del Token de Refresco

El servicio de autenticación centraliza la lógica de renovación de tokens y el logout.

```typescript
// src/app/services/auth.service.ts
import { HttpClient } from '@angular/common/http';
import { inject, Injectable } from '@angular/core';
import { Router } from '@angular/router';
import { map, Observable, tap } from 'rxjs';

@Injectable({
  providedIn: 'root'
})
export class AuthService {
  baseUrl = 'http://localhost:4000' // En producción, usar variables de entorno

  http = inject(HttpClient)
  router = inject(Router)

  refreshToken(): Observable<string> {
    const refreshToken = localStorage.getItem('refreshToken')

    // Si no hay refreshToken, la sesión está totalmente caducada → logout
    if (!refreshToken) {
      this.logout()
    }

    // Llamada al endpoint que devuelve un nuevo accessToken
    return this.http.post<{ refreshToken: string }>(`${this.baseUrl}/token`, { refreshToken })
      .pipe(
        // map: transformamos la respuesta del servidor (objeto) en solo el token (string)
        map(response => response.refreshToken),

        // tap: efecto secundario — guardamos el nuevo token sin alterar el stream
        // tap no modifica el valor que fluye por el Observable, solo "lo toca"
        tap((newAccessToken: string) => {
          localStorage.setItem('token', newAccessToken)
          return newAccessToken
        })
      )
  }

  logout() {
    localStorage.clear()               // Eliminamos todos los tokens
    this.router.navigate(['/login'])   // Redirigimos al login
  }
}
```

> 💡 `tap()` se usa para **efectos secundarios** (guardar en localStorage, hacer logging) sin modificar el valor que fluye por el Observable. `map()` sí transforma el valor: convierte el objeto `{ refreshToken: string }` en solo el `string` del token.

---

## Registrar el interceptor — `withInterceptors()`

Los interceptores funcionales se registran en `app.config.ts` usando `withInterceptors()`. El orden del array importa: se ejecutan en orden para las peticiones, y en orden inverso para las respuestas.

```typescript
// src/app/app.config.ts
import { ApplicationConfig } from '@angular/core';
import { provideRouter } from '@angular/router';
import { provideHttpClient, withFetch, withInterceptors } from '@angular/common/http';
import { authInterceptor } from './interceptors/auth.interceptor';
import { routes } from './app.routes';

export const appConfig: ApplicationConfig = {
  providers: [
    provideHttpClient(
      withFetch(),                          // Usa la Fetch API en lugar de XMLHttpRequest
      withInterceptors([authInterceptor])   // Registra los interceptores funcionales
      // Si hay varios: withInterceptors([loggingInterceptor, authInterceptor, errorInterceptor])
    ),
    provideRouter(routes)
  ]
};
```

> 💡 `withFetch()` hace que Angular use la **Fetch API** nativa del navegador en lugar de `XMLHttpRequest`. Esto es compatible con SSR, mejora el rendimiento y permite usar features modernas como streaming. Es recomendado en Angular 17+.

---

## `map` vs `tap` en RxJS

Dos operadores RxJS muy usados en el flujo de interceptores y servicios:

```typescript
import { map, tap } from 'rxjs';

// map: TRANSFORMA el valor que fluye por el Observable
// El siguiente operador recibe el valor transformado
observable.pipe(
  map(response => response.data)   // { data: [...], total: 100 } → [...]
  // A partir de aquí, el stream contiene solo el array, no el objeto completo
)

// tap: OBSERVA el valor sin modificarlo (efecto secundario)
// El siguiente operador recibe exactamente el mismo valor
observable.pipe(
  tap(value => console.log('Debug:', value)),  // Solo para logging / side effects
  tap(token => localStorage.setItem('token', token)), // Guardar sin interrumpir el stream
  // A partir de aquí, el stream sigue con el mismo valor que entró
)
```

| Operador | Modifica el valor | Caso de uso |
|---|---|---|
| `map` | Sí | Transformar respuesta de API, extraer propiedad |
| `tap` | No | Logging, guardar en localStorage, efectos secundarios |
| `catchError` | Depende | Recuperarse de errores o relanzarlos |
| `switchMap` | Sí (cambia Observable) | Encadenar llamadas HTTP dependientes |

---

## Flujo completo de una petición autenticada

```
Componente / Servicio
        ↓
  this.http.get('/api/data')
        ↓
  ── authInterceptor ──────────────────────────────────────────
  │  1. isPlatformServer? → si sí, pasar sin modificar         │
  │  2. Leer token de localStorage                             │
  │  3. Clonar petición con headers (Content-Type + Bearer)    │
  │  4. next(authReq) → enviar al servidor                     │
  │                                                            │
  │  ← Respuesta del servidor                                  │
  │  5. catchError:                                            │
  │     - 401/403 → refreshToken() → switchMap → reintento    │
  │     - otros errores → throwError()                         │
  ── ─────────────────────────────────────────────────────────
        ↓
  Componente / Servicio recibe la respuesta (o el error final)
```

---

## Resumen de conceptos clave

| Concepto | Qué es | Para qué sirve |
|---|---|---|
| **`HttpInterceptorFn`** | Tipo de la función interceptora `(req, next) => Observable` | Define la firma del interceptor funcional |
| **`req.clone()`** | Crea una copia inmutable de la petición con cambios | Modificar headers, URL, body sin mutar el original |
| **`req.headers.set()`** | Devuelve un nuevo `HttpHeaders` con el header añadido | Construir los headers de forma inmutable |
| **`withInterceptors([])`** | Registra interceptores funcionales en el pipeline HTTP | Conectar el interceptor a todas las peticiones |
| **`withFetch()`** | Activa la Fetch API nativa en lugar de XHR | Compatibilidad SSR, mejor rendimiento |
| **`catchError`** | Operador RxJS que captura errores del Observable | Manejar errores HTTP globalmente |
| **`throwError()`** | Crea un Observable que emite un error | Propagar errores que no queremos manejar |
| **`switchMap`** | Cancela el Observable actual y crea uno nuevo | Encadenar el refresh de token con el reintento |
| **`map`** | Transforma el valor del Observable | Extraer datos de la respuesta del servidor |
| **`tap`** | Efecto secundario sin modificar el valor | Guardar token, logging, side effects |
| **`PLATFORM_ID`** | Token de inyección que identifica la plataforma | Detectar si estamos en browser o servidor (SSR) |
| **`isPlatformServer()`** | Devuelve `true` si la ejecución es en Node.js | Evitar acceder a `localStorage` en SSR |
| **Token Refresh Flow** | Patrón: 401 → refreshToken → reintento | Renovar la sesión automáticamente sin logout |
