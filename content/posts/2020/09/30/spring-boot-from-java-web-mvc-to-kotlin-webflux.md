---
title: "Spring Boot: From Java Web MVC to Kotlin WebFlux"
date: 2020-09-30
authors: ["fanda"]
summary: Spring Boot has been around for some time and still seems to be the most popular framework when it comes to building modern web applications. Similarly about Kotlin language. It’s not as old as Spring framework but it’s been highly accepted and used by many developers. To me, it’s no surprise that people behind Spring (Pivotal) put the Kotlin language almost at the same level as Java when it comes to integration into the framework itself.
images:
  - posts/2020/09/30/images/cover-img.jpg
caption:
---
# From Spring Web MVC to WebFlux

Spring Boot has been around for some time and still seems to be the most popular framework when it comes to building modern web applications.

Similarly about Kotlin language. It's not as old as Spring framework but it's been highly accepted and used by many developers. To me, it's no surprise that people behind Spring (Pivotal) put the Kotlin language almost at the same level as Java when it comes to integration into the framework itself.

Most of my projects were not super complex and I could categorize them as **CRUD** applications based on **Spring Web MVC** stack using REST APIs, JPA (Java Persitance API with Hibernate) and Liquibase as database migration tool.

<blockquote class="blue">

CRUD - acronym for Create, Read, Update, Delete operations

</blockquote>

### Spring Web MVC

Spring's web MVC framework is request-driven, designed around a central Servlet that dispatches requests to controllers and provides functionality that facilitates the development of web application. It's still one of most used framework within the Spring Boot applications in the industry.

