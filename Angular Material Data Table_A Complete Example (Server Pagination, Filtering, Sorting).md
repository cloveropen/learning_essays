# Angular Material Data Table: A Complete Example (Server Pagination, Filtering, Sorting)
  翻译自 https://blog.angular-university.io/angular-material-data-table/
  
  在本篇文章中，我们将通过一个完整的例子来说明如何使用Angular Material DataTable。
  我们将介绍围绕Angular Material数据表组件的许多最常见的应用，例如：服务器端分页、排序和过滤。
  这是一个循序渐进的教程，所以邀请您一起编写代码，因为我们将从一个简单的初始场景开始。然后，我们将逐步地逐个添加特性，并解释逐步的每个特性（包括gotchas）。
  我们将详细了解在Angular Material数据表和Angular Material CDK数据源设计中涉及的所有反应性设计原则。
  这篇文章的最终目标是：
  + 一个完整的例子，说明如何使用自定义的CDK数据源实现服务器端分页、排序和过滤的角度材料数据表
  + Github上提供的一个运行示例，其中包括一个小型后端Express服务器，用于为分页数据提供服务

## 目录:
  我们将介绍以下主题：
  + Angular Material Data Table -不仅用于Material Design
  + Material Data Table 反应式设计
  + Material分页器和服务器端分页
  + 可排序表头和服务器端排序
  + 使用Material输入框进行服务器端过滤
  + 加载进度指示器
  + 完整示例的源代码（在Github上）
  + 结论
  
  立即行动起来开始我们的Material数据表学习！
  
## 导入Angular Material模块
  为了运行我们的示例，让我们首先导入我们需要的所有Angular Material模块：
  ```
  import { MatInputModule, MatPaginatorModule, MatProgressSpinnerModule, 
         MatSortModule, MatTableModule } from "@angular/material";

@NgModule({
    declarations: [
        ... 
    ],
    imports: [
        BrowserModule,
        BrowserAnimationsModule,
        HttpClientModule,
        MatInputModule,
        MatTableModule,
        MatPaginatorModule,
        MatSortModule,
        MatProgressSpinnerModule
    ],
    providers: [
      ...
    ],
    bootstrap: [AppComponent]
})
export class AppModule {

}

  ```
  每个Angular Material模块的内容详细介绍：
  + MatInputModule:其中包含用于将Material设计输入框添加到我们的应用程序的组件和指令（需要搜索输入框）
  + MatTableModule:这是核心数据表模块，其中包括mat-table组件以及许多相关组件和指令
  + MatPaginatorModule:这是一个通用分页模块，通常可用于分页数据。 该模块也可以与数据表分开使用，例如，用于在主从设置中实现细节分页逻辑
  + MatSortModule:这是一个可选模块，允许将可排序的标头添加到数据表
  + MatProgressSpinnerModule: 该模块包括进度指示器组件，我们将使用它来指示正在从后端加载数据
  
## Angular Material数据表简介
  Angular Material数据表组件是用于显示列表数据的通用组件。 尽管我们可以轻松地赋予它“Material Design”的外观和感觉，但这实际上不是必须要使用的。
  实际上，如果需要，我们可以为Angular Material Data表提供替代的UI设计。 为了看到这种情况，让我们开始创建一个数据表，其中的表单元格只是普通的div，没有应用自定义CSS。
  该数据表将显示课程列表，并包含3列（序列号，描述和持续时间）
  ```
  <mat-table class="lessons-table mat-elevation-z8" [dataSource]="dataSource">
  
    <ng-container matColumnDef="seqNo">
        <div *matHeaderCellDef>#</div>
        <div *matCellDef="let lesson">{{lesson.seqNo}}</div>
    </ng-container>
  
    <ng-container matColumnDef="description">
        <div *matHeaderCellDef>Description</div>
        <div class="description-cell"
                  *matCellDef="let lesson">{{lesson.description}}</div>
    </ng-container>
  
    <ng-container matColumnDef="duration">
        <div *matHeaderCellDef>Duration</div>
        <div class="duration-cell"
                  *matCellDef="let lesson">{{lesson.duration}}</div>
    </ng-container>
    
    <mat-header-row *matHeaderRowDef="displayedColumns"></mat-header-row>
    
    <mat-row *matRowDef="let row; columns: displayedColumns"></mat-row>
    
</mat-table>
  ```
## Material Data Table列定义
  如我们所见，该表定义了3列，每列都在其自己的ng-container元素内。 ng-container元素将不会呈现到屏幕上，但是它将提供一个用于应用matColumnDef指令的元素。
  matColumnDef指令使用键seqNo,description或duration唯一地标识给定的列。 在ng-container元素内设置给定列的所有配置。
  请注意，ng-container元素的顺序不会确定列的视觉顺序  
## Material数据表辅助定义指令
  Material数据表具有一系列辅助结构指令（使用*directiveName语法应用），它们使我们能够标记某些模板节在整体数据表设计中具有特定作用。
  这些伪指令始终以Def后缀结尾，用于将角色分配给模板部分。 我们将介绍的前两个指令是matHeaderCellDef和matCellDef。
