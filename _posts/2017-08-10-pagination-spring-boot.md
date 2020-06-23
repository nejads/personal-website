---
layout: page
title: Pagination in Spring Boot Applications
permalink: /pagination-spring-boot/
---

# Paging

Paging in web applications is a mechanism to separate a big resultset to smaller chunks. Providing a fluent paging navigator could increase both the value of your website for search engines and enhance user experience through minimizing the response time. The famous example of paging is google's search result page. Normally, the result page is separated into several pages. To avoid the bad user experience mentioned before, a lot of sites show only the current, the first, the last and some adjacent pages.
To implement paging in Spring framework we can choose different alternatives. The Spring framework provides an out-of-the-box feature for paging which needs the number of pages and the number of elements per page. This is very useful when we would implement a "static paging" which can offer a user to choose between different pages.
How can we implement a dynamic paging which offers automatically loading the content of a page as we scroll down? In this article, I will present both static and dynamic paging and different ways to implement such services.

#### Static paging with SpringData
This is the most convenient way to implement paging in a web application. It only needs to get page and number of result per page for a query. Instead of using Repository or CrudRepository you should use PagingAndSortingRepository which accept an object of "Pageable" type.
The Pageable object needs a number of page and number of the element per page. This attributes of a pageable object will return a different result in a query. The result is a "page" which has the element from specific rows of the whole resultset. The response also includes metadata of total element of the specific query you sent in and total pages. This metadata information could be useful for frontend and to build a UI including links to previous and next pages. In the following link, you can see the result of such approach of paging: http://softwareengineering.stackexchange.com/search?tab=relevance&pagesize=50&q=excel%20

