# Angular: Directivas, Servicios y Arquitectura de Componentes
## Directivas 
Son aquellas que brindan funcionalidad a algo que antes no la tenía.
> 💡 En Angular, las directivas son clases que modifican el comportamiento o la apariencia de elementos del DOM. Son como "superpoderes" que le das a tus elementos HTML.

 - Directiva Estructural:
	+ <div *ngIf=""></div> -> *ngIf = Si se cumple una condición (boolean), el elemento cargará o no.
	+ <div *ngFor=""></div> -> *ngFor = Se renderizará el elemento X  veces
	> 💡 `*ngIf` y `*ngFor` llevan asterisco (*) porque son directivas estructurales: modifican la estructura del DOM añadiendo o eliminando elementos. El asterisco es azúcar sintáctico que Angular transforma internamente en un `<ng-template>`.
	
	> Ejemplo de *ngFor:
	> ```html
	> <!-- items = ['Manzana', 'Pera', 'Naranja'] -->
	> <ul>
	>   <li *ngFor="let item of items; let i = index">
	>     {{i + 1}}. {{item}}
	>   </li>
	> </ul>
	> <!-- Resultado: 1. Manzana, 2. Pera, 3. Naranja -->
	> ```

- Directiva de Atributos
<div [ngClass]="{'active': isActive}"></div> -> En este caso, la clase de estilo se aplicará si está activa
> 💡 Las directivas de atributo NO añaden ni eliminan elementos del DOM, sino que modifican la apariencia o comportamiento del elemento en el que están. Otros ejemplos comunes:
> ```html
> <!-- ngStyle: aplica estilos inline dinámicamente -->
> <div [ngStyle]="{'color': isError ? 'red' : 'green'}">Mensaje</div>
> 
> <!-- ngClass con múltiples clases -->
> <div [ngClass]="{'active': isActive, 'disabled': isDisabled, 'highlight': isNew}">
> ```


- Directiva ejemplo
	Esto es una directiva Estructural porque rendereizamos algo en el DOM
```typescript
	@Directive({ /* Decorador de las directivas. Le dice a Angular que esta clase es una directiva */
		standalone:true, /* No necesita declararse en un NgModule, puede importarse directamente */
		selector: '[appShowonScreenSize]' /* Los corchetes [] indican que se usa como atributo HTML: <div appShowOnScreenSize="small"> */
	})

	export class ShowOnScreenSizeDirective implementes OnInit{
		@Input() appShowOnScreenSize: 'small' | 'medium' | 'large'
		/* @Input() permite que el componente padre pase un valor a la directiva.
		   En este caso, el padre le dice qué tamaño de pantalla debe mostrar el elemento */
		constructor(){
			private templateRef: TemplateRef<any>, /* Referencia al contenido del template donde se usa la directiva (el <div> y su contenido) */
			private viewContainer: ViewContainerRef /* Contenedor donde se puede insertar o eliminar vistas dinámicamente */
		}{}
		ngOnInit(){ /* Ciclo de vida: se ejecuta una vez cuando la directiva se inicializa, similar a componentDidMount en React */
			this.updateView();
			// Escucha el evento de redimensionamiento de la ventana y actualiza la vista en consecuencia
			window.addEventListener('resize', this.updateView.bind(this));
			/* .bind(this) es necesario para que dentro de updateView, 'this' siga refiriéndose a la instancia de la directiva */

			private updateView(){
				const width = window.innerWidth;
				this.viewContainer.clear(); /* Limpia el contenedor antes de volver a renderizar, evita duplicados */

				if(this.shouldContent(width)){
					this.viewContainer.createEmbeddedView(this.templateRef); /* Inserta el template en el DOM si la condición se cumple */
				}
			}
			private shouldContent(width:number):boolean{
				if(this.appShowOnScreenSize === 'small' && width <=600){
					return true;
				}
				if(this.appShowOnScreenSize === 'medium' && width >= 600 && width <= 1024){ 
					return true;
				}
				if(this.appShowOnScreenSize === 'large' && width <= 1024){ /* ⚠️ Posible bug: debería ser width > 1024 para pantallas grandes */ 
					return true;
				}
				return false;
			}
		}
	}
```

