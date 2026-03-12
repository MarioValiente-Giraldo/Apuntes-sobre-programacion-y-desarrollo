# RxJS, NgRx y Testing con Jest en Angular

---

## Parte 1: Observable vs BehaviorSubject en RxJS

### ¿Qué es un Observable?

Un **Observable** es el tipo central de la librería RxJS. Representa una secuencia de valores asíncronos que se emiten a lo largo del tiempo. Los observables son **"fríos"** por defecto: no hacen nada hasta que alguien se suscribe a ellos.

- Solo emite valores cuando hay suscriptores.
- No tiene valor inicial.
- No puedes "empujar" valores desde fuera con `.next()`.
- Ideal para: peticiones HTTP, eventos del DOM, streams de datos.

```typescript
import { Observable } from 'rxjs';

const observable$ = new Observable<number>((observer) => {
  observer.next(1);
  observer.next(2);
  observer.next(3);
  observer.complete();
});

// Suscriptor A
observable$.subscribe(value => console.log('A:', value));
// A: 1, A: 2, A: 3

// Suscriptor B (llega tarde) no recibe nada porque el observable ya completó
```

---

### ¿Qué es un Subject?

Un **Subject** es un Observable que también puede actuar como emisor. Puedes usar `.next()` para empujar valores manualmente.

```typescript
import { Subject } from 'rxjs';

const subject$ = new Subject<number>();

subject$.subscribe(v => console.log('Suscriptor A:', v));

subject$.next(1); // A: 1
subject$.next(2); // A: 2

// Un suscriptor que llega tarde NO recibe los valores anteriores
subject$.subscribe(v => console.log('Suscriptor B:', v));
subject$.next(3); // A: 3, B: 3
```

---

### ¿Qué es un BehaviorSubject?

Un **BehaviorSubject** es una extensión de Subject que:

- **Requiere un valor inicial** al crearse.
- **Guarda el último valor emitido** (estado actual).
- **Cuando un nuevo suscriptor llega, recibe inmediatamente el último valor** aunque lo haya perdido.
- Puedes leer el valor actual en cualquier momento con `.getValue()`.

```typescript
import { BehaviorSubject } from 'rxjs';

const behaviorSubject$ = new BehaviorSubject<number>(0); // valor inicial: 0

// Suscriptor A se suscribe y recibe inmediatamente el valor actual (0)
behaviorSubject$.subscribe(v => console.log('A:', v)); // A: 0

behaviorSubject$.next(1); // A: 1
behaviorSubject$.next(2); // A: 2

// Suscriptor B llega tarde pero recibe el último valor (2)
behaviorSubject$.subscribe(v => console.log('B:', v)); // B: 2

behaviorSubject$.next(3); // A: 3, B: 3

// Leer el valor actual sin suscribirse
console.log(behaviorSubject$.getValue()); // 3
```

---

### Tabla comparativa

| Característica | Observable | Subject | BehaviorSubject |
|---|---|---|---|
| Valor inicial | No | No | **Sí (obligatorio)** |
| Emite al suscribirse tarde | No | No | **Sí (último valor)** |
| Puede emitir con `.next()` | No | Sí | Sí |
| `.getValue()` disponible | No | No | **Sí** |
| Ideal para | HTTP, streams | Eventos manuales | **Estado compartido** |

---

### Caso de uso real: Servicio compartido entre componentes

```typescript
// auth.service.ts
import { Injectable } from '@angular/core';
import { BehaviorSubject, Observable } from 'rxjs';

interface User {
  name: string;
  role: string;
}

@Injectable({ providedIn: 'root' })
export class AuthService {
  // BehaviorSubject privado (fuente de verdad)
  private currentUserSubject = new BehaviorSubject<User | null>(null);

  // Observable público (solo lectura para los componentes)
  currentUser$: Observable<User | null> = this.currentUserSubject.asObservable();

  login(user: User): void {
    this.currentUserSubject.next(user);
  }

  logout(): void {
    this.currentUserSubject.next(null);
  }

  getCurrentUser(): User | null {
    return this.currentUserSubject.getValue();
  }
}
```

```typescript
// header.component.ts
@Component({ ... })
export class HeaderComponent {
  user$ = inject(AuthService).currentUser$;
}
```

```html
<!-- header.component.html -->
<div *ngIf="user$ | async as user">
  Bienvenido, {{ user.name }}
</div>
```

> **Buena práctica:** expón siempre el BehaviorSubject como `.asObservable()` para evitar que componentes externos llamen a `.next()` directamente.

---

## Parte 2: Entendiendo NgRx

### ¿Qué es NgRx?