Typicall code structure with Web MVC looks like the following code snippets - they come from one of my very old pet project that would allow searching for a city by its name and planing a trip from one city to another.  The purpose of this application was purely educational - I just wanted to understand a bit more [JPA](https://docs.oracle.com/javaee/6/tutorial/doc/bnbpz.html), [Annotations](https://en.wikipedia.org/wiki/Java_annotation) and Spring framework itself in detail.

```java

@RestController
@RequestMapping("/api/cities")
public class CityController {

    private final CityService cityService;

    public CityController(CityService cityService) {
        this.cityService = cityService;
    }

    @GetMapping
    public List<City> findAllCities() {
        return cityService.findAll();
    }
}

```

```java

@Service
public class CityService {

    private final CityRepository cityRepository;
    
    public CityService(CityRepository cityRepository) {
        this.cityRepository = cityRepository;
    }
    
    public List<City> findAll(){
        return this.cityRepository.findAll();
    }
}

```

```java

interface CityRepository extends CrudRepository<City, Long> {
}

```

```java

@Entity
public class City {
    public City() {
    }
    @Id
    @GeneratedValue
    private Long id;
    private String name;
    @Column(name = "country")
    private String countryCode;
    @Column(name = "lat")
    private Double latitude;
    @Column(name = "lng")
    private Double longitude;
}

```

As you can see it's a simple application written **in imperative way** in Java language. In one of my latest project I've been a given a chance to develop application using **Spring WebFlux**, **Kotlin** and **functional REST API endpoints**. This experience made me to re-visit this old project and re-write it in a different way using the freshly acquired knowledge. To use **Kotlin** and define my REST API as **functional endpoints**.

### Spring WebFlux

Spring WebFlux is a framework based on the reactive-stack. It has been created as an alternative and at some point successor of the previous Spring Web MVC framework where the main goal was leaverage and support following features:

<div class="list-margin">

- non-blocking approach to handle concurrency with smaller number of threads
- scalability
- **functional programming and functional endpoints**

</div>

<blockquote class="blue">

Reactive programming = paradigm that allows building non-blocking (async) applications. Such applications can better handle the flow controll (back-pressure). More detailed explenation under [spring.io/reactive](https://spring.io/reactive)

</blockquote>

Both of the frameworks are still sharing some modules and can work side be side independently. Both frameworks support asynchronous and [reactive types](https://spring.io/blog/2016/04/19/understanding-reactive-types), but when it comes to writes to response then Web MVC is using **BLOCKING** approach (separate thread execution). WebFlux relies in this case on **NON-BLOCKING I/O** and doesn't requrie extra thread as its asynchronous by design (concept of event loop).

![](../images/image1.png#image-center-75)\
*Diagram taken from WebFlux documentation*

### Functional REST API endpoints

When we define functional enpdoints then the traditional concept of @RestController and @RequestMapping is replaced by **RouterFunction**. Since Spring version 5.2 the endpoints can be defined in Web MVC using functional approach as well.

```java

@Bean
public RouterFunction<ServerResponse> findAllCities(CityService cityService) {
    return route().GET("/api/cities", req -> ok().body(cityService.findAll()))
      .build();
}

```

As you can see the @RestController, @GetMapping, @RequestMapping annotations are gone and the route definition is moved now the method itself.

When defining the same functional endpoint using **WebFlux** and **Kotlin** then there are 2 options how to do it

<div class="list-margin">

- Registering the **route bean object** via @Bean annotation
- Registering the **route  bean object** using lambdas and bean DSL (Domain Specific Language)

</div>

```kotlin

@Bean
fun findAllCities(cityService: CityService): RouterFunction<ServerResponse> {
    return route().GET("/api/cities") { ok().body(cityService.findAll()) }
            .build()
}

```

When we compare it with the Java equivalent then the Kotlin version is almost identical. Difference comes when using the Bean DSL.

```kotlin

bean {
    coRouter {
        val cityHandler = CityHandler(ref())
        GET("/api/cities", cityHandler::findCities)
    }
}

class CityHandler(private val cityService: CityService) {

    suspend fun findCities(request: ServerRequest): ServerResponse {
        return ServerResponse
            .ok()
            .contentType(MediaType.APPLICATION_JSON)
            .bodyAndAwait(cityServices.findAll())
    }  
}

```

As you can see the second option is using something called **coRouter{...}**. This is an extension function comming from **CoRouterFunctionDsl** that registers the **route bean** object. Optionaly you can use just **router{...}** but **coRouter** supports Coroutines and enable usage of **suspend** functions.

<blockquote class="blue">

Coroutines = light-weight computations (thread like) used in Kotlin language to suspend the execution without blocking the thread

</blockquote>

**CoRouterFunctionDsl** class can be very usefull if you want to define your API endpoints in more descriptive way, if your API is complex and contains nesting routes, multiple HTTP methods or supports multiple MediaTypes.

<blockquote class="blue">

What is **ref()**? Function allowing automatical type inference to get the required bean. Under the hood is Spring ApplicationContext called

```

context.getBean(T::class.java)

```

</blockquote>

<blockquote class="red">

Using OpenAPI 3 (Swagger UI) to document your functional REST API might be tricky and amount of code to describe the endpoints might be 10x more then the code itself.

There is no out of the box solution to document it automatically. Spring Web MVC has better support in that matter.

</blockquote>

Registering route beans (or any other beans) into your Spring application context via Bean DSL has one positive impact. It doesn't require direct class reflexion or CGLIB (code generation library) proxies. This is very important aspect if you know that your application will be built and delivered as **Graal VM Native Image**. Even very powerfull PC with lots CPU cores and memory needs quite some time to create Native Image from Java/Kotlin aplication that uses reflexion and annotations. Removing this obstacle help the build process significantly.

### Learnings

As I mentioned at the beginning my old application was using JPA with blocking JDBC calls. This doesn't make much sense in combination with WebFlux. Combining WebFlux and blockin APIs in general doesn't bring any performance gain - it can work together but no efficiency is acheived with it as it breaks the basic paradigm of non-blocking approach. One blocking call can lock the whole application.

If there is no performance or scaling issue in your application then Web MVC works in most of the cases fine. Debugging, maintaining and developing reactive applications is bit harder and learning curve is steeper.

From (subjective) coding perspective I can say that defining the functional REST API endpoints with WebFlux gave me feeling that I had the code more under the controll. The code written in Kotlin was more compact and better readable in comparism to Java. Kotlin integration within the Spring Boot is very well done and there are lots of "syntactic sugar" functions that boost up the productiviy when writing the code.

---

# From JPA to R2DBC

In previous article I've mentioned that combination of of WebFlux and JPA doesn't make much sense as the underlying calls in JPA are done via **blocking JDBC calls**. In order to make my application fully reactive and fully use the potential I've decided to migrate the JPA part to **R2DBC**.

### R2DBC - Reactive Relational Database Connectivity

R2DBC is a specification for reactive SQL database drivers where the main focus lays on the non-blocking reactive approach.

Integration of R2DBC into Spring framework has been here for a while already. First vendors' support was done mostly for the NoSQL databases (Mongo, Cassandra). Nowadays R2DBC is available for relational databases as well: **MySQL**, **PostgreSQL** and of course **H2** (used for the test), to name just some. Spring R2DBC is a part of the **Spring Data** family that makes it easy for developers to work with persistence.

```kotlin

implementation("org.springframework.boot:spring-boot-starter-data-r2dbc")
implementation("org.liquibase:liquibase-core")
implementation("com.h2database:h2")
runtimeOnly("io.r2dbc:r2dbc-h2")

```

It is very easy to set a R2DBC connection up. It requires only few configuration properties, a persistence model and a repository.

```kotlin

spring.r2dbc.url=r2dbc:h2:file:///~/travel;MODE=MySQL;AUTO_SERVER=TRUE
spring.r2dbc.username=root
spring.r2dbc.password=pwd1

```

```java

@Entity
public class City {
    public City() {
    }
    @Id
    @GeneratedValue
    private Long id;
    private String name;
    @Column(name = "country")
    private String countryCode;
    @Column(name = "lat")
    private Double latitude;
    @Column(name = "lng")
    private Double longitude;
}

```

### Interaction with database

Spring Data R2DBC provides 2 basic options how to communicate with the database

<div class="list-margin">

- Using **repository** pattern
- Using **Reactive Database Client**

</div>

```kotlin

interface CityRepository : ReactiveCrudRepository<City, Long> {

}

```

The original repository file needed basicaly no change in order to make it reactive. Only changing the inheritance from **CrudRepository** to **ReactiveCrudRepository**.

```kotlin

val connectionFactory = ConnectionFactories
.get("r2dbc:h2:file:///~/travel;MODE=MySQL;AUTO_SERVER=TRUE")

val client = DatabaseClient.create(connectionFactory)

val firstCity = client
      .select()
      .from(City::class)
      .fetch()
      .first()

```

For simple usecases the repository is fully sufficient solution. Reactive database client is **thread-safe** and once it is configured it can be used across the whole spring application as singleton instance.

When more complexer SQL queries are needed then both options can be used. Either use **@Query annotation** in the repository or write query as parameter using the reactive database client.

```kotlin

data class City(
    val id: Long? = -1L,
    val country: String,
    val name: String,
    val lat: Double,
    val lng: Double
)

```

As you can see the R2DBC City model is already free of any annotation to compare the original JPA example. The main difference is that the attributes in the Kotlin R2DBC model must be exactly named as the columns in the city table if we want to use the default mapper implemented in Spring Data module.

### Liquibase migration

The original application was using Liquibase tool to create the **city table**. Nor Liquibase nor Flyaway provides R2DBC connection drivers that would allow the migrationion in non-blocking way. At this point it doesn't really matter as Liquibase is executing the blocking call only once before the full Spring application context is initilized.

```

-- liquibase formatted sql
-- changeset frantisek.schneider:mun-000-init-schemas

create table if not exists city
(
    id      bigint          not null primary key auto_increment,
    country varchar(50)     not null,
    name    varchar(200)    not null,
    lat     decimal(60, 30) not null,
    lng     decimal(60, 30) not null
);

```

I've found out 2 options how to execute the Liquibase migration ([SQL DDL](https://docs.oracle.com/cd/B14117_01/server.101/b10759/statements_1001.htm#i2099120) script)

<div class="list-margin">

- Using the **@Configuration** annotation
- Using **custom context initialzier**

</div>

```kotlin

@Configuration
@PropertySource("classpath:liquibase.properties")
class LiquibaseConfiguration {

    @Value("\${liquibase.changeLog}")
    private lateinit var changeLog: String

    @Value("\${liquibase.defaultSchema}")
    private lateinit var defaultSchema: String

    @Value("\${liquibase.datasource.url}")
    private lateinit var url: String

    @Value("\${liquibase.datasource.driverClassName}")
    private lateinit var driverClassName: String

    @Value("\${liquibase.datasource.password}")
    private lateinit var password: String

    @Value("\${liquibase.datasource.username}")
    private lateinit var username: String

    private val springLiquibase: SpringLiquibase = SpringLiquibase()

    @PostConstruct
    fun initDB() {
       with(springLiquibase) {
           changeLog = this@DatabaseSchemaMigration.changeLog
           defaultSchema = this@DatabaseSchemaMigration.defaultSchema
           dataSource = dataSource()
           resourceLoader = DefaultResourceLoader()
       }
       springLiquibase.afterPropertiesSet()
    }

    private fun dataSource(): DataSource = with(DataSourceBuilder.create()) {
        driverClassName(this@DatabaseSchemaMigration.driverClassName)
        url(this@DatabaseSchemaMigration.url)
        username(this@DatabaseSchemaMigration.username)
        password(this@DatabaseSchemaMigration.password)
        type(JdbcDataSource::class.java)
        build()
    }
}

```

As you can see the whole trick is just calling the right method in the right time. This is achieved via **@PostConstruct** lifecycle hook **annotation**. Once the configuration bean was initilized then the Liquibase engine took over the control - **springLiquibase.afterPropertiesSet**.

```kotlin

class LiquibaseConfiguration :
    ApplicationContextInitializer<GenericApplicationContext> {

    private lateinit var propertySource: ResourcePropertySource

    private val springLiquibase: SpringLiquibase = SpringLiquibase()

    override fun initialize(applicationContext: GenericApplicationContext) {
        propertySource = ResourcePropertySource("classpath:liquibase.properties")

        with(springLiquibase) {
            changeLog = propertySource.get("liquibase.changeLog")
            defaultSchema = propertySource.get("liquibase.defaultSchema")
            dataSource = dataSource()
            resourceLoader = DefaultResourceLoader()
        }

        springLiquibase.afterPropertiesSet()
    }

    private fun dataSource(): DataSource = with(DataSourceBuilder.create()) {
        driverClassName(propertySource.get("liquibase.datasource.driverClassName"))
        url(propertySource.get("liquibase.datasource.url"))
        username(propertySource.get("liquibase.datasource.username"))
        password(propertySource.get("liquibase.datasource.password"))
        type(JdbcDataSource::class.java)
        build()
    }
}

private fun ResourcePropertySource.get(name: String): String {
    return this.getProperty(name) as String
}

```

The second version was smaller and more readable to me. Plus it didn't have any annotations (which I like) so I sticked with it.

<blockquote class="red">

As you could see in the connection string URL im using H2 database engine that is persisting the data into the file in filesystem. It is very important to mention that the **r2dbc-driver** runs on top of internals of H2 database engine and various parts of H2 are still **blocking** like file and network access. In memory access and all the layers above H2 are non-blocking.

</blockquote>

At this point no more work had to be done in order to migrate the JPA to R2DBC.

### Learnings

Using R2DBC together with WebFlux can be tempting as it gives certain **promise of performance for free**. We have always keep in mind that the usage must be justifiable. It wouldn't make much sense to use R2DBC if we know that concurrency aspect will be low. R2DBC performs best in high concurrency model.

When comparing to JPA we have to accept the fact that lots of code must be written by the developer - custom mappings or One-To-Many/Many-To-Many relationships. I see perfect usage for applications having simple database models and where high concurrecy aspect is expected. It is still quite young technology and when it comes to speed of development (subjective point of view) JPA and ORM frameworks can outperform the R2DBC despite being sometimes criticized for its heaviness and complexness.