```html
<!-- La directiva se usa como un atributo en cualquier elemento HTML -->
<div *appShowOnScreenSize="'small'"> Contenido para pantallas pequeñas</div>
<!-- Este div solo aparecerá en el DOM si el ancho de pantalla es <= 600px -->
```

- Directiva de ejemplo 2
	Esto es una directiva de **Atributo** (no Estructural) porque no añade/elimina elementos del DOM, sino que modifica su estilo

```typescript
	@Directive({
		standalone:true,
		selector: '[appHighlight]'
	})

	export class HighLightDirective {
		constructor(private el:ElementRef, private renderer: Renderer2){}
		/* ElementRef: referencia directa al elemento HTML nativo donde se aplicó la directiva
		   Renderer2: forma segura de manipular el DOM (mejor que acceder a nativeElement directamente,
		   ya que es compatible con SSR/Server Side Rendering y otros entornos no-browser) */
		
		//Decorador para escuchar eventos del host (el elemento donde se aplica la directiva)
		@HostListener('mouseenter') onMouseEnter(){
			this.renderer.setStyle(this.el.nativeElement, 'background-color', 'yellow');
		} 
		/* @HostListener('mouseenter') = equivalente a addEventListener('mouseenter') pero de forma declarativa.
		   Angular se encarga de añadir y eliminar el listener automáticamente */
	
	@HostListener('mouseleave') onMouseLeave(){
		this.renderer.removeStyle(this.el.nativeElement, 'background-color');
	}
```

 ```html
 <!-- Solo añadiendo el atributo appHighlight, el párrafo reaccionará al mouse -->
 <p appHighlight> Pasa el mouse sobre este texto para ver que cambia el contenido </p>
 ```


---

## Pipes
> 💡 Los Pipes son transformadores de datos para las plantillas HTML. Toman un valor de entrada y devuelven un valor transformado para mostrar. No modifican el dato original, solo su presentación.

```html
<!-- Sintaxis: valor | nombrePipe -->
<p>{{ nombre | uppercase }}</p>           <!-- JUAN -->
<p>{{ precio | currency:'EUR' }}</p>       <!-- €1,234.56 -->
<p>{{ fecha | date:'dd/MM/yyyy' }}</p>     <!-- 26/02/2026 -->
<p>{{ descripcion | slice:0:50 }}</p>      <!-- Primeros 50 caracteres -->
<p>{{ objeto | json }}</p>                 <!-- Útil para debug: muestra el objeto como JSON -->
```

```typescript
/* También puedes crear Pipes personalizados */
@Pipe({
  standalone: true,
  name: 'capitalize' /* Nombre con el que se usará en el template */
})
export class CapitalizePipe implements PipeTransform {
  transform(value: string): string {
    if (!value) return '';
    return value.charAt(0).toUpperCase() + value.slice(1).toLowerCase();
  }
}
```

```html
<!-- Uso del pipe personalizado -->
<p>{{ 'hOLA MUNDO' | capitalize }}</p>  <!-- Hola mundo -->
```

> 💡 Los pipes se pueden encadenar: `{{ texto | lowercase | capitalize }}`


---

## Signals
> 💡 Introducidos en Angular 16, los Signals son la nueva forma reactiva de gestionar el estado. Son una alternativa más eficiente a la detección de cambios tradicional de Angular.

```typescript
import { signal, computed, effect } from '@angular/core';

@Component({
  standalone: true,
  selector: 'app-counter',
  template: `
    <p>Contador: {{ count() }}</p>       <!-- Los signals se leen llamándolos como funciones -->
    <p>Doble: {{ doubleCount() }}</p>
    <button (click)="increment()">+1</button>
  `
})
export class CounterComponent {
  /* signal() crea un valor reactivo. Cuando cambia, Angular actualiza solo los templates que lo usan */
  count = signal(0);

  /* computed() crea un signal derivado. Se recalcula automáticamente cuando 'count' cambia */
  doubleCount = computed(() => this.count() * 2);

  increment() {
    this.count.update(val => val + 1); /* update() modifica el valor basándose en el anterior */
    // También puedes usar: this.count.set(5); -> establece un valor directo
  }
}
```

