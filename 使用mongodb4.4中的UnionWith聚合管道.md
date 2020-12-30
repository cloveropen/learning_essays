## 如何使用mongodb4.4中的Union-All聚合管道步骤
***
翻译自 https://developer.mongodb.com/how-to/use-union-all-aggregation-pipeline-stage
***
mongodb4.4的发布带来了一个新的聚合管道阶段$unionWith，允许将多个集合合并到单个结果集中。
用法如下:
    + 简化的语法，不需要对指定集合进行额外处理
```
db.collection.aggregate([
  { $unionWith: "<anotherCollection>" }
])
```
    + 扩展语法，使用pipeline字段
```
db.collection.aggregate([
  { $unionWith: { coll: "<anotherCollection>", pipeline: [ <stage1>, etc. ] } }
])
```
[~注意]如果在合并之前使用pipeline字段来处理集合，请记住不能使用$out和$merge等写入数据的阶段！
生成的文档将当前集合（或管道）的文档流与指定集合/管道中的文档合并。请记住，这可能包括重复数据！

#### 这听起来有点儿熟悉
如果以前在SQL中使用过UNION ALL操作，$unionWith stage的功能听起来可能很熟悉，两者都合并来自多个查询的结果集并返回合并的行，其中一些行可能是重复的。然而，这就是全部的相似之处。与MongoDB的$unionWith stage不同，要在SQL中运行有效的UNION ALL操作，必须遵循以下规则：
+ 两个查询的列数要相同
+ 列的顺序相同
+ 匹配的列是兼容的数据类型

在SQL中类似这样：
```
SELECT column1, expression1, column2
FROM table1
UNION ALL
SELECT column1, expression1, column2
FROM table2
WHERE [conditions]
```
使用MongoDB中的$unionWith stage，不必担心以上这些严格的限制。

#### 那么MongoDB的$unionWith stage有何不同呢？
$unionWith stage和其他联合操作之间最方便的区别是没有匹配的模式限制。这种灵活的模式支持意味着您可以组合可能不具有相同类型或字段数的文档。这在某些场景中很常见，我们需要使用的数据来自不同的来源：
+ 按月/季度/其他时间单位存储的时间轴数据

+ 
物联网设备数据，每个设备数据结构不同
+ 
存储在数据池中的历史数据和当前数据
+ 

区域数据
使用MongoDB的$unionWith stage，可以组合以上这些数据源。