**NgRx** es una librería para Angular basada en el patrón **Redux**. Su objetivo es gestionar el **estado global** de la aplicación de forma reactiva, centralizada y predecible.

Se basa en un flujo unidireccional de datos:

```
Componente → dispatch(Action) → Reducer → Store → Selector → Componente
                                     ↕
                                  Effects (async)
```

---

### Instalación

```bash
ng add @ngrx/store
ng add @ngrx/effects
ng add @ngrx/store-devtools  # DevTools para debugging
```

---

### Conceptos clave

#### 1. Store

El **Store** es el contenedor centralizado del estado. Es un único objeto inmutable que representa el estado completo de la app.

```typescript
// Estado global de ejemplo
interface AppState {
  products: ProductsState;
  user: UserState;
}
```

---

#### 2. Actions

Las **Actions** son objetos simples que describen **qué ocurrió** en la aplicación. Son la única forma de cambiar el estado.

```typescript
import { createAction, props } from '@ngrx/store';
import { Product } from './product.model';

// Acción sin payload
export const loadProducts = createAction('[Products Page] Load Products');

// Acción con payload
export const loadProductsSuccess = createAction(
  '[Products API] Load Products Success',
  props<{ products: Product[] }>()
);

export const loadProductsFailure = createAction(
  '[Products API] Load Products Failure',
  props<{ error: string }>()
);
```

---

#### 3. Reducers

Un **Reducer** es una función **pura** que recibe el estado actual y una action, y devuelve un **nuevo estado**. Nunca muta el estado directamente.

```typescript
import { createReducer, on } from '@ngrx/store';
import { loadProducts, loadProductsSuccess, loadProductsFailure } from './products.actions';
import { Product } from './product.model';

export interface ProductsState {
  products: Product[];
  loading: boolean;
  error: string | null;
}

const initialState: ProductsState = {
  products: [],
  loading: false,
  error: null,
};

export const productsReducer = createReducer(
  initialState,

  on(loadProducts, (state) => ({
    ...state,
    loading: true,
    error: null,
  })),

  on(loadProductsSuccess, (state, { products }) => ({
    ...state,
    loading: false,
    products,
  })),

  on(loadProductsFailure, (state, { error }) => ({
    ...state,
    loading: false,
    error,
  }))
);
```

---

#### 4. Selectors

Los **Selectors** son funciones que extraen porciones específicas del Store para los componentes. Son memoizados (eficientes).

```typescript
import { createFeatureSelector, createSelector } from '@ngrx/store';
import { ProductsState } from './products.reducer';

// Selector de la feature
const selectProductsState = createFeatureSelector<ProductsState>('products');

// Selectores derivados
export const selectAllProducts = createSelector(
  selectProductsState,
  (state) => state.products
);

export const selectProductsLoading = createSelector(
  selectProductsState,
  (state) => state.loading
);

export const selectProductById = (id: number) => createSelector(
  selectAllProducts,
  (products) => products.find(p => p.id === id)
);
```

---

#### 5. Effects

Los **Effects** manejan las **operaciones asíncronas** (llamadas HTTP, etc.). Escuchan actions, ejecutan una tarea y dispatchen nuevas actions con el resultado.

```typescript
import { Injectable } from '@angular/core';
import { Actions, createEffect, ofType } from '@ngrx/effects';
import { ProductsService } from './products.service';
import { loadProducts, loadProductsSuccess, loadProductsFailure } from './products.actions';
import { catchError, map, switchMap } from 'rxjs/operators';
import { of } from 'rxjs';

@Injectable()
export class ProductsEffects {
  loadProducts$ = createEffect(() =>
    this.actions$.pipe(
      ofType(loadProducts),
      switchMap(() =>
        this.productsService.getAll().pipe(
          map((products) => loadProductsSuccess({ products })),
          catchError((error) => of(loadProductsFailure({ error: error.message })))
        )
      )
    )
  );

  constructor(
    private actions$: Actions,
    private productsService: ProductsService
  ) {}
}
```

---

#### 6. Componente conectado al Store

```typescript
@Component({
  selector: 'app-products',
  template: `
    <div *ngIf="loading$ | async">Cargando...</div>
    <div *ngIf="error$ | async as error">Error: {{ error }}</div>
    <ul>
      <li *ngFor="let product of products$ | async">{{ product.name }}</li>
    </ul>
    <button (click)="load()">Cargar productos</button>
  `
})
export class ProductsComponent {
  products$ = this.store.select(selectAllProducts);
  loading$  = this.store.select(selectProductsLoading);
  error$    = this.store.select(selectProductsError);

  constructor(private store: Store) {}

  load(): void {
    this.store.dispatch(loadProducts());
  }
}
```