> 💡 Ventaja clave: con Signals, Angular sabe exactamente qué partes del DOM hay que actualizar, sin necesidad de revisar todo el árbol de componentes (como hacía con Zone.js).


---

## Services (Servicios)
- Se utiliza para conectarse con entidades externas, como APIs, o para compartir datos entre componentes.
- Se utiliza para compartir información entre componentes que no tienen una relación directa de padre-hijo.
- SINGLETON => Hay una única instancia del servicio en toda la aplicación, lo que permite compartir datos de manera eficiente.
	+ La información se mantiene constante en toda la aplicación, suele ser mejor que un componente porque no tiene un ciclo de vida, es decir, no se destruye ni se vuelve a crear como los componentes, lo que permite mantener la información de manera persistente.
	> 💡 Patrón Singleton: imagínalo como una única "caja fuerte" compartida. Todos los componentes acceden a la misma caja, por lo que si uno modifica algo, todos ven el cambio.

	+ Por ejemplo es muy útil para controlar un dark mode de una aplicación, ya que el estado del dark mode se mantiene constante en toda la aplicación, incluso si el usuario navega a diferentes componentes o páginas.

- Servicio de ejemplo 1 
```typescript
	@Injectable({
		providedIn: 'root' /* 'root' = disponible en toda la aplicación como Singleton.
		                      Angular crea UNA sola instancia y la reutiliza en todos los componentes */
	})

	export class AuthService {
		isAuthenticated:boolean = false;
		
		/* Método de instancia: necesita acceder a 'this.isAuthenticated', por eso NO es estático */
		changeAuthentication(){
			this.isAuthenticated = !this.isAuthenticated;
		}
		
		/* Método estático: no necesita instancia del servicio para ejecutarse.
		   Se llama como AuthService.login() directamente, sin inyección de dependencias */
		static login(){
			console.log('usuario no autenticado');
		}
	}
```

- Servicio de ejemplo 2
```typescript
	@Injectable({
		providedIn: 'any' /* 'any' = se crea una instancia separada por cada módulo lazy-loaded que lo use.
		                     A diferencia de 'root', puede haber múltiples instancias si hay múltiples módulos */
	})
	export class LoginService {
		log(message:string){
			console.log('log: ', message);
		}
	}
```

- Servicio de ejemplo 3 - Scope local (solo para un componente y sus hijos)
```typescript
	@Component({
		selector: 'app-local',
		template:`<p>Contenido del componente local</p>`,
		providers:[LocalService] /* Al declararlo aquí (y no en @Injectable), se crea una instancia NUEVA
		                           para este componente. No se comparte con el resto de la app.
		                           Los componentes hijos de este también tendrán acceso a esta instancia */
	})

	export class LocalComponent {
		localService = inject(LocalService); /* inject() es la forma moderna de inyectar dependencias.
		                                       Alternativa al constructor: constructor(private ls: LocalService) */
	}
```

---

### Tipos de providers

> 💡 Los `providers` configuran cómo Angular crea e inyecta los servicios. Hay 4 maneras:

**useClass** - Sustituir una clase por otra
```typescript 
	@Injectable()
	export class MockDataService{
		getData(){
			return 'Mock data'
		}
	}

	@Injectable()
	export class RealDataService{
		getData(){
			return 'Real data'
		}
	}
	@Component({
		standalone:true,
		selector:'app-root',
		template: `<p>{{data}}</p>`,
		providers: [
			{ provide: MockDataService, useClass: RealDataService}
			/* Cuando alguien pida MockDataService, Angular le dará RealDataService.
			   Muy útil para testing: en tests usas el Mock, en producción la clase Real */
		]
	})
	export class AppComponent {
		dataService = inject(MockDataService); /* Aunque pedimos Mock, recibiremos Real */
		data = this.dataService.getData();
	}
```