准备好试试新的 $unionWith stage吗？接下来先完成几个设置步骤。或者，可以跳到[代码示例](https://developer.mongodb.com/how-to/use-union-all-aggregation-pipeline-stage#examples)。

#### 先决条件
首先，对[聚合框架](https://docs.mongodb.com/manual/aggregation/#aggregation-framework)的基本概念以及如何使用对于本教程的其余部分非常重要。如果不熟悉聚合框架，请查看这篇[关于MongoDB聚合框架的精彩介绍](https://developer.mongodb.com/quickstart/introduction-aggregation-framework)，它是由开发人员Ken Alger撰写的！
其次，根据实际情况，你可能已经具备了一些先决条件，或者需要从头开始。不管是哪种方式，请选择您的场景来配置所需的内容，以便您可以遵循本教程的其余部分！
选择场景:
#### 没有建立Altas cluster:
1. 如果没有MongoDB Atlas帐号需要先建立一个，然后登录Atlas帐户
2. 设置一个[免费的Atlas集群](https://docs.atlas.mongodb.com/tutorial/deploy-free-tier-cluster/)(不需要信用卡!).请务必在附加设置中选择MongoDB 4.4（可能是Beta版，这是可以的）作为您的版本！
[~注意]如果没有看到创建集群的提示：在看到创建第一个集群的提示之前，可能会提示您先创建一个项目。在这种情况下，请先创建一个项目（保留所有默认设置）。然后继续按照说明部署您的第一个免费集群！
3. 一旦群集设置完毕，请将您的IP地址添加到群集的连接设置中。我们动态IP填入0.0.0.0
4. 最后，为集群[创建一个数据库用户](https://docs.atlas.mongodb.com/tutorial/create-mongodb-user-for-cluster/)。为了安全起见，Atlas要求任何访问其集群的人或任何人以MongoDB数据库用户身份进行身份验证！把这些证书放在手边，因为你以后会需要它们的。
5. 继续执行连接到群集中的步骤。

#### 已经建立好Atlas cluster:
太好了！您可以跳到连接到群集步骤。

#### 连接到集群
 为了连接到集群，我们将使用MongoDB for visualstudio代码扩展（简称VS-Code）??). 您可以直接查看您的数据，与您的集合进行交互，以及使用此有用的扩展进行更多操作！使用它还将我们的工作区整合到一个窗口中，消除了我们在代码和MongoDB Atlas之间来回跳转的需要！
[~注意]?? 尽管在本教程的其余部分我们将使用VS代码扩展和VS代码，但使用$unionWith pipeline阶段并不是必需的！如果愿意，还可以使用CLI、特定语言的驱动程序或Compass！
1. 安装 MongoDB for VS Code扩展(如果没有安装VS Code应该先安装VS Code)
2. 要连接到集群，需要一个连接字符串。可以从群集连接设置中获取此连接字符串。转到群集并选择“连接”选项：
(https://developer.mongodb.com/images/article/union-all/connect.png)
3. 选择“使用MongoDB Compass连接”选项。这将为我们提供一个DNS种子列表连接格式的连接字符串，我们可以使用MongoDB扩展。
(https://developer.mongodb.com/images/article/union-all/connect-with-compass.png)
[~注意]mongodb for vs代码扩展还支持标准连接字符串格式。使用DNS种子列表连接格式纯粹是首选。
4. 跳到第二步并复制连接字符串(不要担心其他设置，您将不需要它们):
(https://developer.mongodb.com/images/article/union-all/copy-connection-string.png)
5. 切换回VS代码。按Ctrl+Shift+P（在Windows上）或Shift+Command+P（在Mac上）打开命令工具箱。这将显示所有VS代码命令的列表。
(https://developer.mongodb.com/images/article/union-all/show-commands.png)  
6. 输入“MongoDB”，直到看到MongoDB扩展的可用命令列表。选择“MongoDB:connectwithconnectionstring”选项。
(https://developer.mongodb.com/images/article/union-all/mdb-connect-with-connection-string.png)
7. 粘贴复制的连接字符串。?? 别忘了！您必须用实际密码替换占位符密码！
(https://developer.mongodb.com/images/article/union-all/mdb-paste-conn-string.png)
8. 按回车键连接！如果您看到右下角的确认消息，您将知道连接成功。当您展开MongoDB扩展窗格时，还将看到列出的集群。

安装了MongoDB扩展并连接了集群后，现在可以使用MongoDB Playgrounds测试$unionWith的例子！MongoDB Playgrounds为我们提供了一个很好的沙盒，可以方便地编写和测试Mongo查询。我喜欢在进行原型开发或尝试一些新的东西时使用它，因为它有查询自动完成和语法高亮显示，这在大多数终端中是没有的。
最后让我们来看看一些例子！

#### 例子
你可以使用本篇文章的[MongoDB Playground](https://github.com/adriennetacke/mdb-union-all-walkthrough)文件或自己创建文件
[~注意]?? 如果创建自己的游乐场，请记住先更改数据库名称并删除默认模板的代码！

使用管道来使用$unionWith
[~注意]?? 如果您想遵循本示例的预先编写的代码，请使用[这个playground](https://github.com/adriennetacke/mdb-union-all-walkthrough/blob/master/unionWith_examples_similar_schemas.mongodb)。
在顶部，指定要使用的数据库。在本例中，我使用的数据库也称为 union-walkthrough:
```
use('union-walkthrough');
```
[~注意]?? 实际上，我还没有在Atlas中创建一个名为union walkthrough的数据库，但这没有问题！当playground运行时，它将看到它还不存在，并创建一个指定名称的数据库
下一步，我们需要数据！特别是一些行星，某一系列电影中的行星。
使用[SWAPI API](https://swapi.dev/),我搜集了一些关于行星的信息，我们把他们写入用popularity区分的两个集合中。
任何出现在至少两部或更多电影中的行星都被认为是受欢迎的。否则，我们会将它们添加到lonely_planets集合中：
```
// Insert a few documents into the lonely_planets collection.
db.lonely_planets.insertMany([
    {
        "name": "Endor",
        "rotation_period": "18",
        "orbital_period": "402",
        "diameter": "4900",
        "climate": "temperate",
        "gravity": "0.85 standard",
        "terrain": "forests, mountains, lakes",
        "surface_water": "8",
        "population": "30000000",
        "residents": [
            "http://swapi.dev/api/people/30/"
        ],
        "films": [
            "http://swapi.dev/api/films/3/"
        ],
        "created": "2014-12-10T11:50:29.349000Z",
        "edited": "2014-12-20T20:58:18.429000Z",
        "url": "http://swapi.dev/api/planets/7/"
    },
    {
        "name": "Kamino",
        "rotation_period": "27",
        "orbital_period": "463",
        "diameter": "19720",
        "climate": "temperate",
        "gravity": "1 standard",
        "terrain": "ocean",
        "surface_water": "100",
        "population": "1000000000",
        "residents": [
            "http://swapi.dev/api/people/22/",
            "http://swapi.dev/api/people/72/",
            "http://swapi.dev/api/people/73/"
        ],
        "films": [
            "http://swapi.dev/api/films/5/"
        ],
        "created": "2014-12-10T12:45:06.577000Z",
        "edited": "2014-12-20T20:58:18.434000Z",
        "url": "http://swapi.dev/api/planets/10/"
    },
    {
        "name": "Yavin IV",
        "rotation_period": "24",
        "orbital_period": "4818",
        "diameter": "10200",
        "climate": "temperate, tropical",
        "gravity": "1 standard",
        "terrain": "jungle, rainforests",
        "surface_water": "8",
        "population": "1000",
        "residents": [],
        "films": [
            "http://swapi.dev/api/films/1/"
        ],
        "created": "2014-12-10T11:37:19.144000Z",
        "edited": "2014-12-20T20:58:18.421000Z",
        "url": "http://swapi.dev/api/planets/3/"
    },
    {
        "name": "Hoth",
        "rotation_period": "23",
        "orbital_period": "549",
        "diameter": "7200",
        "climate": "frozen",
        "gravity": "1.1 standard",
        "terrain": "tundra, ice caves, mountain ranges",
        "surface_water": "100",
        "population": "unknown",
        "residents": [],
        "films": [
            "http://swapi.dev/api/films/2/"
        ],
        "created": "2014-12-10T11:39:13.934000Z",
        "edited": "2014-12-20T20:58:18.423000Z",
        "url": "http://swapi.dev/api/planets/4/"
    },
    {
        "name": "Bespin",
        "rotation_period": "12",
        "orbital_period": "5110",
        "diameter": "118000",
        "climate": "temperate",
        "gravity": "1.5 (surface), 1 standard (Cloud City)",
        "terrain": "gas giant",
        "surface_water": "0",
        "population": "6000000",
        "residents": [
            "http://swapi.dev/api/people/26/"
        ],
        "films": [
            "http://swapi.dev/api/films/2/"
        ],
        "created": "2014-12-10T11:43:55.240000Z",
        "edited": "2014-12-20T20:58:18.427000Z",
        "url": "http://swapi.dev/api/planets/6/"
    }
]);

// Insert a few documents into the popular_planets collection.
db.popular_planets.insertMany([
    {
        "name": "Tatooine",
        "rotation_period": "23",
        "orbital_period": "304",
        "diameter": "10465",
        "climate": "arid",
        "gravity": "1 standard",
        "terrain": "desert",
        "surface_water": "1",
        "population": "200000",
        "residents": [
            "http://swapi.dev/api/people/1/",
            "http://swapi.dev/api/people/2/",
            "http://swapi.dev/api/people/4/",
            "http://swapi.dev/api/people/6/",
            "http://swapi.dev/api/people/7/",
            "http://swapi.dev/api/people/8/",
            "http://swapi.dev/api/people/9/",
            "http://swapi.dev/api/people/11/",
            "http://swapi.dev/api/people/43/",
            "http://swapi.dev/api/people/62/"
        ],
        "films": [
            "http://swapi.dev/api/films/1/",
            "http://swapi.dev/api/films/3/",
            "http://swapi.dev/api/films/4/",
            "http://swapi.dev/api/films/5/",
            "http://swapi.dev/api/films/6/"
        ],
        "created": "2014-12-09T13:50:49.641000Z",
        "edited": "2014-12-20T20:58:18.411000Z",
        "url": "http://swapi.dev/api/planets/1/"
    },
    {
        "name": "Alderaan",
        "rotation_period": "24",
        "orbital_period": "364",
        "diameter": "12500",
        "climate": "temperate",
        "gravity": "1 standard",
        "terrain": "grasslands, mountains",
        "surface_water": "40",
        "population": "2000000000",
        "residents": [
            "http://swapi.dev/api/people/5/",
            "http://swapi.dev/api/people/68/",
            "http://swapi.dev/api/people/81/"
        ],
        "films": [
            "http://swapi.dev/api/films/1/",
            "http://swapi.dev/api/films/6/"
        ],
        "created": "2014-12-10T11:35:48.479000Z",
        "edited": "2014-12-20T20:58:18.420000Z",
        "url": "http://swapi.dev/api/planets/2/"
    },
    {
        "name": "Naboo",
        "rotation_period": "26",
        "orbital_period": "312",
        "diameter": "12120",
        "climate": "temperate",
        "gravity": "1 standard",
        "terrain": "grassy hills, swamps, forests, mountains",
        "surface_water": "12",
        "population": "4500000000",
        "residents": [
            "http://swapi.dev/api/people/3/",
            "http://swapi.dev/api/people/21/",
            "http://swapi.dev/api/people/35/",
            "http://swapi.dev/api/people/36/",
            "http://swapi.dev/api/people/37/",
            "http://swapi.dev/api/people/38/",
            "http://swapi.dev/api/people/39/",
            "http://swapi.dev/api/people/42/",
            "http://swapi.dev/api/people/60/",
            "http://swapi.dev/api/people/61/",
            "http://swapi.dev/api/people/66/"
        ],
        "films": [
            "http://swapi.dev/api/films/3/",
            "http://swapi.dev/api/films/4/",
            "http://swapi.dev/api/films/5/",
            "http://swapi.dev/api/films/6/"
        ],
        "created": "2014-12-10T11:52:31.066000Z",
        "edited": "2014-12-20T20:58:18.430000Z",
        "url": "http://swapi.dev/api/planets/8/"
    },
    {
        "name": "Coruscant",
        "rotation_period": "24",
        "orbital_period": "368",
        "diameter": "12240",
        "climate": "temperate",
        "gravity": "1 standard",
        "terrain": "cityscape, mountains",
        "surface_water": "unknown",
        "population": "1000000000000",
        "residents": [
            "http://swapi.dev/api/people/34/",
            "http://swapi.dev/api/people/55/",
            "http://swapi.dev/api/people/74/"
        ],
        "films": [
            "http://swapi.dev/api/films/3/",
            "http://swapi.dev/api/films/4/",
            "http://swapi.dev/api/films/5/",
            "http://swapi.dev/api/films/6/"
        ],
        "created": "2014-12-10T11:54:13.921000Z",
        "edited": "2014-12-20T20:58:18.432000Z",
        "url": "http://swapi.dev/api/planets/9/"
    },
    {
        "name": "Dagobah",
        "rotation_period": "23",
        "orbital_period": "341",
        "diameter": "8900",
        "climate": "murky",
        "gravity": "N/A",
        "terrain": "swamp, jungles",
        "surface_water": "8",
        "population": "unknown",
        "residents": [],
        "films": [
            "http://swapi.dev/api/films/2/",
            "http://swapi.dev/api/films/3/",
            "http://swapi.dev/api/films/6/"
        ],
        "created": "2014-12-10T11:42:22.590000Z",
        "edited": "2014-12-20T20:58:18.425000Z",
        "url": "http://swapi.dev/api/planets/5/"
    }
]);
```
这种区分表明我们的数据是如何分组的。尽管区分了，如果我们需要将这两个集合作为单个结果集进行分析，我们可以使用$unionWith stage来组合它们！



假设我们需要找出按气候分组的行星总数。另外，我们想从我们的计算中剔除任何没有人口数据的行星。我们可以使用聚合：
```
// Run an aggregation to view total planet populations, grouped by climate type.
use('union-walkthrough');

db.lonely_planets.aggregate([
    {
        $match: {
            population: { $ne: 'unknown' }
        }
    },
    {
        $unionWith: {
            coll: 'popular_planets',
            pipeline: [{
                $match: {
                    population: { $ne: 'unknown' }
                }
            }]
        }
    },
    {
        $group: {
            _id: '$climate', totalPopulation: { $sum: { $toLong: '$population' } }
        }
    }
]);
```
如果您已经在自己的MongoDB playground中进行了跟踪，并且已经复制了代码，请尝试运行聚合！



如果您使用的是我创建的MongoDB playground，请突出显示第264-290行，然后运行所选代码。

[~注意]?? 您将注意到在上面的代码片段中，我在聚合代码的正上方添加了另一个use（'union-walkthrough'）；方法。我这样做是为了在playground更容易地选择相关代码。对正确的数据库运行所需的代码，以便它也可以运行。但是，通过选择多行也可以实现同样的效果，即顶部的原始use（'union-walkthrough'）行和任何您想运行的其他示例！

应该看到如下类似的结果:
```
[
  {
    _id: 'arid',
    totalPopulation: 200000
  },
  {
    _id: 'temperate',
    totalPopulation: 1007536000000
  },
  {
    _id: 'temperate, tropical',
    totalPopulation: 1000
  }
]
```
毫不奇怪，气候“温和”的行星似乎有更多的居民。我想大概是75华氏度23.8摄氏度吧

让我们分析这个聚合查询：
我们传递到聚合中的第一个对象也是我们的第一个阶段，这里用作我们的筛选条件。具体来说，我们使用$match pipeline阶段：
```
{
    $match: {
        population: { $ne: 'unknown' }
    }
},
```
在本例中，我们使用$ne（not equal）操作符筛选出所有具有未知人口数量的文档。
聚合中的下一个对象（和下一个阶段）是$unionWith阶段。在这里，我们指定要与哪个集合执行联合（包括任何副本）。我们还使用pipeline字段类似地筛选出流行的集合中具有未知总体的任何文档：
```
{
    $unionWith: {
        coll: 'popular_planets',
        pipeline: [
            {
                $match: {
                    population: { $ne: 'unknown' }
                }
            }
        ]
    }
},
```
最后，我们有了聚合的最后阶段。在合并了我们的lonely_planets和流行的\u planets集合（两者都过滤掉没有人口数据的文档）之后，我们使用$group阶段对生成的文档进行分组：
```
{
    $group: {
        _id: '$climate',
        totalPopulation: { $sum: { $toLong: '$population' } }
    }
}
```
因为我们想知道每种气候类型的总人口，所以我们首先从合并结果集中指定 _id为$climate字段。然后，我们通过使用$sum运算符将每个匹配文档的总体值相加，来计算一个名为totalPopulation的新字段。您还将注意到，根据我们拥有的数据，我们需要使用$toLong运算符首先将$population字段转换为可计算的值！

#### 不使用pipeline的$unionWith
[~注意]如果您想遵循本示例的预先编写的代码，请使用[这个playground](https://github.com/adriennetacke/mdb-union-all-walkthrough/blob/master/unionWith_examples_similar_schemas.mongodb)。
现在，如果您不需要对要合并的集合运行一些额外的处理，那么您不必这样做！pipeline字段是可选的，仅在需要时才存在。



如果你需要一个统一的数据集，你也可以这样做：
```
// Run an aggregation with no pipeline
use('union-walkthrough');

db.lonely_planets.aggregate([
    { $unionWith: 'popular_planets' }
]);
```
将此聚合复制到您自己的playground并运行它！或者，如果使用提供的MongoDB平台，则选择并运行第293-297行！现在可以使用这个统一的数据集进行分析或进一步处理。

#### 不同的Schemas
组合相同的模式是很方便的，但是我们也可以在常规SQL中这样做！$unionWith pipeline stage的真正便利之处在于，它还可以组合具有不同模式的集合。我们来看看！
#### $unionWith使用具有不同架构的集合
如果您想遵循本示例的预先编写的代码，请使用[这个playground](https://github.com/adriennetacke/mdb-union-all-walkthrough/blob/master/unionWith_examples_different_schemas.mongodb)。
如前所述，我们指定要使用的数据库：
```
use('union-walkthrough');
```
这一次，我们将使用一些在同一系列电影中使用的关于某些星际飞船和交通工具的信息。让我们将它们添加到各自的集合中：
```
// Insert a few documents into the starships collection
db.starships.insertMany([
    {
        "name": "Death Star",
        "model": "DS-1 Orbital Battle Station",
        "manufacturer": "Imperial Department of Military Research, Sienar Fleet Systems",
        "cost_in_credits": "1000000000000",
        "length": "120000",
        "max_atmosphering_speed": "n/a",
        "crew": 342953,
        "passengers": 843342,
        "cargo_capacity": "1000000000000",
        "consumables": "3 years",
        "hyperdrive_rating": 4.0,
        "MGLT": 10,
        "starship_class": "Deep Space Mobile Battlestation",
        "pilots": []
    },
    {
        "name": "Millennium Falcon",
        "model": "YT-1300 light freighter",
        "manufacturer": "Corellian Engineering Corporation",
        "cost_in_credits": "100000",
        "length": "34.37",
        "max_atmosphering_speed": "1050",
        "crew": 4,
        "passengers": 6,
        "cargo_capacity": 100000,
        "consumables": "2 months",
        "hyperdrive_rating": 0.5,
        "MGLT": 75,
        "starship_class": "Light freighter",
        "pilots": [
            "http://swapi.dev/api/people/13/",
            "http://swapi.dev/api/people/14/",
            "http://swapi.dev/api/people/25/",
            "http://swapi.dev/api/people/31/"
        ]
    },
    {
        "name": "Y-wing",
        "model": "BTL Y-wing",
        "manufacturer": "Koensayr Manufacturing",
        "cost_in_credits": "134999",
        "length": "14",
        "max_atmosphering_speed": "1000km",
        "crew": 2,
        "passengers": 0,
        "cargo_capacity": 110,
        "consumables": "1 week",
        "hyperdrive_rating": 1.0,
        "MGLT": 80,
        "starship_class": "assault starfighter",
        "pilots": []
    },
    {
        "name": "X-wing",
        "model": "T-65 X-wing",
        "manufacturer": "Incom Corporation",
        "cost_in_credits": "149999",
        "length": "12.5",
        "max_atmosphering_speed": "1050",
        "crew": 1,
        "passengers": 0,
        "cargo_capacity": 110,
        "consumables": "1 week",
        "hyperdrive_rating": 1.0,
        "MGLT": 100,
        "starship_class": "Starfighter",
        "pilots": [
            "http://swapi.dev/api/people/1/",
            "http://swapi.dev/api/people/9/",
            "http://swapi.dev/api/people/18/",
            "http://swapi.dev/api/people/19/"
        ]
    },
]);

// Insert a few documents into the vehicles collection
db.vehicles.insertMany([
    {
        "name": "Sand Crawler",
        "model": "Digger Crawler",
        "manufacturer": "Corellia Mining Corporation",
        "cost_in_credits": "150000",
        "length": "36.8 ",
        "max_atmosphering_speed": 30,
        "crew": 46,
        "passengers": 30,
        "cargo_capacity": 50000,
        "consumables": "2 months",
        "vehicle_class": "wheeled",
        "pilots": []
    },
    {
        "name": "X-34 landspeeder",
        "model": "X-34 landspeeder",
        "manufacturer": "SoroSuub Corporation",
        "cost_in_credits": "10550",
        "length": "3.4 ",
        "max_atmosphering_speed": 250,
        "crew": 1,
        "passengers": 1,
        "cargo_capacity": 5,
        "consumables": "unknown",
        "vehicle_class": "repulsorcraft",
        "pilots": [],
    },
    {
        "name": "AT-AT",
        "model": "All Terrain Armored Transport",
        "manufacturer": "Kuat Drive Yards, Imperial Department of Military Research",
        "cost_in_credits": "unknown",
        "length": "20",
        "max_atmosphering_speed": 60,
        "crew": 5,
        "passengers": 40,
        "cargo_capacity": 1000,
        "consumables": "unknown",
        "vehicle_class": "assault walker",
        "pilots": [],
        "films": [
            "http://swapi.dev/api/films/2/",
            "http://swapi.dev/api/films/3/"
        ],
        "created": "2014-12-15T12:38:25.937000Z",
        "edited": "2014-12-20T21:30:21.677000Z",
        "url": "http://swapi.dev/api/vehicles/18/"
    },
    {
        "name": "AT-ST",
        "model": "All Terrain Scout Transport",
        "manufacturer": "Kuat Drive Yards, Imperial Department of Military Research",
        "cost_in_credits": "unknown",
        "length": "2",
        "max_atmosphering_speed": 90,
        "crew": 2,
        "passengers": 0,
        "cargo_capacity": 200,
        "consumables": "none",
        "vehicle_class": "walker",
        "pilots": [
            "http://swapi.dev/api/people/13/"
        ]
    },
    {
        "name": "Storm IV Twin-Pod cloud car",
        "model": "Storm IV Twin-Pod",
        "manufacturer": "Bespin Motors",
        "cost_in_credits": "75000",
        "length": "7",
        "max_atmosphering_speed": 1500,
        "crew": 2,
        "passengers": 0,
        "cargo_capacity": 10,
        "consumables": "1 day",
        "vehicle_class": "repulsorcraft",
        "pilots": [],
    }
]);
```
你可能会想（就像我第一次想的那样），星际飞船和飞船有什么区别？你会很高兴知道星际飞船被定义为任何“具有超光速能力的单一运输飞船”。任何其他不具备超空间驱动能力的单一运输船都被视为运载工具。



如果您查看这两个集合，您会发现它们有两个关键区别：
 + max_atmosphering_speed字段在两个集合中都存在，但在starships集合中是一个字符串，在vehicles集合中是一个int。
 + 星际飞船集合中有两个字段（超光速引擎等级，MGLT）不在载具集合中，因为它只与星际飞船相关。
但你知道吗？这对于$unionWith stage不是问题！您可以像以前一样组合它们
```
// Run an aggregation with no pipeline and differing schemas
use('union-walkthrough');

db.starships.aggregate([
  { $unionWith: 'vehicles' }
]);
```
试着在playground中运行程序，或者如果您在我提供的MongoDB playground，请选择并运行185-189行！您应该获得以下组合结果集输出：
```
[
  {
    _id: 5f306ddca3ee8339643f137e,
    name: 'Death Star',
    model: 'DS-1 Orbital Battle Station',
    manufacturer: 'Imperial Department of Military Research, Sienar Fleet Systems',
    cost_in_credits: '1000000000000',
    length: '120000',
    max_atmosphering_speed: 'n/a',
    crew: 342953,
    passengers: 843342,
    cargo_capacity: '1000000000000',
    consumables: '3 years',
    hyperdrive_rating: 4,
    MGLT: 10,
    starship_class: 'Deep Space Mobile Battlestation',
    pilots: []
  },
  {
    _id: 5f306ddca3ee8339643f137f,
    name: 'Millennium Falcon',
    model: 'YT-1300 light freighter',
    manufacturer: 'Corellian Engineering Corporation',
    cost_in_credits: '100000',
    length: '34.37',
    max_atmosphering_speed: '1050',
    crew: 4,
    passengers: 6,
    cargo_capacity: 100000,
    consumables: '2 months',
    hyperdrive_rating: 0.5,
    MGLT: 75,
    starship_class: 'Light freighter',
    pilots: [
      'http://swapi.dev/api/people/13/',
      'http://swapi.dev/api/people/14/',
      'http://swapi.dev/api/people/25/',
      'http://swapi.dev/api/people/31/'
    ]
  },
  // + 7 other results, omitted for brevity
]
```
你能想象在SQL中这样做吗？提示：你不能！不过，对于MongoDB来说，是不需要担心这种模式限制的！

#### $unionWith使用不同模式和管道的集合
[^注意]如果您想遵循本示例的预编写代码，请使用[这个playground](https://github.com/adriennetacke/mdb-union-all-walkthrough/blob/master/unionWith_examples_different_schemas.mongodb)。
这样我们就可以组合不同的模式。 如果在合并之前需要对集合做一些额外的工作怎么办？ 这就是管道领域的用武之地！
假设我们的车辆数据中包含一些机密信息。 即，任何由Kuat Drive Yards（又称帝国军事研究部门的子公司）制造的车辆。
通过直接命令，指示您在任何情况下都不要提供此信息。 实际上，您需要拦截对车辆信息的任何请求，并将这些分类的车辆从列表中删除！
我们可以这样做：
```
use('union-walkthrough');

db.starships.aggregate([
    {
        $unionWith: {
            coll: 'vehicles',
            pipeline: [
                {
                    $redact: {
                        $cond: {
                            if: { $eq: [ "$manufacturer", "Kuat Drive Yards, Imperial Department of Military Research"] },
                            then: "$$PRUNE",
                            else: "$$DESCEND"
                        }
                    }
                }
            ]
        }
    }
]);
```
在这个例子中，我们使用$unionWith pipeline stage像以前一样组合了星际飞船和车辆集合。 我们还使用$ unionWith的可选管道字段来处理车辆数据：
```
// Pipeline used with the vehicle collection
{
    $redact: {
        $cond: {
            if: { $eq: [ "$manufacturer", "Kuat Drive Yards, Imperial Department of Military Research"] },
            then: "$$PRUNE",
            else: "$$DESCEND"
        }
    }
}
```
在$unionWith的管道中，我们使用$redact stage根据条件限制文档的内容。 使用$cond运算符指定条件，该运算符的作用类似于if /else语句。
在我们的案例中，我们正在评估制造商字段是否具有“军事研究帝国学院Kuat Drive Yards”的值。 如果是的话（嗯，那是分类的！），我们使用一个名为$$ PRUNE的系统变量，该变量允许我们排除当前文档/嵌入式文档级别的所有字段。 如果没有，我们将使用另一个名为$$ DESCEND的系统变量，它将返回当前文档级别的所有字段，但所有嵌入式文档除外。
这非常适合我们的用例。 尝试运行聚合（如果使用提供的MongoDB Playground，则运行第192-211行）。 您应该看到一个组合的结果集，减去任何帝制制造的车辆：
```
[
  {
    _id: 5f306ddca3ee8339643f137e,
    name: 'Death Star',
    model: 'DS-1 Orbital Battle Station',
    manufacturer: 'Imperial Department of Military Research, Sienar Fleet Systems',
    cost_in_credits: '1000000000000',
    length: '120000',
    max_atmosphering_speed: 'n/a',
    crew: 342953,
    passengers: 843342,
    cargo_capacity: '1000000000000',
    consumables: '3 years',
    hyperdrive_rating: 4,
    MGLT: 10,
    starship_class: 'Deep Space Mobile Battlestation',
    pilots: []
  },
  {
    _id: 5f306ddda3ee8339643f1383,
    name: 'X-34 landspeeder',
    model: 'X-34 landspeeder',
    manufacturer: 'SoroSuub Corporation',
    cost_in_credits: '10550',
    length: '3.4 ',
    max_atmosphering_speed: 250,
    crew: 1,
    passengers: 1,
    cargo_capacity: 5,
    consumables: 'unknown',
    vehicle_class: 'repulsorcraft',
    pilots: []
  },
  // + 5 more non-Imperial manufactured results, omitted for brevity
]
```
我们尽力限制了机密信息！
#### 对UNION ALL的限制
现在我们知道$unionWith stage的工作原理，讨论其限制和约束也很重要。
#### 重复项
我们已经提到了它，但是重要的是要重申：使用$unionWith stage将为您提供一个可能包含重复项的组合结果集！ 这等效于UNION ALL运算符在SQL中的工作方式。 作为一种解决方法，建议在管道末尾使用$group阶段删除重复项，但仅在可能的情况下，并且当结果数据不会出现不正确的可能时，才建议这样做。
有计划向UNION添加类似的功能（合并结果集但删除重复项），但是可能会在将来的版本中使用。
#### 共享集合
如果将$unionWith阶段用作$lookup管道的一部分，则无法对您为$unionWith指定的集合进行分片。 例如，看一下这种聚合：
```
// Invalid aggregation (tried to use sharded collection with $unionWith)
db.lonely_planets.aggregate([
    {
      $lookup: {
          from: "extinct_planets",
          let: { last_known_population: "$population", years_extinct: "$time_extinct" },
          pipeline: [
            // Filter criteria
            { $unionWith: { coll: "questionable_planets", pipeline: [ { pipeline } ] } },
            // Other pipeline stages
          ],
          as: "planetdata"
      }
    }
])
```
不能对collquestable_planets（位于$unionWith阶段内）进行分片。 强制执行此操作是为了防止由于群集确定最佳执行计划时由于对群集周围的数据进行改组而导致性能显着下降。

#### Transactions
聚合管道不能在事务内部使用$unionWith阶段，因为在非常特殊的情况下，极少发生但可能的3线程死锁。 此外，在MongoDB 4.4中，首次定义了视图，该视图将限制其在事务中的读取。

#### $out和$merge
$out和$merge阶段不能在$unionWith管道中使用。 由于$out和$merge都是将数据写入集合的阶段，因此它们必须是管道中的最后一个阶段。 这与$ unionWith阶段的用法冲突，因为它将其组合结果集输出到下一个阶段，该结果集可在聚合管道中的任何点使用。
#### Collations
如果您的聚合包含排序规则，则该排序规则将用于该操作，而忽略任何其他排序规则。
但是，如果您的聚合不包含排序规则，它将对运行该聚合的顶级集合/视图使用该排序规则：
  + 如果$ unionWith coll是一个集合，则其排序规则将被忽略。
  + 如果$ unionWith coll是一个视图，则其排序规则必须与顶级集合/视图的排序规则匹配。 否则，操作错误。

#### 结论
我们已经讨论了$ unionWith管道阶段是什么，以及如何在聚合中使用它来合并来自多个集合的数据。 尽管类似于SQL的UNION ALL操作，但MongoDB的$ unionWith阶段通过一些方便且急需的特性来与众不同。 最值得注意的是能够将集合与不同的模式结合起来！ 作为一项急需的改进，使用$ unionWith阶段无需编写其他代码，因为我们没有其他方法可以合并数据，因此无需编写其他代码！

如果您对$ unionWith管道阶段或此博客帖子有任何疑问，请前往[MongoDB社区论坛](https://developer.mongodb.com/community/forums/)或向我发送[推特](https://twitter.com/adriennetacke)！