##### How to implement static paging:
Here you can find sample code of repository and service layer which give back a paged result from the database. If you turn on the debugging logs, [either in application.properties or in logback file](https://docs.spring.io/spring-boot/docs/current/reference/html/howto-logging.html),  you can see the queries which will be created by Spring data.


```sh
@Repository
public interface SomethingRepository extends PagingAndSortingRepository<Something, Long> {
    @Query("Select s from  Something s "
            + "join s.somethingelse as se "
            + "where se.id = :somethingelseid ")
    Page<Something> findBySomethingElseId(@Param("somethingelseid") long somethingelseid,
                                                                        Pageable pageable);

```

```sh
@Service
public class SomethingService {

    private SomethingRepository somethingRepository;

    @Autowired
    public SomethingService(SomethingRepository somethingRepository){
        this.somethingRepository = somethingRepository;
    }

    @Transactional(readOnly=true)
    public PageDto getSomething(long somethingElseId, int page, int size){
         Page<Something> somethings = somethingRepository.findBySomethingElseId(somethingElseId, new PageResult(page, size));

        return new PageDto(somethings.getContent()
                .stream()
                .map(SomethingDto::createDto)
                .sorted(comparing(SomethingDto::getDatum))
                .collect(toList()), somethings.getTotalElements(), somethings.getTotalPages();
    }
}
```

```sh
@Controller
//....
```
#### Dynamic Paging with nativeQuery
As it was mentioned before, another face of paging is dynamic paging. This kind of paging can be done with a native query in Spring framework as following code sample shows. For this kind, you need to specify an offset row and a limit which is the number of elements after this offset. You need also write your "page" class and map the response list to it.

##### How to implement dynamic paging using nativeQuery:
Here you can find repository and service layers and your data transfer object(DTO) which will be used for mapping our result and send it to the controller layer.
```sh
public interface CustomSomethingRepository {
    List<Something> findPagedResultBySomethingElseId(long somethingElseId, int offset, int limit);
}
```
```sh
public class SomethingRepositoryImpl implements CustomSomethingRepository {
    @Autowired
    private EntityManager em;


    @SuppressWarnings("unchecked")
    @Override
    public List<Something> findPagedResultBySomethingElseId(long somethingElseId, int offset, int limit) {

        String query = "select s.* from Something s "
                + "join somethingelse selse on selse.id = s.fk_somethingelse "
                + "where selse.id = :somethingElseId "
                + "order by selse.date";

        Query nativeQuery = em.createNativeQuery(query);
        nativeQuery.setParameter("somethingElseId", somethingElseId);

        //Paginering
        nativeQuery.setFirstResult(offset);
        nativeQuery.setMaxResults(limit);

        final List<Object[]> resultList = nativeQuery.getResultList();
        List<Something> somethingList = Lists.newArrayList();
        resultList.forEach(object -> somethingList.add(//map obj to something));
        return somethingList;
    }
}
```
Hibernate translate your query as follows:
```sh
    SELECT inner_query.*, ROW_NUMBER() OVER (ORDER BY CURRENT_TIMESTAMP) as __hibernate_row_nr__ FROM ( select TOP(?) t as page0_ from Something s join s.somethingelse as selse order by selse.date ) inner_query ) SELECT page0_ FROM query WHERE __hibernate_row_nr__ >= ? AND __hibernate_row_nr__ < ?
```

```sh
@Service
public class SomethingService {

    private SomethingRepository somethingRepository;

    @Autowired
    public SomethingService(SomethingRepository somethingRepository){
        this.somethingRepository = somethingRepository;
    }

    @Transactional(readOnly=true)
    public PageDto getSomething(long somethingElseId, int page, int size){
         List<Something> somethings = somethingRepository.findBySomethingElseId(somethingElseId, offset, limit);

        return new PagedResult<>(somethings
                .stream()
                .map(SomethingDto::createDto)
                .sorted(comparing(SomethingDto::getDatum))
                .collect(toList()), somethings.getTotalElements(), somethings.getTotalPages();
    }
}
```
```sh
@Controller
//....
```

```sh
public class PagedResult<T> {
    public static final long DEFAULT_OFFSET = 0;
    public static final int DEFAULT_MAX_NO_OF_ROWS = 100;

    private int offset;
    private int limit;
    private long totalElements;
    private List<T> elements;

    public PagedResult(List<T> elements, long totalElements, int offset, int limit) {
        this.elements = elements;
        this.totalElements = totalElements;
        this.offset = offset;
        this.limit = limit;
    }


    public boolean hasMore() {
        return totalElements > offset + limit;
    }

    public boolean hasPrevious() {
        return offset > 0 && totalElements > 0;
    }

    public long getTotalElements() {
        return totalElements;
    }

    public int  getOffset() {
        return offset;
    }

    public int getLimit() {
        return limit;
    }

    public List<T> getElements() {
        return elements;
    }
}
```

#### Pros and Cons:
*Pros:* Fewer SQL queries will be generated in comparison with using Spring data. The complex queries can not be written in Spring data and we have to specify our query as a native one can still be paged by using this methodology.
cons: The "object" array must map to a java object. It is painful and hard to maintain.

#### How to implement OffsetLimit paging with Spring data?
As far as I know, there is no "out-of-the-box" support for what you need in default SrpingData repositories. But you can create a custom implementation of Pageable objects, that will take limit/offset parameters.

Make a pageable object and pass it to PagingAndSortingRepository
```sh
public class OffsetLimitRequest implements Pageable {
    private int limit;
    private int offset;

    public OffsetLimitRequest(int offset, int limit){
        this.limit = limit;
        this.offset = offset;
    }
        @Override
    public int getPageNumber() {
        return 0;
    }

    @Override
    public int getPageSize() {
        return limit;
    }

    @Override
    public int getOffset() {
        return offset;
    }
    ....
}
```

It means there is no need to change the repository layer. The only change you would need is to change service layer as follows:

```sh
@Service
public class SomethingService {
    private SomethingRepository somethingRepository;

    @Autowired
    public SomethingService(SomethingRepository somethingRepository){
        this.somethingRepository = somethingRepository;
    }

    @Transactional(readOnly=true)
    public PageDto getSomething(long somethingElseId, int page, int size){
        Page<Something> somethings = somethingRepository.findBySomethingElseId(somethingElseId, new OffsetLimitRequest(offset, limit));
        return new PageDto(somethings.getContent()
                .stream()
                .map(SomethingDto::createDto)
                .sorted(comparing(SomethingDto::getDatum))
                .collect(toList()), somethings.getTotalElements(), somethings.getTotalPages();
    }
}
```

Note that you don't need to map the result manually and it will decrease a good amount of time of the development.

### Sort the result
As you can see in the code snippet above, you can sort the result based on properties using stream interface in Java8. You can also develop your pageable object with a "sort" object argument from the package "org.springframework.data.domain". A sort object can be initiated with a Direction which is ASC or DESC and an iterable object of your property names of your entity. Our previous OffsetLimitRequest class changed as follows:

```sh
public class OffsetLimitRequest implements Pageable, Serializable {
    private static final long serialVersionUID = -4541509938956089562L;

    private int limit;
    private int offset;
    private Sort sort;

    public OffsetLimitRequest(int offset, int limit, Sort sort){
        this.limit = limit;
        this.offset = offset;
        this.sort = sort;
    }


    @Override
    public int getPageNumber() {
        return 0;
    }

    @Override
    public int getPageSize() {
        return limit;
    }

    @Override
    public int getOffset() {
        return offset;
    }

    @Override
    public Sort getSort() {
        return sort;
    }
    ....
}
```
which can be initated in our service layer:
```sh
@Service
public class SomethingService {
    private SomethingRepository somethingRepository;

    @Autowired
    public SomethingService(SomethingRepository somethingRepository) {
        this.somethingRepository = somethingRepository;
    }

    @Transactional(readOnly=true)
    public PageDto getSomething(long somethingElseId, int page, int size) {
        List<String> sortingOnSomethingsAttributes = Arrays.asList("attr1", "attr2");
        Page<Something> somethings = somethingRepository.findBySomethingElseId(somethingElseId, new OffsetLimitRequest(offset, limit, new Sort(Sort.Direction.fromString("ASC"), sortingOnSomethingsAttributes));
        return new PageDto(somethings.getContent()
                .stream()
                .map(SomethingDto::createDto)
                .collect(toList()), somethings.getTotalElements(), somethings.getTotalPages();
    }
}
```
Note that we don't need sorted in the stream the return statement. When we handle big result sets It is better to have sorting functions near the database to get better performance.

#### Link on DZone
You can find [this post on DZone](https://dzone.com/articles/pagination-in-springboot-applications)

#### Useful links:
* [Spring data reference](http://docs.spring.io/spring-data/rest/docs/current/reference/html/)
* [The art of pagination](http://blog.novatec-gmbh.de/art-pagination-offset-vs-value-based-paging/)
* [Paging with Spring MVC and Spring Data JPA](https://blog.zenika.com/2012/06/15/hateoas-paging-with-spring-mvc-and-spring-data-jpa/)
* [REST Pagination in Spring](https://dzone.com/articles/rest-pagination-spring)