**useValue** - Inyectar un valor constante (no una clase)
```typescript
	const CONFIG = { API_URL: 'https://api.example.com' };

	@Component({
		standalone:true,
		selector:'app-root',
		template: `<p>{{config.API_URL}}</p>`,
		providers: [
			{ provide: 'CONFIG', useValue: CONFIG}
			/* 'CONFIG' es un token de tipo string. Útil para inyectar constantes, configuraciones o
			   valores primitivos que no son clases */
		]
	})
	export class AppConfigComponent {
		config = inject('CONFIG'); /* Inyectamos usando el mismo token string */
	}
```

**useFactory** - Crear el servicio con una función (para lógica de inicialización)
```typescript
	@Injectable()
	export class DataService{
		constructor(public apiUrl: string){} /* El constructor ahora recibe la URL */
	}
	
	export function dataServiceFactory(hostname:string){
		/* La factory decide qué URL usar según el entorno */
		const apiUrl = hostname === 'localhost' ? 'http://localhost:3000' : 'https://api.example.com';
		return new DataService(apiUrl);
	}
	
	@Component({
		standalone:true,
		selector:'app-root',
		template: `<p>{{dataService.apiUrl}}</p>`,
		providers: [
			{ 
				provide: DataService, 
				useFactory: dataServiceFactory,
				deps: [/* dependencias que se pasarán como argumentos a la factory */]
			}
			/* useFactory se usa cuando necesitas lógica para construir el servicio,
			   como leer variables de entorno, hacer cálculos, etc. */
		]
	})
	export class AppDataServiceComponent {
		dataService = inject(DataService);
		data = this.dataService.apiUrl;
	}
```

**useExisting** - Reutilizar una instancia ya existente (alias)
```typescript
	@Injectable()
	export class BaseService {
		getData(){
			return 'Base data';
		}
	}
	
	@Injectable()
	export class DerivedService{
		baseService = inject(BaseService);
		getData(){
			return this.baseService.getData() + ' - derived data ';
		}
	}

	@Component({
		standalone:true,
		selector:'app-root',
		template: `<p>{{derivedService.getData()}}</p>`,
		providers:[
			BaseService,
			{provide: DerivedService, useExisting: BaseService}
			/* useExisting NO crea una nueva instancia, sino que apunta a una ya existente.
			   Diferencia con useClass: useClass crea instancia nueva, useExisting reutiliza la misma.
			   Útil para crear aliases o cuando quieres que dos tokens compartan exactamente la misma instancia */
		]
	})
	export class AppComponent {
		derivedService = inject(DerivedService);
		data = this.derivedService.getData();
	}
```

---

## Ciclo de vida de un componente (Lifecycle Hooks)
> 💡 Angular ejecuta una serie de métodos en momentos concretos de la vida de un componente. Son equivalentes a los hooks de React (useEffect, etc.) pero declarativos.

```typescript
@Component({
	standalone: true,
	selector: 'app-lifecycle',
	template: `<p>{{mensaje}}</p>`
})
export class LifecycleComponent implements OnInit, OnChanges, OnDestroy {
	@Input() dato: string = '';
	mensaje: string = '';

	/* 1. Se ejecuta cuando cambia algún @Input(). Es el primero en ejecutarse */
	ngOnChanges(changes: SimpleChanges){
		console.log('Input cambió:', changes);
	}

	/* 2. Se ejecuta UNA VEZ tras la primera renderización. Ideal para llamadas a APIs */
	ngOnInit(){
		console.log('Componente inicializado');
		// Aquí harías: this.apiService.getData().subscribe(...)
	}

	/* 3. Se ejecuta cada vez que Angular detecta cambios en el componente */
	ngDoCheck(){
		console.log('Comprobando cambios...');
		/* Úsalo con cuidado, se ejecuta muy frecuentemente */
	}

	/* 4. Se ejecuta cuando el componente se destruye (navegas a otra página, *ngIf=false, etc.) */
	ngOnDestroy(){
		console.log('Componente destruido');
		/* Importante: cancelar subscripciones, limpiar listeners, etc. para evitar memory leaks */
		window.removeEventListener('resize', this.miHandler);
	}
}
```

> 💡 Orden de ejecución: `constructor` → `ngOnChanges` → `ngOnInit` → `ngDoCheck` → ... → `ngOnDestroy`


