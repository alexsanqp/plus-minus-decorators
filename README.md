# @plus-minus/decorators

An Angular decorators.

## Installation

- `npm install --save @plus-minus/decorators`

## Usage

#### Mixins

Decorator that helps you implement incomplete multiple inheritance.

**Mixin** classes do not support DI. Used to refine the behavior of other classes,
not intended to spawn self-used objects. The mixins are useful for bringing common
functionality into a single place and being reusable.

```ts
  class SubscriptionMixin {
     // ...
  }
 
  class GoodbyeMixin {
     // ...
  }
 
  class GreeterMixin {
     public get fullName(): string {
       return `${this.firstName} ${this.surName}`;
     }
 
     public constructor(
       private firstName: string,
       private surName: string
     ) {
     }
 
     public greet(): string {
       return `Hello, ${this.firstName} ${this.surName}!`;
     }
  }
```

The parent class needs to implement the mixin classes (only the declaration of properties or methods)
so that the project can be built without errors at the compilation stage.

If the mixin class has private fields or methods that you do not want
to declare in the parent class, then you can create your own type/interface and declare it:

```ts
  type GreeterMixinType = Omit<GreeterMixin, 'firstName' | 'surName'>;
  // or
  interface GreeterInterface extends Omit<GreeterMixin, 'firstName' | 'surName'> {}
```

If you want to pass parameters to the constructor of the mixin class,
you need to pass an array where the first element will be the mixin class itself,
and all subsequent elements are its parameters.

```ts
  import { Mixins } from '@plus-minus/decorators';
 
  @Component({...})
  @Mixins([
     SubscriptionMixin,
     GoodbyeMixin,
     [GreeterMixin, 'John', 'Wick'],
  ])
  class HelloWorld implements GreeterMixinType, SubscriptionMixin, GoodbyeMixin {
     public fullName: string;
     public greet: () => string;
     // Other declarations: SubscriptionMixin, GoodbyeMixin
 
     public constructor() {
       alert(this.greet());
     }
 }
```
----

#### Memorize
    In computing, memoization or memoisation is an optimization technique used
    primarily to speed up computer programs by storing the results of expensive
    function calls and returning the cached result when the same inputs occur
    again.

##### Notes
    Memoization better used only with pure functions.

You can pass a parameter object `MemorizeOptions`, where: 
* **size** is the size of the list that needs to be cached, the default value is 20, this means that we can cache a list that does not exceed 20 elements.
* **duration** is cache expiration time in milliseconds. Default value is 0ms.

```ts
interface MemorizeOptions {
  size?: number;
  duration?: number;
}
```

**Where can you use Memoization?** For example, you have a list of products,

```ts
interface Product {
  title: string;
  vendorCode: string;
  cost: number; // in cents
}
```

the name of which is concatenated from several properties from the product object, and the price adjusted based on the exchange rate against the dollar, then your list will, at best, be recalculated once per change detection.

```ts
@Injectable()
class ProductService {
  public findAll(): Product[] {
    return [
      {
        title: 'Subframe for Lexus CT',
        vendorCode: '2062CF0EF',
        cost: 41500,
      },
      {
        title: 'Gearbox for Lexus CT',
        vendorCode: '1FCA2E90D',
        cost: 66500,
      },
      {
        title: 'Rear bumper for Lexus IS',
        vendorCode: '186882573',
        cost: 23000,
      },
    ];
  }
}
```

If you use memoization, then it will be executed once, and the cached value of the result of your method will be given to all subsequent changes.

```ts
import { Memorize } from '@plus-minus/decorators';

@Component({
  selector: 'pm-root',
  template: `<div class="col-12">
    <div *ngFor="let product of productService.findAll();" class="d-flex flex-column mb-4">
      <span class="title">{{displayTitle(product)}}</span>
      <span class="cost">{{displayCost(product.cost, currentUAExchangeRate) | currency:'USD'}}</span>
    </div>
  </div>`,
})
export class AppComponent {
  public currentUAExchangeRate = 28.24;

  public constructor(
    public productService: ProductService,
  ) {
  }

  @Memorize()
  public displayTitle({ title, vendorCode }: Product): string {
    return `#${ vendorCode } ${ title }`;
  }

  @Memorize()
  public displayCost(cost: number, exchangeRate: number): number {
    return cost / 100 * exchangeRate;
  }

  @Memorize({ duration: 3000 })
  public someCapitalize(str: string): string {
    return `${str.charAt(0).toUpperCase()}${str.slice(1)}`;
  }
}
```

You can also use the memoization function directly in your code.

```ts
import { memorizeFn } from '@plus-minus/decorators';

const displayTitleMemo = memorizeFn(({ title, vendorCode }: Product): string => {
    return `#${ vendorCode } ${ title }`;
}, { 
  size: 5,
  duration: 3000,
});
```

You can also combine the `@Mixins` decorator with the `@Memorize` decorator.

```ts
class ProductMixin {
  @Memorize()
  public displayTitle({ title, vendorCode }: Product): string {
    return `#${ vendorCode } ${ title }`;
  }

  @Memorize()
  public displayCost(cost: number, exchangeRate: number): number {
    return cost / 100 * exchangeRate;
  }
}

@Component({/*...*/})
@Mixins([ProductMixin])
export class ProductsComponent implements ProductMixin {
  public displayTitle: (product: Product) => string;
  public displayCost: (cost: number, exchangeRate: number) => number;
}
```
----

#### RuntimeType

Decorator `@RuntimeType(...)` is intended for type checking at runtime. It can check primitives, functions and classes, but it cannot compare objects that have been created, for example, via an object literal.
Decorator `@T(...)` is necessary to mark the type of method parameters.

Decorator `@RuntimeType(RuntimeTypeOptions)` accepts an object with options.
* **level** - error output level. By default, this setting is RuntimeTypeLevel.Throw

```ts
interface RuntimeTypeOptions {
  level?: RuntimeTypeLevel;
}
```

With the `setGlobalRuntimeTypeOptions(RuntimeTypeOptions)` function, you can set options globally for the entire application.

```ts
setGlobalRuntimeTypeOptions({
  level: environment.production ? RuntimeTypeLevel.Warn : RuntimeTypeLevel.Error,
});
```

If you specify options at the method decorator level, then they will take precedence over global ones.

```ts
@Component(...)
export class SomeComponent {
  public constructor() {
    this.someMethod('23');
  }

  @RuntimeType({ level: RuntimeTypeLevel.Log })
  public someMethod(@T(Number) param?: any): void {
    // ...
  }
}
```
    > The parameter 'param' must be of type 'Number', but not the type 'string'.

#####More examples

```ts
class DataModel {
  // ...
}

class OtherClass  {
  // ...
}

class SecondClass extends DataModel {
  // ...
}

@Component(...)
export class SomeComponent {
  public constructor() {
    this.someMethod('23' as any, new SecondClass(), new OtherClass());
  }

  @RuntimeType()
  public someMethod(
    @T(Number) first: number,
    @T(DataModel) second: DataModel, 
    @T(DataModel)third: DataModel
  ): void {
    // ...
  }
}
// > The parameter 'first' must be of type 'Number', but not the type 'String'.
// > The parameter 'third' must be of type 'DataModel', but not the type 'OtherClass'.
```
    
----

#### OverrideProps
    Coming soon...