---

#### 7. Registro en el módulo

```typescript
// app.module.ts
@NgModule({
  imports: [
    StoreModule.forRoot({ products: productsReducer }),
    EffectsModule.forRoot([ProductsEffects]),
    StoreDevtoolsModule.instrument({ maxAge: 25 }),
  ]
})
export class AppModule {}
```

---

### ¿Cuándo usar NgRx?

| Usar NgRx | No usar NgRx |
|---|---|
| Estado compartido entre muchos componentes | Apps pequeñas con estado local simple |
| Flujos de datos complejos con async | Estado que vive en un solo componente |
| Múltiples fuentes modificando el mismo estado | Prototipado rápido |
| Necesitas debugging con DevTools | Overhead no justificado |

---

## Parte 3: Testing con Jest en Angular

### Configuración inicial

```bash
# Instalar Jest en un proyecto Angular
npm install --save-dev jest @types/jest jest-preset-angular
```

```js
// jest.config.js
module.exports = {
  preset: 'jest-preset-angular',
  setupFilesAfterFramework: ['<rootDir>/setup-jest.ts'],
  testPathPattern: '.*\\.spec\\.ts$',
};
```

```ts
// setup-jest.ts
import 'jest-preset-angular/setup-jest';
```

---

### Testeando un Servicio con BehaviorSubject

```typescript
// auth.service.spec.ts
import { AuthService } from './auth.service';

describe('AuthService', () => {
  let service: AuthService;

  beforeEach(() => {
    service = new AuthService();
  });

  it('debe tener null como valor inicial', () => {
    expect(service.getCurrentUser()).toBeNull();
  });

  it('debe emitir el usuario al hacer login', (done) => {
    const mockUser = { name: 'Mario', role: 'admin' };

    service.currentUser$.subscribe(user => {
      if (user) {
        expect(user.name).toBe('Mario');
        done();
      }
    });

    service.login(mockUser);
  });

  it('debe emitir null al hacer logout', (done) => {
    const mockUser = { name: 'Mario', role: 'admin' };
    service.login(mockUser);

    service.logout();

    service.currentUser$.subscribe(user => {
      expect(user).toBeNull();
      done();
    });
  });
});
```

---

### Testeando Reducers de NgRx

Los reducers son funciones puras, por lo tanto son **muy fáciles de testear**: sin mocks, sin async.

```typescript
// products.reducer.spec.ts
import { productsReducer, initialState } from './products.reducer';
import { loadProducts, loadProductsSuccess, loadProductsFailure } from './products.actions';

describe('ProductsReducer', () => {
  it('debe retornar el estado inicial', () => {
    const state = productsReducer(undefined, { type: '@@INIT' });
    expect(state).toEqual(initialState);
  });

  it('debe poner loading en true al disparar loadProducts', () => {
    const state = productsReducer(initialState, loadProducts());
    expect(state.loading).toBe(true);
    expect(state.error).toBeNull();
  });

  it('debe cargar productos correctamente', () => {
    const products = [{ id: 1, name: 'Laptop' }];
    const state = productsReducer(initialState, loadProductsSuccess({ products }));

    expect(state.loading).toBe(false);
    expect(state.products).toEqual(products);
  });

  it('debe manejar el error correctamente', () => {
    const state = productsReducer(
      initialState,
      loadProductsFailure({ error: 'Error de red' })
    );

    expect(state.loading).toBe(false);
    expect(state.error).toBe('Error de red');
  });
});
```

---

### Testeando Selectors de NgRx

```typescript
// products.selectors.spec.ts
import { selectAllProducts, selectProductsLoading } from './products.selectors';
import { ProductsState } from './products.reducer';

describe('ProductsSelectors', () => {
  const mockState: ProductsState = {
    products: [
      { id: 1, name: 'Laptop' },
      { id: 2, name: 'Mouse' },
    ],
    loading: false,
    error: null,
  };

  it('debe seleccionar todos los productos', () => {
    const result = selectAllProducts.projector(mockState);
    expect(result).toHaveLength(2);
    expect(result[0].name).toBe('Laptop');
  });

  it('debe seleccionar el estado de carga', () => {
    const result = selectProductsLoading.projector(mockState);
    expect(result).toBe(false);
  });
});
```

---

### Testeando Effects de NgRx