---

## Comunicación entre componentes

### Padre → Hijo: @Input()
```typescript
/* Hijo */
@Component({ selector: 'app-hijo', template: `<p>Hola, {{nombre}}!</p>` })
export class HijoComponent {
	@Input() nombre: string = ''; /* El padre puede pasar un valor a esta propiedad */
}

/* Padre */
// <app-hijo [nombre]="'María'"></app-hijo>
```

### Hijo → Padre: @Output() + EventEmitter
```typescript
/* Hijo: emite un evento cuando el usuario hace algo */
@Component({
	selector: 'app-hijo',
	template: `<button (click)="enviarMensaje()">Enviar</button>`
})
export class HijoComponent {
	@Output() mensaje = new EventEmitter<string>();
	
	enviarMensaje(){
		this.mensaje.emit('Hola desde el hijo!');
	}
}

/* Padre: escucha el evento del hijo */
// <app-hijo (mensaje)="recibirMensaje($event)"></app-hijo>
// recibirMensaje(texto: string){ console.log(texto); }
```

> 💡 Para componentes sin relación directa padre-hijo, usa un **Service** (como se explica arriba) o la librería de estado que prefieras.


---

## HTTP Client
> 💡 Para consumir APIs REST, Angular provee `HttpClient`. Se inyecta como servicio y devuelve Observables.

```typescript
/* app.config.ts - Configuración global */
import { provideHttpClient } from '@angular/common/http';

export const appConfig = {
	providers: [provideHttpClient()]
};
```

```typescript
/* servicio que consume una API */
@Injectable({ providedIn: 'root' })
export class PostsService {
	private http = inject(HttpClient);
	private apiUrl = 'https://jsonplaceholder.typicode.com/posts';

	getPosts() {
		return this.http.get<Post[]>(this.apiUrl); /* Devuelve un Observable */
	}

	createPost(post: Partial<Post>) {
		return this.http.post<Post>(this.apiUrl, post);
	}
}

/* Uso en un componente */
@Component({
	standalone: true,
	imports: [AsyncPipe, NgFor],
	template: `
		<div *ngFor="let post of posts">
			<h3>{{post.title}}</h3>
		</div>
	`
})
export class PostsComponent implements OnInit {
	posts: Post[] = [];
	postsService = inject(PostsService);

	ngOnInit(){
		this.postsService.getPosts().subscribe(data => {
			this.posts = data;
		});
	}
}
```


---

## Routing
> 💡 Angular Router gestiona la navegación entre "páginas" (componentes) sin recargar el navegador (SPA - Single Page Application).

```typescript
/* app.routes.ts */
import { Routes } from '@angular/router';

export const routes: Routes = [
	{ path: '', component: HomeComponent },               /* Ruta raíz */
	{ path: 'about', component: AboutComponent },
	{ path: 'users/:id', component: UserDetailComponent }, /* Parámetro dinámico */
	{ path: '**', component: NotFoundComponent }           /* Ruta comodín (404) */
];
```

```typescript
/* Lazy loading: el módulo/componente solo se carga cuando el usuario navega a esa ruta */
{ 
	path: 'dashboard', 
	loadComponent: () => import('./dashboard/dashboard.component').then(c => c.DashboardComponent)
}
/* Esto mejora el tiempo de carga inicial de la app */
```

```html
<!-- En el template -->
<nav>
	<a routerLink="/">Inicio</a>
	<a routerLink="/about">Sobre nosotros</a>
	<a [routerLink]="['/users', userId]">Ver usuario</a>  <!-- Con variable -->
</nav>

<!-- Aquí se renderiza el componente de la ruta activa -->
<router-outlet></router-outlet>
```

```typescript
/* Navegar desde TypeScript */
export class MiComponent {
	router = inject(Router);
	route = inject(ActivatedRoute);

	irADetalle(id: number){
		this.router.navigate(['/users', id]);
	}

	ngOnInit(){
		/* Leer parámetros de la URL */
		this.route.params.subscribe(params => {
			console.log(params['id']); /* Si la URL es /users/42, imprime 42 */
		});
	}
}
```