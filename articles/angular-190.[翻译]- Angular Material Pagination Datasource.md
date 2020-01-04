# Angular Material Pagination Datasource

[原文链接](https://medium.com/angular-in-depth/angular-material-pagination-datasource-73080d3457fe)

[原作者:Nils Mehlhorn](https://medium.com/@n_mehlhorn?source=post_page-----73080d3457fe----------------------)

[译者:尊重](https://www.zhihu.com/people/yiji-yiben-ming/posts)

![logo](../assets/angular-190/1.png)

在本文中，我们将为 [Angular Material Library](https://material.angular.io/) 开发一个响应式 datasource。datasource 将复用于多个分页需求中，允许开发者基于每一个实例配置搜索和排序的输入。完整的 demo 在 [这个地址](https://stackblitz.com/edit/angular-paginated-material-datasource)。

[虽然使用 JavaScript 可以实现非常多的功能](https://nils-mehlhorn.de/posts/what-you-can-do-with-javascript-today)，但是大多数时候，我们使用 JavaScript 获取和展示数据。在 Angular 中，通常我们使用 HTTP 获取数据，使用各种不同的用户界面组件来展示数据（可能是列表，表格，树形结构或是其他需要的方式）。

Angular Material Library 提供了许多可以用于展示数据的组件，比如 [table component](https://material.angular.io/components/table/overview)。组件的创建者甚至预料到了'从数据展示层面控制数据获取的链接状态' 这种需求，因此为 Material 的开发者提供了名为 [DataSource](https://material.angular.io/components/table/overview#advanced-data-sources) 的概念。

> For most real-world applications, providing the table a DataSource instance will be the best way to manage data. The DataSource is meant to serve a place to encapsulate any sorting, filtering, pagination, and data retrieval logic specific to the application. — Angular Material Documentation

> 对于大多数现实世界的应用而言，向 table 组件提供一个 DataSource 实例可能是管理数据的最佳方案。DataSouce 作为一个中继器，处理应用中的任何排序，筛选，分页和数据获取逻辑都将封装于其中。- Angular Material 文档

通常来说，真实业务需要展示的数据量都很大，很难一次全部获取。开发者可以通过分页的方式切分和传递数据，用户也可以通过页面导航平顺地查看数据。这样的功能对于表现形式不同但是都需要展示数据的页面而言都是刚需，因此将相关的功能进行封装提供复用的机制就很明智了。

## Pagination and Sorting Datasource

让我们看一个 datasource 的实现：允许对数据进行排序，获取连续的页面数据。

首先，我们将 Material datasouce 稍微简化一下：

```typescript
import { DataSource } from '@angular/cdk/collections';

export interface SimpleDataSource<T> extends DataSource<T> {
  connect(): Observable<T[]>;
  disconnect(): void;
}
```

通常来说，`connect()` 和 `disconnect()` 方法会接受参数 [CollectionViewer](https://github.com/angular/components/blob/396154413538857811cb0c6bb71e4b4e26ecb320/src/cdk/collections/collection-viewer.ts)。然而，让展示数据的组件去决定展示哪一部分的数据不太明智。所以，官方的 [datasource for the Material table](https://github.com/angular/components/blob/396154413538857811cb0c6bb71e4b4e26ecb320/src/material/table/table-data-source.ts) 也移除了参数。

第二步，在独立的文件 `page.ts` 中定义可复用的分页数据接口。

```typescript
export interface Sort<T> {
  property: keyof T;
  order: 'asc' | 'desc';
}

export interface PageRequest<T> {
  page: number;
  size: number;
  sort?: Sort<T>;
}

export interface Page<T> {
  content: T[];
  totalElements: number;
  size: number;
  number: number;
}

export type PaginatedEndpoint<T> = (req: PageRequest<T>) => Observable<Page<T>>
```

泛型参数 `T` 永远指向将会处理的数据类型 - 后续例子中的 `User`。

接口 `Sort<T>` 定义了用于数据的排序规则（发送给服务端）。其可以通过 [the headers of a Material table](https://material.angular.io/components/table/overview#sorting) 或 [selection](https://material.angular.io/components/select/overview) 创建。

接口 `PageRequest<T>` 则作为最终传递给服务进行 HTTP 请求的参数。该服务则会接受以 `Page<T>` 为类型的响应结果。

`PaginatedEndpoint<T>` 则是一个接受 `PageRequest<T>` 作为参数并返回一个 RxJS 数据流的函数。 Observable 包含了相应的 `Page<T>`。

现在，我们就可以继承上述接口并实现我们的分页 `datasouce` 了，代码如下：

```typescript
export class PaginatedDataSource<T> implements SimpleDataSource<T> {
  private pageNumber = new Subject<number>();
  private sort = new Subject<Sort<T>>();

  public page$: Observable<Page<T>>;

  constructor(
    endpoint: PaginatedEndpoint<T>,
    initialSort: Sort<T>,
    size = 20) {
      this.page$ = this.sort.pipe(
        startWith(initialSort),
        switchMap(sort => this.pageNumber.pipe(
          startWith(0),
          switchMap(page => endpoint({page, sort, size}))
        )),
        share()
      )
  }

  sortBy(sort: Sort<T>): void {
    this.sort.next(sort);
  }

  fetch(page: number): void {
    this.pageNumber.next(page);
  }

  connect(): Observable<T[]> {
    return this.page$.pipe(pluck('content'));
  }

  disconnect(): void {}
}
```

让我们从构造器开始一步步讲解。

构造器接受了三个参数：

- 一个分页 endpoint 用于获取页面（作为函数）
- 一个初始化的排序参数
- 一个可选的每页大小参数，默认是每页20个

使用 RxJS subject 实例化了一个名为 `sort` 的属性。通过使用 RxJS subject，每当类函数 `sortBy(sort: Sort<T>)` 被调用时，都将触发排序规则的变化，而 `sortBy()` 函数仅仅是向 `sort` subject 传递了一个值。在构造器中还初始化了一个名为 `pageNumber` 的subject，这样就可以通过类函数 `fetch(page:number)` 通知 `datasouce` 获取不同页面的数据了。

datasouce 通过名为 `page$` 的参数向外暴露用于承载页面新的数据流。我们基于排序的变化构造了这个 Observable 数据流。而 RxJS 操作符 `startWith` 则用于提供一个初始排序规则。

当 datasource 需要去获取不同的页面时，实际上是 `fetch(page: number)` 函数被触发的过程：使用需要的参数调用 `paginated endpoint` 函数。最终，数据流可能会向多个消费组件提供新的页面数据。因此，我们需要使用 `share()` 操作符同步组件对 `page$` 的订阅。

最后，在 `connect()` 函数中，我们使用 `pluck()` 操作符完成对页面的内容映射，从而提供一个页面内容数据流。该函数将由 Material Table 或是其他兼容 `DataSource` 接口的组件所调用。你可能会好奇，为什么我们不将页面与其内容直接映射 - 因为我们还需要其他页面属性 `size` 和 `number` ，这些属性则是为为 [MatPaginator](https://material.angular.io/components/paginator/overview) 服务的。

`disconnect()` 函数不会进行任何操作，当所有消费的组件取消订阅后，我们的 `datasouce` 将会自动关闭。

## 在组件中使用 DataSource

现在，我们就可以配合 Material Table 使用我们的 `datasouce` 对组件中的数据进行处理了。

创建一个实例并传递参数，用于将页面请求引导到对应的服务中。在组件中，使用一个默认的排序规则。

`UserService` 负责将 `PageRequest<User>` 转化为一个合适的 HTTP 请求。

```typescript
@Component(...)
export class UsersComponent  {
    displayedColumns = ['id', 'name', 'email', 'registration']

    data = new PaginatedDataSource<User>(
      request => this.users.page(request),
      {property: 'username', order: 'desc'}
    )

    constructor(private users: UserService) {}
}
```

此时，一旦用户选择了一个新的排序规则，就需要使用 `data.sortBy(sort)` 函数改变页面的排序规则数据。

在模板文件中将 `datasource` 传递给 Material table。模板中还需要定义一个 `MatPaginator` 才可以允许用户切换页面。分页功能则可以通过 `AsyncPipe` 消费 `datasouce` 的页面数据流（通过调用 `data.fetch(page:number)`） 以获取不同的页面数据信息。

```html
<table mat-table [dataSource]="data">
  <ng-container matColumnDef="name">
    <th mat-header-cell *matHeaderCellDef>Username</th>
    <td mat-cell *matCellDef="let user">{{user.username}}</td>
  </ng-container>
  ...
</table>
<mat-paginator *ngIf="data.page$ | async as page"
  [length]="page.totalElements" [pageSize]="page.size"
  [pageIndex]="page.number" [hidePageSize]="true" 
  (page)="data.fetch($event.pageIndex)">
</mat-paginator>
```

## 添加查询参数

对于用户而言，快速查询到自己想找的东西是很常见的需求。所以作为开发者就需要提供一个文本搜索或者一个结构化输入帮助用户通过某些属性筛选数据。查询的参数基于用户查询的内容进行变化。

为了将这个普遍的需求融合到 `datasource` 中，我们需要添加一个查询参数泛型集。

首先向 `datasouce` 的类型中添加一个泛型参数 `Q` (代表查询的数据模型)，并将查询参数添加到 `PaginatedDataSource<T, Q>` 中。

之后，向构造器参数中加入一个初始化的查询参数，并创建一个 `BehaviorSubject` `query` - `this.query = new BehaviourSubject<Q>(initalQuery)`。利用 `BehaviorSubject` 的特性，我们可以获取 `query` 参数的上一个值；这样就可以通过下述实例函数部分更新查询参数了：

```typescript
queryBy(query: Partial<Q>): void {
    const lastQuery = this.query.getValue();
    const nextQuery = {...lastQuery, ...query};
    this.query.next(nextQuery);
}
```

该实例函数接受一个查询参数模型的 [partial representation](https://www.typescriptlang.org/docs/handbook/utility-types.html#partialt)。将 `BehaviorSubject<Q>` 的上一个值与新的查询参数合并（使用[ spread operator](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Operators/Spread_syntax)）。通过这样的方式，当只有一个参数被更新时，老的查询属性就不会被覆写了。

现在，页面信息数据流除了依托于排序 `subject` 之外，还将与 查询 `BehaviorSubject` 通过 `combineLatest()` 操作符进行合并。查询和排序的数据流都有拥有其初始值：`sort` 通过 `startWith()` 获取初始值，`query` 则通过 `BehaviorSubject` 的构造参数获取初始值。

```typescript
const param$ = combineLatest([
    this.query, 
    this.sort.pipe(startWith(initialSort))
]);
this.page$ = param$.pipe(
    switchMap(([query, sort]) => this.pageNumber.pipe(
      startWith(0),
      switchMap(page => endpoint({page, sort, size}, query))
    )),
    share()
)
```

接下来，我们还需要将查询信息传递给 `paginated endpoint`。操作如下：

```typescript
export type PaginatedEndpoint<T, Q> = 
        (req: PageRequest<T>, query: Q) => Observable<Page<T>>
```

现在，组件就可以支持额外的查询输入信息了，只是还需要一点调整。

首先，使用 `UserQuery` 调整 `PaginatedDataSource<T, Q>` 的初始化。

其次，使用 `paginated endpoint` 用于将页面请求和查询参数发送给 `UserService`。

最后，传递初始查询值。

```typescript
interface UserQuery {
  search: string
  registration: Date
}
```

在本案例中，除了允许用户通过文本输入提供查询内容，也支持通过日期选择器基于用户的注册时间信息进行查询：

```typescript
data = new PaginatedDataSource<User, UserQuery>(
    (request, query) => this.users.page(request, query),
    {property: 'username', order: 'desc'},
    {search: '', registration: undefined}
)
```

而在模板文件中，则只需要调用 `data.queryBy()` 函数将输入数据传递给 datasouce（该函数使用包含查询参数的部分查询模型）

```html
<mat-form-field>
    <mat-icon matPrefix>search</mat-icon>
    <input #in (input)="data.queryBy({search: in.value})" type="text" matInput placeholder="Search">
</mat-form-field>
<mat-form-field>
    <input (dateChange)="data.queryBy({registration: $event.value})" matInput [matDatepicker]="picker" placeholder="Registration"/>
    <mat-datepicker-toggle matSuffix [for]="picker"></mat-datepicker-toggle>
    <mat-datepicker #picker></mat-datepicker>
</mat-form-field>
<table mat-table [dataSource]="data">
  ...
</table>
```

现在，只要输入内容发生了变化，页面的展示数据就会自动变化，前提是正确地把查询参数发送到服务端，并在那里正确地处理它们。

## Loading indication

如果你想在获取数据时，向你的用户发出提示，可以使用一个对应的私有 `subject` 继承 `PaginatedDataSource<T, Q> `:

```typescript
private loading = new Subject<boolean>();

public loading$ = this.loading.asObservable();
```

之后，你就可以在调用 `PaginatedEndpoint<T, Q>` 之前和之后手动更新 `subject` 的值了，或者你也可以使用我的独创的 `indicate(indicator: Subject<boolean>)` 操作符实现同样的功能（如果你对这个操作符感兴趣，可以阅读这篇文章 [loading indication in Angular](https://nils-mehlhorn.de/posts/indicating-loading-the-right-way-in-angular)），你只需要将其附加在 `paginated endpoint` 所返回的 Observable 上即可：

```typescript
this.page$ = param$.pipe(
    switchMap(([query, sort]) => this.pageNumber.pipe(
      startWith(0),
      switchMap(page => this.endpoint({page, sort, size}, query)
        .pipe(indicate(this.loading))
      )
    )),
    share()
)
```

使用如下方式展示 `loading indicator`：

```html
<my-loading-indicator *ngIf="data.loading$ | async">
</my-loading-indicator>
```

## 总结

通过行为参数化方案，我们可以轻松地将大段逻辑进行复用，并可写出适用于任何数据的强健可配置组件。我们对 Material datasource 实现的拓展功能可以帮助开发者仅通过几行代码就实现对远程数据的分页，排序，筛选功能。

你可以在 [StackBlitz](https://angular-paginated-material-datasource.stackblitz.io) 上找到完整的案例，希望你阅读愉快。