```typescript
// products.effects.spec.ts
import { TestBed } from '@angular/core/testing';
import { provideMockActions } from '@ngrx/effects/testing';
import { of, throwError } from 'rxjs';
import { ReplaySubject } from 'rxjs';
import { ProductsEffects } from './products.effects';
import { ProductsService } from './products.service';
import { loadProducts, loadProductsSuccess, loadProductsFailure } from './products.actions';

describe('ProductsEffects', () => {
  let actions$: ReplaySubject<any>;
  let effects: ProductsEffects;
  let productsService: jest.Mocked<ProductsService>;

  beforeEach(() => {
    actions$ = new ReplaySubject(1);

    TestBed.configureTestingModule({
      providers: [
        ProductsEffects,
        provideMockActions(() => actions$),
        {
          provide: ProductsService,
          useValue: { getAll: jest.fn() },
        },
      ],
    });

    effects = TestBed.inject(ProductsEffects);
    productsService = TestBed.inject(ProductsService) as jest.Mocked<ProductsService>;
  });

  it('debe dispatchar loadProductsSuccess al cargar con éxito', (done) => {
    const products = [{ id: 1, name: 'Laptop' }];
    productsService.getAll.mockReturnValue(of(products));

    actions$.next(loadProducts());

    effects.loadProducts$.subscribe(action => {
      expect(action).toEqual(loadProductsSuccess({ products }));
      done();
    });
  });

  it('debe dispatchar loadProductsFailure cuando hay un error', (done) => {
    productsService.getAll.mockReturnValue(throwError(() => new Error('Error de red')));

    actions$.next(loadProducts());

    effects.loadProducts$.subscribe(action => {
      expect(action).toEqual(loadProductsFailure({ error: 'Error de red' }));
      done();
    });
  });
});
```

---

### Testeando un Componente con Store mockeado

```typescript
// products.component.spec.ts
import { ComponentFixture, TestBed } from '@angular/core/testing';
import { provideMockStore, MockStore } from '@ngrx/store/testing';
import { ProductsComponent } from './products.component';
import { selectAllProducts, selectProductsLoading } from './products.selectors';

describe('ProductsComponent', () => {
  let fixture: ComponentFixture<ProductsComponent>;
  let store: MockStore;

  const initialState = {
    products: { products: [], loading: false, error: null }
  };

  beforeEach(() => {
    TestBed.configureTestingModule({
      declarations: [ProductsComponent],
      providers: [provideMockStore({ initialState })],
    });

    fixture = TestBed.createComponent(ProductsComponent);
    store = TestBed.inject(MockStore);
    fixture.detectChanges();
  });

  it('debe crearse correctamente', () => {
    expect(fixture.componentInstance).toBeTruthy();
  });

  it('debe mostrar los productos del store', () => {
    store.overrideSelector(selectAllProducts, [
      { id: 1, name: 'Laptop' }
    ]);
    store.refreshState();
    fixture.detectChanges();

    const items = fixture.nativeElement.querySelectorAll('li');
    expect(items.length).toBe(1);
    expect(items[0].textContent).toContain('Laptop');
  });

  it('debe dispatchar loadProducts al hacer click en el botón', () => {
    const dispatchSpy = jest.spyOn(store, 'dispatch');
    const button = fixture.nativeElement.querySelector('button');

    button.click();

    expect(dispatchSpy).toHaveBeenCalledWith(loadProducts());
  });
});
```

---

### Matchers útiles de Jest

```typescript
// Igualdad
expect(value).toBe(42);               // ===
expect(obj).toEqual({ id: 1 });       // deep equality

// Verdad/Falsedad
expect(value).toBeTruthy();
expect(value).toBeFalsy();
expect(value).toBeNull();
expect(value).toBeUndefined();

// Arrays / Strings
expect(arr).toHaveLength(3);
expect(arr).toContain('item');
expect(str).toContain('texto');

// Funciones / Spies
expect(fn).toHaveBeenCalled();
expect(fn).toHaveBeenCalledWith('param');
expect(fn).toHaveBeenCalledTimes(2);

// Errores
expect(() => fn()).toThrow();
expect(() => fn()).toThrow('mensaje de error');
```

---

## Resumen general

| Concepto | Para qué sirve |
|---|---|
| **Observable** | Stream de datos asíncronos, solo lectura |
| **BehaviorSubject** | Estado compartido con valor actual accesible |
| **NgRx Store** | Estado global centralizado e inmutable |
| **Actions** | Describen eventos que ocurren en la app |
| **Reducers** | Funciones puras que calculan el nuevo estado |
| **Selectors** | Extraen y memoizan partes del estado |
| **Effects** | Manejan side effects (HTTP, etc.) |
| **Jest** | Framework de testing con mocks y matchers potentes |