## matHeaderCellDef 和 matCellDef 指令
  在具有给定列定义的每个ng容器内部，有几个配置元素：
  + 我们定义如何显示给定列标题的模板，该模板通过matHeaderCellDef结构指令标识
  + 我们还有另一个模板，定义了如何使用matCellDef结构指令显示给定列的数据单元格

  这两个结构指令仅标识哪些模板元素具有给定的角色(单元模板，表头模板)，但它们不对这些元素附加任何样式。
  例如，在这种情况下，可以将matCellDef和matHeaderCellDef应用于没有样式的普通div，这就是为什么此表还没有Material设计的原因。
## 将Material Design应用于数据表
  现在让我们来看一下如何为该数据表提供Material外观。为此，我们将在表头和单元格模板定义中使用几个内置组件：
  ```
  <mat-table class="lessons-table mat-elevation-z8" [dataSource]="dataSource">

    <ng-container matColumnDef="seqNo">
        <mat-header-cell *matHeaderCellDef>#</mat-header-cell>
        <mat-cell *matCellDef="let lesson">{{lesson.seqNo}}</mat-cell>
    </ng-container>

    <ng-container matColumnDef="description">
        <mat-header-cell *matHeaderCellDef>Description</mat-header-cell>
        <mat-cell class="description-cell"
                  *matCellDef="let lesson">{{lesson.description}}</mat-cell>

    </ng-container>

    <ng-container matColumnDef="duration">
        <mat-header-cell *matHeaderCellDef>Duration</mat-header-cell>
        <mat-cell class="duration-cell"
                  *matCellDef="let lesson">{{lesson.duration}}</mat-cell>
    </ng-container>

    <mat-header-row *matHeaderRowDef="displayedColumns"></mat-header-row>

    <mat-row *matRowDef="let row; columns: displayedColumns"></mat-row>

</mat-table>
  ```
  
  该模板与我们之前看到的模板几乎相同，但是现在我们在列定义中使用了mat-header-cell和mat-cell组件，而不是普通的div。
  使用这些组件，现在让我们看一下新Material Design的数据表的外观：
  ![avatar](https://s3-us-west-1.amazonaws.com/angular-university/blog-images/angular-material-data-table/1-material-data-table.png)
  请注意，该表已经有一些数据！ 稍后我们将介绍数据源，现在让我们继续探索模板的其余部分。
## matCellDef 指令
  数据单元格模板可以访问正在显示的数据。在这种情况下，我们的数据表将显示课程列表，因此每一行的课程对象都可以通过let课程语法进行访问，并且可以像使用任何组件变量一样在模板中使用。
## mat-header-row组件和matHeaderRowDef指令
  此组合组件/指令的这种组合以以下方式工作：
  + matHeaderRowDef标识表标题行的配置元素，但不会对该元素应用任何样式
  + mat-header-row应用一些基本的Material 样式
  matHeaderRowDef指令还定义了显示列的顺序。 在我们的例子中，指令表达式指向一个名为displayColumns的组件变量。
  这是displayColumns组件变量的样子：
  ```
  displayedColumns = ["seqNo", "description", "duration"];
  ```
  该数组的值是列的主键，必须与ng-container列节点定义的名称相同（通过matColumnDef指令指定）。
  注意：此数组确定列的显示顺序 
## mat-row组件和matRowDef指令
  该组件/指令对的工作方式也与之前的例子类似：
  + matRowDef标识mat-table中的哪个元素为数据行的外观提供配置，而不提供任何特定样式
  + mat-row将为数据行提供一些Material样式
  使用mat-row，我们还导出了一个已命名为row的变量，其中包含给定数据行的数据，并且我们必须指定columns属性，该属性包含应定义数据单元格的顺序。
## 与给定表数据行进行交互
  我们甚至可以使用matRowDef指令标识的元素与给定的数据行进行交互。 例如，这是我们可以检测是否单击了给定数据行的方法：
  ```
  <mat-row *matRowDef="let row; columns: displayedColumns" 
         (click)="onRowClicked(row)">
  </mat-row>
  ```
  单击一行后，我们将调用onRowClicked（）组件方法，该方法将把行数据记录到控制台：
  ```
  onRowClicked(row) {
    console.log('Row clicked: ', row);
  }
  ```
  如果现在单击数据表的第一行，则结果将在控制台上显示如下：
  ![avatar](https://s3-us-west-1.amazonaws.com/angular-university/blog-images/angular-material-data-table/2-material-data-table.png)
  如我们所见，第一行的数据已按预期打印到控制台上！ 但是这些数据从哪里来？
  为了回答这个问题，让我们接下来讨论链接到该数据表的数据源，然后讨论“Material数据表”反应式设计。
## 数据源和数据表反应式设计
  我们一直在呈现的数据表从数据源接收需要显示的数据，该数据源实现了基于Observable的API并遵循常见的反应式设计原则。
  例如数据表组件不知道数据来自何处。 数据可能来自后端，也可能来自客户端缓存，但这对数据表是透明的。
  数据表仅预订数据源提供的Observable。 当该Observable发出新值时，它将包含课程列表，然后将其显示在数据表中。

## 数据表核心设计原则
  使用此基于Observable的API，不仅数据表不知道数据来自何处，而且数据表还不知道是什么触发了新数据的到达。
  以下是触发新数据的一些可能的条件：
  + 最开始显示数据表
  + 用户单击分页按钮
  + 用户通过单击可排序标题来对数据进行排序
  + 用户使用输入框输入搜索
  同样，数据表没有明确通知是什么事件导致新数据到达，这使得数据表组件和指令仅专注于显示数据，而不是获取数据。
  接下来让我们看看如何实现这种反应性数据源。  
## 为什么不使用MatTableDataSource？  
  在此示例中，我们将不使用内置的MatTableDataSource，因为它是为过滤，排序和分页客户端数据数组而设计的。
  在我们的案例中，所有的过滤，排序和分页都将在服务器上进行，我们将根据第一个原则构建自己的Angular CDK反应性数据源。
## 从后端获取数据
  为了从后端获取数据，我们的自定义数据源将使用LessonsService。 这是使用Angular HTTP客户端在内部构建的标准的基于可观察的无状态单例服务。
  让我们看一下该服务，并分析其实现方式：   
  ```
  @Injectable()
export class CoursesService {

    constructor(private http:HttpClient) {}

    findLessons(
        courseId:number, filter = '', sortOrder = 'asc',
        pageNumber = 0, pageSize = 3):  Observable<Lesson[]> {

        return this.http.get('/api/lessons', {
            params: new HttpParams()
                .set('courseId', courseId.toString())
                .set('filter', filter)
                .set('sortOrder', sortOrder)
                .set('pageNumber', pageNumber.toString())
                .set('pageSize', pageSize.toString())
        }).pipe(
            map(res =>  res["payload"])
        );
    }
}
      
  ```
## 分析LessonsService的实现
  我们看到，该服务是完全无状态的，每个方法都使用HTTP客户端将调用转发到后端，然后将Observable返回给调用者。
  我们的REST API在/api目录下的URL中可用，并且可以使用多种服务（此处https://github.com/angular-university/angular-material-course/blob/2-data-table-finished/src/app/services/courses.service.ts是完整的实现）。
  在此代码段中，我们仅显示findLessons()方法，该方法允许为给定课程获取一个经过过滤和排序的课程数据页面。
  这是我们可以传递给此函数的参数：
  + courseId:确定了指定的课程，我们要为其检索课程页面
  + filter: 这是一个搜索字符串，可以帮助我们过滤结果。 如果我们传递空字符串''，则表示服务器上未进行任何过滤
  + sortOrder:我们的后端允许我们基于seqNo列进行排序，并使用此参数，我们可以指定排序顺序是升序(这是默认的asc值)，还是通过传递值desc降序
  + pageNumber:在对结果进行过滤和排序后，我们将指定所需结果的完整列表中的哪一页。 默认为返回第一页(索引为0)
  + pageSize:这指定页面大小，默认为最多3个元素

  使用这些此参数，然后loadLessons()方法将对/api/lessons上可用的后端端点建立一个HTTP GET调用。
  这是获取第一页课程的HTTP GET调用的样子：
  http://localhost:4200/api/lessons?courseId=1&filter=&sortOrder=asc&pageNumber=0&pageSize=3
  如我们所见，我们正在使用HTTP Params fluent API将一系列HTTP查询参数附加到GET URL。
  这个loadLessons()方法将成为我们数据源的基础，因为它将使我们能够涵盖服务器分页，排序和过滤用例。
## 实现自定义Angular CDK数据源
  现在使用LessonsService实现自定义的基于Observable的Angular CDK数据源。下面是一些初始代码，以便我们可以讨论其Reactive设计（完整版本将在稍后显示）：    
  ```
  import {CollectionViewer, DataSource} from "@angular/cdk/collections";

export class LessonsDataSource implements DataSource<Lesson> {

    private lessonsSubject = new BehaviorSubject<Lesson[]>([]);

    constructor(private coursesService: CoursesService) {}

    connect(collectionViewer: CollectionViewer): Observable<Lesson[]> {
      ...
    }

    disconnect(collectionViewer: CollectionViewer): void {
      ...
    }
  
    loadLessons(courseId: number, filter: string,
                sortDirection: string, pageIndex: number, pageSize: number) {
      ...
    }  
}
  
  ```
## 分析Angular CDK数据源的设计
  我们已经看到，为了创建数据源需要创建一个实现DataSource的类。这意味着该类需要实现几种方法：connect()和disconnect()。
  请注意，这些方法提供的参数是CollectionViewer，该参数提供了Observable，该Observable发出有关正在显示哪些数据(起始索引和结束索引)的信息。
  我们建议现在暂时不要将重点放在CollectionViewer上，而应将重点放在对理解整个设计更重要的事情上：connect()方法的返回值。
  
## 如何实现DataSource connect()方法
  表启动时数据表将一次调用此方法。 数据表希望此方法返回一个Observable，并且该Observable的值包含Data Table需要显示的数据。
  在这个例子里，此可观察对象将发出“课程”列表。 当用户单击分页器并切换到新页面时，该可观察对象将在新课程页面中发出新值。
  我们将通过订阅在此类之外不可见的主题来实现此方法。 该主题(lessonsSubject)将发出从后端检索的值。
  lessonsSubject是一个BehaviorSubject，这意味着它的订阅者将始终获得其最新发出的值(或初始值)，即使他们订阅得晚(在发出该值之后)。这个是RxJS里面的冷订阅。
## 为什么使用BehaviorSubject？
  使用BehaviorSubject是一种很好的编写代码方式，该代码的工作方式与我们执行异步操作的顺序无关，例如：调用后端，将数据表绑定到数据源等。
  例如，在这种设计中，数据源不知道数据表，或者数据表在何时需要数据。 因为数据表已订阅了connect（）可观察的数据，所以即使发生以下情况，它也最终将获得数据：
  + 数据自HTTP后端仍在传输中
  + 或者已经加载了数据
  BehaviorSubject每次加载都能拿到上次推送的最新的值，而不是没有返回。

## 自定义Material CDK数据源-实现方式的全面总结
  现在我们已经了解数据源的响应式设计，下面让我们看一下完整的最终实现并逐步进行回顾。
  请注意，在此最终实现中，我们还包括了加载标志的概念，该标志将在以后用于向用户显示旋转的加载指示器：  
  ```
  export class LessonsDataSource implements DataSource<Lesson> {

    private lessonsSubject = new BehaviorSubject<Lesson[]>([]);
    private loadingSubject = new BehaviorSubject<boolean>(false);

    public loading$ = this.loadingSubject.asObservable();

    constructor(private coursesService: CoursesService) {}

    connect(collectionViewer: CollectionViewer): Observable<Lesson[]> {
        return this.lessonsSubject.asObservable();
    }

    disconnect(collectionViewer: CollectionViewer): void {
        this.lessonsSubject.complete();
        this.loadingSubject.complete();
    }

    loadLessons(courseId: number, filter = '',
                sortDirection = 'asc', pageIndex = 0, pageSize = 3) {

        this.loadingSubject.next(true);

        this.coursesService.findLessons(courseId, filter, sortDirection,
            pageIndex, pageSize).pipe(
            catchError(() => of([])),
            finalize(() => this.loadingSubject.next(false))
        )
        .subscribe(lessons => this.lessonsSubject.next(lessons));
    }    
}

  ```
## 数据源加载标记实现方式分析
  我们开始分析此代码，我们将从加载指示器的实现开始。 因为此Data Source类具有反应性设计，所以让我们通过暴露一个称为load$的布尔可观察对象来实现加载标志。
  此可观察对象将作为第一个值false(在BehaviorSubject构造函数中定义)发出，这意味着最初没有数据加载。
  loading$ observable是使用asObservable()从对数据源类保持私有的主题派生的,只有此类才知道何时加载数据，因此只有此类可以访问主题并为加载标志发出新值。 

## Connect()的实现方法
  现在让我们讨论connect方法的实现：  
  ```
  connect(collectionViewer: CollectionViewer): Observable<Lesson[]> {
    return this.lessonsSubject.asObservable();
}
```
  此方法将需要返回一个Observable，该Observable发出课程数据，但是我们不想直接公开内部主题lessonsSubject。
  公开该主题意味着要控制何时由数据源发出什么数据以及什么数据，我们希望避免这种情况。 我们要确保只有此类可以发出课程数据的值。
  因此，我们还将使用asObservable（）方法返回从lessonsSubject派生的Observable。 这使数据表（或任何其他订户）具有订阅可观察的课程数据的能力，而无法发出该可观察的值。

## disconnect() 的实现方法
  现在让我们分解disconnect方法的实现：
  ```
disconnect(collectionViewer: CollectionViewer): void {
    this.lessonsSubject.complete();
    this.loadingSubject.complete();
}
```
  数据表在组件销毁时一次调用此方法。 在这种方法中，我们将完成在此类内部创建的所有可观察变量，以避免内存泄漏。
  我们将同时完成lesssonSubject和loadingSubject，这将触发所有派生的可观察对象的完成。

## loadLessons()方法的实现分析
  最后，让我们讨论loadLessons方法的实现：
```

loadLessons(courseId: number, filter = '',
            sortDirection = 'asc', pageIndex = 0, pageSize = 3) {

    this.loadingSubject.next(true);

    this.coursesService.findLessons(courseId, filter, sortDirection,
        pageIndex, pageSize).pipe(
        catchError(() => of([])),
        finalize(() => this.loadingSubject.next(false))
    )
    .subscribe(lessons => this.lessonsSubject.next(lessons));
}

```
  数据源公开了这个名为loadLessons()的公共方法。 将响应多个用户操作(分页，排序，过滤)以加载给定的数据页而调用此方法。
  这个方法的工作原理如下：
  + 首先通过向loadingSubject发出true来报告正在加载某些数据，这将导致loading$也发出true
  + LessonsService将从REST后端获取数据页
  + 产生一个findLessons()调用，返回一个Observable
  + 通过订阅该observable，我们触发了HTTP请求
  + 如果数据是从后端成功到达的，我们将通过connect()将其发送回数据表
  .为此，我们将使用课程数据在 lessonsSubject 上调用next（）
  .connect（）返回的可观察到的派生课程然后将课程数据发送到数据表
## 处理后端错误
  现在，仍然在loadLessons()方法中，看看数据源如何处理后端错误以及如何管理加载指示器：
  + 如果HTTP请求中发生错误，则findLessons（）返回的Observable将出错
  + 如果发生这种情况，我们将使用catchError（）捕获该错误，并返回一个Observable，它使用发出空数组
  + 我们可以补充地使用另一个MessagesService向用户显示可关闭的错误弹出窗口
  + 不管对后端的调用是成功还是失败，我们都将通过使用finalize（）在两种情况下使loading $ Observable发出false（其工作方式类似于在普通Java语言try / catch / finally中的finally）
  最后，我们完成了对自定义数据源的审查！
  此版本的数据源将支持我们所有的用例：分页，排序和过滤。 如我们所见，设计的全部目的是使用基于Observable的API向数据表透明地提供数据。
  现在，让我们看看如何获取该数据源并将其插入数据表。

## 将数据源与数据表链接
  数据表将显示为组件模板的一部分,让我们编写该组件的初始版本，以显示课程的第一页：
```
@Component({
    selector: 'course',
    templateUrl: './course.component.html',
    styleUrls: ['./course.component.css']
})
export class CourseComponent implements OnInit {

    dataSource: LessonsDataSource;
    displayedColumns= ["seqNo", "description", "duration"];

    constructor(private coursesService: CoursesService) {}

    ngOnInit() {
        this.dataSource = new LessonsDataSource(this.coursesService);
        this.dataSource.loadLessons(1);
    }
}

```
  此组件包含几个属性：
  + displayColumns数组定义列的显示顺序
  + dataSource属性定义了LessonsDataSource的实例，该实例通过模板传递给mat-table
  
## 分析ngOnInit方法
  在ngOnInit方法中，我们正在调用数据源loadLessons()方法来触发第一个课程页面的加载。 让我们详细说明该调用的结果：
  + 数据源调用LessonsService，该服务会触发HTTP请求以获取数据
  + 然后，数据源通过lessonsSubject发出数据，这导致connect（）返回的Observable发出Lessons页面
  + mat-table数据表组件已订阅了可观察的connect（）并检索了新的课程页面
  + 然后，数据表将显示新的课程页面，而不知道数据来自何处或触发数据到达的原因
  有了这个“胶水”组件，我们现在有了一个工作数据表，可以显示服务器数据！
  现在的问题是这个初始示例总是只加载数据的第一页，页面大小为3，没有搜索条件。
    
  让我们以这个例子为起点，开始添加：加载指示器、分页、排序和筛选。  
## 显示Material 加载进程指示器
  为了显示加载进度指示器，我们将使用数据源的loading$observable。我们将使用mat-spinner Material组件：
```

<div class="course">

    <div class="spinner-container" *ngIf="dataSource.loading$ | async">
        <mat-spinner></mat-spinner>
    </div>

    <mat-table class="lessons-table mat-elevation-z8" [dataSource]="dataSource">
        ....
    </mat-table>
</div>

```  
  可以看到，我们使用async管道和ngIf来显示或隐藏material进度加载器，下面是数据正在加载时的样子:
  ![avatar](https://s3-us-west-1.amazonaws.com/angular-university/blog-images/angular-material-data-table/3-material-data-table.png)
  当使用分页，排序或过滤在两个数据页之间转换时，我们还将使用加载指示器。
## 添加Material数据表分页器
  我们将使用的Material Paginator组件是基于Observable的API附带的通用paginator。 该分页器可用于对任何内容进行分页，并且未专门链接到数据表。
  例如，在Master-Detail组件设置中，我们可以使用此分页器在两个Detail元素之间导航。
  在模板中使用mat-paginator组件的方式： 
  ```
<div class="course">

    <div class="spinner-container" *ngIf="dataSource.loading$ | async">
        <mat-spinner></mat-spinner>
    </div>

    <mat-table class="lessons-table mat-elevation-z8" [dataSource]="dataSource">
        ....
    </mat-table>

    <mat-paginator [length]="course?.lessonsCount" [pageSize]="3"
                   [pageSizeOptions]="[3, 5, 10]"></mat-paginator>

</div>
   ```  
  可以看到模板中没有将分页器与数据源或数据表链接的内容-连接将在CourseComponent级别完成。
  分页器只需要知道要对多少个项目进行分页（通过length属性），就可以知道有多少个页面！
  它基于分页器将启用或禁用导航按钮的信息（加上当前页面索引）。
  为了将信息传递给分页器，我们使用了新课程对象的lessonsCount属性。  

## 如何将Material分页器链接到数据源
  现在，让我们看一下CourseComponent，看看课程来自何处以及分页器如何链接到数据源：
  ```
  
@Component({
    selector: 'course',
    templateUrl: './course.component.html',
    styleUrls: ['./course.component.css']
})
export class CourseComponent implements AfterViewInit, OnInit {

    course:Course;
    dataSource: LessonsDataSource;
    displayedColumns= ["seqNo", "description", "duration"];

    @ViewChild(MatPaginator) paginator: MatPaginator;

    constructor(private coursesService: CoursesService, private route: ActivatedRoute) {}

    ngOnInit() {
        this.course = this.route.snapshot.data["course"];
        this.dataSource = new LessonsDataSource(this.coursesService);
        this.dataSource.loadLessons(this.course.id, '', 'asc', 0, 3);
    }    
    
    ngAfterViewInit() {
        this.paginator.page
            .pipe(
                tap(() => this.loadLessonsPage())
            )
            .subscribe();
    }

    loadLessonsPage() {
        this.dataSource.loadLessons(
            this.course.id,
            '',
            'asc',
            this.paginator.pageIndex,
            this.paginator.pageSize);
    }
}

  ```
## 分析ngOnInit()方法
  让我们从课程对象开始：正如我们所见，该对象在组件构建时可以通过路由器使用。
  使用路由器数据解析器在路由器导航时从后端检索了此数据对象（请参见此处https://github.com/angular-university/angular-material-course/blob/016186af48c2deb7f27fa392121a2aca67998c4f/src/app/services/courses.resolver.ts的示例）。
  这是一个非常常见的设计，可确保目标导航屏幕已经具有一些准备显示的预取数据。
  我们还使用这种方法直接加载第一页数据（第20行）。
## 分页器如何链接到数据源？
  我们可以在上面的代码中看到，分页器和数据源之间的链接是在ngAfterViewInit（）方法中完成的，因此让我们对其进行分析：
```
ngAfterViewInit() {
    this.paginator.page
        .pipe(
            tap(() => this.loadLessonsPage())
        )
        .subscribe();
}

```  
  我们正在使用AfterViewInit生命周期挂钩，因为我们需要确保通过@ViewChild查询的分页器组件已经可用。
  分页器还具有基于Observable的API，并公开了一个页面Observable。 每当用户单击分页器导航按钮或页面大小下拉菜单时，此可观察值将发出一个新值。
  因此，为了响应分页事件而加载新页面，我们要做的就是订阅此可观察的事件，并且响应分页事件，我们将调用数据源loadLessons（）方法， 通过调用loadLessonsPage（）。
  在对loadLessons（）的调用中，我们将向数据源传递要加载的页面索引，页面大小以及该信息直接从分页器获取。

## 为什么使用tap()操作符？
  我们也可以从subscribe（）处理程序中完成对数据源的调用，但是在这种情况下，我们使用称为pipeable 的RxJs do运算符的可移植版本实现了该调用。
  rxjs v6+中的do/tap运算符或tap运算符与所有其他运算符的不同之处在于，它不会以任何方式修改通过它的项。传递函数的结果不被考虑用于进一步处理（返回类型为void）。
这使得执行具有“副作用”的代码更安全，即修改可观察管道外部状态的代码。一个典型的例子是在console.log函数中使用tap，因为调用该函数会运行改变浏览器状态的代码。
此外，如果您确实需要在subscribe()之前更改变量或属性，您还应该在tap函数中执行此操作。否则，您可能会破坏可观察管道的纯净度（纯净度意味着，对于相同的输入，您总是得到相同的输出）。
小心更改发送给tap操作符的项，因为这将更改其他管道的行为并更改最终结果（从而再次破坏纯净度）。
## 查看实际运行中的分页器
  我们现在有了一个可以工作的Material分页器！ 这是显示课程列表第2页时，Material分页器在屏幕上的外观：
  ![avatar](https://s3-us-west-1.amazonaws.com/angular-university/blog-images/angular-material-data-table/5-material-data-table.png)
  现在，让我们继续为示例添加更多功能，让我们添加另一个非常常用的功能：可排序的表头。

## 添加可排序的Material标题
  为了将可排序的标题添加到我们的数据表中，我们将需要使用matSort指令对其进行注释。 在这种情况下，我们将使表中只有一列可排序，即seqNo列。
  这是带有所有多个与排序相关的指令的模板的样子：
  ```
  
<mat-table class="lessons-table mat-elevation-z8" [dataSource]="dataSource"
           matSort matSortActive="seqNo" matSortDirection="asc" matSortDisableClear>

    <ng-container matColumnDef="seqNo">
        <mat-header-cell *matHeaderCellDef mat-sort-header>#</mat-header-cell>
        <mat-cell *matCellDef="let lesson">{{lesson.seqNo}}</mat-cell>
    </ng-container>

    ....

</mat-table>

  ```
  除了matSort指令之外，我们还向mat-table组件添加了两个额外的与排序相关的辅助指令：
  + matSortActive：当数据传递到数据表时，通常已经对其进行了排序。 该指令使我们可以通知数据表，数据已经通过seqNo列进行了初始排序，因此seqNo列的排序图标将显示为向上箭头
  + matSortDirection：这是matSortActive的伴随指令，它指定初始排序的方向。 在这种情况下，数据最初由seqNo列按升序排序，因此列标题将相应地适应排序图标（下面的屏幕截图）
  + matSortDisableClear：有时，除了升序和降序之外，我们可能还需要可排序列标题的第三个“未排序”状态，以便我们可以清除排序顺序。 在这种情况下，我们要禁用它，以确保seqNo列始终显示升序或降序状态
  这是整个数据表的排序配置，但是我们还需要准确确定哪些表头是可排序的！
  在我们的例子中，只有seqNo列是可排序的，因此我们使用mat-sort-header指令注释列标题单元格。
  这涵盖了模板更改，现在让我们看一下对CourseComponent所做的更改，以启用表标题排序。
  
## 将可排序的列标题链接到数据源
  与分页一样，可排序标题将公开一个Observable，当用户单击可排序列标题时，该Observable会发出值。
  然后，MatSort指令公开一个Observable排序，它可以通过以下方式触发新页面加载：
  ```
  
@Component({
    selector: 'course',
    templateUrl: './course.component.html',
    styleUrls: ['./course.component.css']
})
export class CourseComponent implements AfterViewInit, OnInit  {

    course:Course;
    dataSource: LessonsDataSource;
    displayedColumns= ["seqNo", "description", "duration"];

    @ViewChild(MatPaginator) paginator: MatPaginator;
    @ViewChild(MatSort) sort: MatSort;

    constructor(private coursesService: CoursesService, private route: ActivatedRoute) {}

    ngOnInit() {
        this.course = this.route.snapshot.data["course"];
        this.dataSource = new LessonsDataSource(this.coursesService);
        this.dataSource.loadLessons(this.course.id, '', 'asc', 0, 3);
    }    
    
    ngAfterViewInit() {
        
        // reset the paginator after sorting
        this.sort.sortChange.subscribe(() => this.paginator.pageIndex = 0);
        
        merge(this.sort.sortChange, this.paginator.page)
            .pipe(
                tap(() => this.loadLessonsPage())
            )
            .subscribe();
    }

    loadLessonsPage() {
        this.dataSource.loadLessons(
            this.course.id,  '',  this.sort.direction, 
            this.paginator.pageIndex, this.paginator.pageSize);
    }
}

  ```
   如我们所见，Observable排序现在正在与Observable页面合并！ 现在，在两种情况下将触发新的页面加载：
  + 当发生分页事件时
  + 发生排序事件时

  seqNo列的排序方向现在从sort指令（通过@ViewChild（）注入）获取到后端。
  请注意，在每次排序之后，我们还通过强制显示已排序数据的第一页来重置分页器。
  
## Material表头排序表现
  这是带有可排序标题的数据表的外观，在加载数据并单击可排序标题之后（通过seqNo触发降序排序）：
  ![avatar](https://s3-us-west-1.amazonaws.com/angular-university/blog-images/angular-material-data-table/6-material-data-table.png)
  请注意seqNo列上的排序图标
  至此，我们已经完成了服务器分页和排序。 现在，我们准备添加最终的主要功能：服务器端过滤。
## 增加服务器端过滤
  为了实现服务器端过滤，我们需要做的第一件事是在模板中添加一个搜索框。
  并且由于这是最终版本，因此让我们显示具有其所有功能的完整模板：分页，排序以及服务器端过滤：
  ```
  
<div class="course">

    <!-- New part: this is the search box -->
    <mat-input-container>
        <input matInput placeholder="Search lessons" #input>
    </mat-input-container>

    <div class="spinner-container" *ngIf="dataSource.loading$ | async">
        <mat-spinner></mat-spinner>
    </div>

    <mat-table class="lessons-table mat-elevation-z8" [dataSource]="dataSource"
               matSort matSortActive="seqNo" matSortDirection="asc" matSortDisableClear>

        <ng-container matColumnDef="seqNo">
            <mat-header-cell *matHeaderCellDef mat-sort-header>#</mat-header-cell>
            <mat-cell *matCellDef="let lesson">{{lesson.seqNo}}</mat-cell>
        </ng-container>

        <ng-container matColumnDef="description">
            <mat-header-cell *matHeaderCellDef>Description</mat-header-cell>
            <mat-cell class="description-cell"
                      *matCellDef="let lesson">{{lesson.description}}</mat-cell>
        </ng-container>

        <ng-container matColumnDef="duration">
            <mat-header-cell *matHeaderCellDef>Duration</mat-header-cell>
            <mat-cell class="duration-cell"
                      *matCellDef="let lesson">{{lesson.duration}}</mat-cell>
        </ng-container>

        <mat-header-row *matHeaderRowDef="displayedColumns"></mat-header-row>
        <mat-row *matRowDef="let row; columns: displayedColumns"></mat-row>

    </mat-table>

    <mat-paginator [length]="course?.lessonsCount" [pageSize]="3"
                   [pageSizeOptions]="[3, 5, 10]"></mat-paginator>
</div>

  ```
## 分析搜索框的实现
  如我们所见，此最终模板版本中唯一的新部件是mat-input-container，其中包含“Material 输入”框，用户可以在其中输入搜索查询。
  此输入框遵循在Material 库中找到的常见模式：mat-input-container正在包装纯HTML输入并将其投影。
  这使我们能够完全访问所有标准输入属性，包括例如所有与可访问性相关的属性。 这也使我们与Angular Forms兼容，因为我们可以直接在输入HTML元素中应用Form指令。
  在这篇文章中阅读有关如何构建类似组件的更多信息：Angular ng-content和Content Projection：完整指南。
  请注意，此输入框甚至没有附加事件处理程序！ 然后让我们看一下组件，看看它是如何工作的。
  
## 服务器分页，排序和过滤的最终版本组件
  这是CourseComponent的最终版本，其中包括所有功能： 
```

@Component({
    selector: 'course',
    templateUrl: './course.component.html',
    styleUrls: ['./course.component.css']
})
export class CourseComponent implements OnInit, AfterViewInit {

    course:Course;
    dataSource: LessonsDataSource;
    displayedColumns= ["seqNo", "description", "duration"];

    @ViewChild(MatPaginator) paginator: MatPaginator;
    @ViewChild(MatSort) sort: MatSort;
    @ViewChild('input') input: ElementRef;

    constructor(
        private route: ActivatedRoute, 
        private coursesService: CoursesService) {}

    ngOnInit() {
        this.course = this.route.snapshot.data["course"];
        this.dataSource = new LessonsDataSource(this.coursesService);
        this.dataSource.loadLessons(this.course.id, '', 'asc', 0, 3);
    }

    ngAfterViewInit() {

        // server-side search
        fromEvent(this.input.nativeElement,'keyup')
            .pipe(
                debounceTime(150),
                distinctUntilChanged(),
                tap(() => {
                    this.paginator.pageIndex = 0;
                    this.loadLessonsPage();
                })
            )
            .subscribe();

        // reset the paginator after sorting
        this.sort.sortChange.subscribe(() => this.paginator.pageIndex = 0);

        // on sort or paginate events, load a new page
        merge(this.sort.sortChange, this.paginator.page)
        .pipe(
            tap(() => this.loadLessonsPage())
        )
        .subscribe();
    }

    loadLessonsPage() {
        this.dataSource.loadLessons(
            this.course.id,
            this.input.nativeElement.value,
            this.sort.direction,
            this.paginator.pageIndex,
            this.paginator.pageSize);
    }
}
```  
然后，让我们讨论分析服务器过滤部分。

## 获取对搜索输入框的引用
  我们可以看到我们已经使用@ViewChild（'input'）向<input>元素注入了DOM引用。 注意，这一次注入机制为我们提供了对DOM元素而不是组件的引用。
  使用该DOM参考，以下是当用户键入新查询时触发服务器端搜索的部分：
  ```
  
// server-side search
fromEvent(this.input.nativeElement,'keyup')
    .pipe(
        debounceTime(150),
        distinctUntilChanged(),
        tap(() => {
            this.paginator.pageIndex = 0;
            this.loadLessonsPage();
        })
    )
    .subscribe();

  ```
  我们在此所做的代码段是：我们正在使用搜索输入框，并且正在使用fromEvent创建一个Observable。
  每次发生新的keyUp事件时，此Observable都会发出一个值。 然后，我们将对此操作符应用几个运算符：
  .debounceTime(150):用户可以在输入框中快速键入内容，这可能会触发许多服务器请求。 使用此运算符，我们将发出的服务器请求的数量限制为每150ms最多一个。
  .distinctUntilChanged(): 该运算符将消除重复的值
  有了这两个运算符之后，我们现在可以通过通过tap（）运算符将查询字符串，页面大小和页面索引传递给数据源来触发页面加载。
  现在让我们看一下如果用户键入搜索词“ hello”，屏幕的外观：
  
  至此，我们已经完成了示例！ 现在，对于如何通过服务器端分页，排序和过滤来实现角度材料数据表，我们有了一个完整的解决方案。
  现在让我们快速总结一下我们学到的知识。

## 总结:
  数据表，数据源和相关组件是使用基于可观察的API的反应式设计的一个很好的例子。突出设计的关键点：
  + Material 数据表预期通过可观察对象从数据源接收数据
  + 数据源的主要作用是构建并提供一个Observable，该Observable将新版本的表格数据发送到数据表
  + 然后，像CourseService这样的组件类将所有内容“粘合”在一起
  这种反应性设计有助于确保所涉及的多个元素之间的松散耦合，并且可以将关注点分离开。

## 源代码+ Github运行示例
  Github上此分支上提供了完整代码的运行示例，其中包括小型后端Express服务器，该服务器提供数据并在服务器端进行排序/分页/过滤。
  我希望这篇文章能帮助您开始使用Angular Material Data Table，并且您喜欢它！
