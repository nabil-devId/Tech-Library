# Spring HATEOAS: Complete Guide

## What is HATEOAS?

**HATEOAS** (Hypermedia as the Engine of Application State) is a constraint of the REST architectural style that makes a RESTful API truly self-descriptive and discoverable.

In a HATEOAS-driven API:
- Each response includes not just data, but also links to related resources
- Clients navigate the API by following these links, rather than constructing URLs
- The API becomes self-documenting and evolves without breaking clients

## Key Benefits of HATEOAS

- **Loose Coupling**: Clients don't need hardcoded knowledge of endpoint URLs
- **Evolvability**: The API can change without breaking clients
- **Discoverability**: Clients can explore available operations
- **Self-Documentation**: The API explains itself through hypermedia

## Spring HATEOAS Components

### Core Classes

| Class | Purpose |
|-------|---------|
| `EntityModel<T>` | Wraps a domain object and adds links to it |
| `CollectionModel<T>` | Wraps a collection of resources and adds links to the collection |
| `PagedModel<T>` | Wraps a paged collection with pagination links and metadata |
| `RepresentationModelAssembler<T,R>` | Converts domain objects to resources with links |
| `WebMvcLinkBuilder` | Builds links pointing to Spring MVC controllers |

### Link Building

```java
// Static imports to make link building more readable
import static org.springframework.hateoas.server.mvc.WebMvcLinkBuilder.*;

// Building a link to a controller method
Link selfLink = linkTo(methodOn(BookController.class).getBookById(book.getId())).withSelfRel();

// Adding multiple links to a resource
return EntityModel.of(book,
    linkTo(methodOn(BookController.class).getBookById(book.getId())).withSelfRel(),
    linkTo(methodOn(BookController.class).getAllBooks()).withRel("books")
);
```

## Implementation Examples

### Basic Resource

```java
@RestController
@RequestMapping("/api/books")
public class BookController {

    private final BookRepository repository;
    
    @GetMapping("/{id}")
    public EntityModel<Book> getBook(@PathVariable Long id) {
        Book book = repository.findById(id)
                .orElseThrow(() -> new BookNotFoundException(id));
                
        return EntityModel.of(book,
            linkTo(methodOn(BookController.class).getBook(id)).withSelfRel(),
            linkTo(methodOn(BookController.class).getAllBooks()).withRel("books")
        );
    }
}
```

### Resource Assembler Pattern

```java
@Component
public class BookModelAssembler implements RepresentationModelAssembler<Book, EntityModel<Book>> {

    @Override
    public EntityModel<Book> toModel(Book book) {
        return EntityModel.of(book,
            linkTo(methodOn(BookController.class).getBook(book.getId())).withSelfRel(),
            linkTo(methodOn(BookController.class).getAllBooks()).withRel("books"),
            linkTo(methodOn(BookController.class).getAuthor(book.getAuthorId())).withRel("author")
        );
    }
}
```

### Collection Resources

```java
@GetMapping
public CollectionModel<EntityModel<Book>> getAllBooks() {
    List<EntityModel<Book>> books = repository.findAll().stream()
            .map(bookAssembler::toModel)
            .collect(Collectors.toList());
            
    return CollectionModel.of(books,
        linkTo(methodOn(BookController.class).getAllBooks()).withSelfRel()
    );
}
```

### Pagination with HATEOAS

```java
@GetMapping
public ResponseEntity<PagedModel<EntityModel<Book>>> getAllBooks(
        @PageableDefault(size = 10) Pageable pageable) {
    
    Page<Book> bookPage = repository.findAll(pageable);
    
    PagedModel<EntityModel<Book>> pagedModel = 
        pagedResourcesAssembler.toModel(bookPage, bookAssembler);
    
    return ResponseEntity.ok(pagedModel);
}
```

## Complete Pagination Example

### Controller Implementation

```java
@RestController
@RequestMapping("/api/books")
public class BookController {

    private final BookRepository repository;
    private final BookModelAssembler assembler;
    private final PagedResourcesAssembler<Book> pagedResourcesAssembler;

    public BookController(
            BookRepository repository,
            BookModelAssembler assembler,
            PagedResourcesAssembler<Book> pagedResourcesAssembler) {
        this.repository = repository;
        this.assembler = assembler;
        this.pagedResourcesAssembler = pagedResourcesAssembler;
    }

    @GetMapping
    public ResponseEntity<PagedModel<EntityModel<Book>>> getAllBooks(
            @PageableDefault(size = 10, sort = "title") Pageable pageable,
            ServerHttpRequest request) {
        
        Page<Book> bookPage = repository.findAll(pageable);
        
        // Create a self link that preserves any query parameters
        Link selfLink = Link.of(request.getURI().toString()).withSelfRel();
        
        PagedModel<EntityModel<Book>> pagedModel = 
            pagedResourcesAssembler.toModel(bookPage, assembler, selfLink);
        
        // Add additional links
        pagedModel.add(
            linkTo(methodOn(BookController.class).createBook(null)).withRel("create-book"),
            linkTo(methodOn(CategoryController.class).getAllCategories()).withRel("categories")
        );
        
        return ResponseEntity.ok(pagedModel);
    }
    
    // Other controller methods...
}
```

### Assembler Implementation

```java
@Component
public class BookModelAssembler implements RepresentationModelAssembler<Book, EntityModel<Book>> {

    @Override
    public EntityModel<Book> toModel(Book book) {
        return EntityModel.of(book,
            linkTo(methodOn(BookController.class).getBookById(book.getId())).withSelfRel(),
            linkTo(methodOn(BookController.class).updateBook(book.getId(), null)).withRel("update"),
            linkTo(methodOn(BookController.class).deleteBook(book.getId())).withRel("delete"),
            linkTo(methodOn(AuthorController.class).getAuthor(book.getAuthorId())).withRel("author"),
            linkTo(methodOn(ReviewController.class).getReviewsForBook(book.getId(), null))
                .withRel("reviews")
        );
    }
}
```

## Example Response

```json
{
  "_embedded": {
    "books": [
      {
        "id": 1,
        "title": "Spring Boot in Action",
        "author": "Craig Walls",
        "isbn": "978-1617292545",
        "price": 39.99,
        "_links": {
          "self": { "href": "http://localhost:8080/api/books/1" },
          "update": { "href": "http://localhost:8080/api/books/1" },
          "delete": { "href": "http://localhost:8080/api/books/1" },
          "author": { "href": "http://localhost:8080/api/authors/5" },
          "reviews": { "href": "http://localhost:8080/api/books/1/reviews" }
        }
      },
      // More books...
    ]
  },
  "_links": {
    "first": { "href": "http://localhost:8080/api/books?page=0&size=10&sort=title,asc" },
    "self": { "href": "http://localhost:8080/api/books?page=0&size=10&sort=title,asc" },
    "next": { "href": "http://localhost:8080/api/books?page=1&size=10&sort=title,asc" },
    "last": { "href": "http://localhost:8080/api/books?page=5&size=10&sort=title,asc" },
    "create-book": { "href": "http://localhost:8080/api/books" },
    "categories": { "href": "http://localhost:8080/api/categories" }
  },
  "page": {
    "size": 10,
    "totalElements": 53,
    "totalPages": 6,
    "number": 0
  }
}
```

## Advanced Topics

### Custom Link Relations

```java
// Using standard IANA link relations
import org.springframework.hateoas.IanaLinkRelations;

entityModel.add(
    linkTo(methodOn(BookController.class).getBookById(id)).withRel(IanaLinkRelations.SELF),
    linkTo(methodOn(BookController.class).getAllBooks(null)).withRel(IanaLinkRelations.COLLECTION)
);

// Custom link relations
Link customLink = linkTo(methodOn(BookController.class).getRelatedBooks(id))
    .withRel("related-books");
```

### Affordances (Actions on Resources)

```java
import static org.springframework.hateoas.server.mvc.WebMvcLinkBuilder.afford;

EntityModel<Book> bookModel = EntityModel.of(book,
    linkTo(methodOn(BookController.class).getBookById(book.getId())).withSelfRel()
        .andAffordance(afford(methodOn(BookController.class).updateBook(book.getId(), null)))
        .andAffordance(afford(methodOn(BookController.class).deleteBook(book.getId())))
);
```

### Link Templates

```java
UriTemplate template = UriTemplate.of("/api/books{?title,author,category}");
Link searchLink = Link.of(template).withRel("search");
```

## Best Practices

1. **Use Resource Assemblers**: Create dedicated classes for converting domain objects to resources
2. **Consistent Link Relations**: Use standard IANA link relations where possible
3. **Context-Sensitive Links**: Only include links that are relevant to the current state
4. **Include Pagination Information**: When returning collections, include pagination metadata
5. **Preserve Query Parameters**: When generating self links, preserve existing query parameters
6. **Document Link Relations**: Clearly document the link relations your API uses
7. **Use HAL Format**: Spring HATEOAS uses HAL by default which is widely supported

## Configuration

### Maven Dependencies

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-web</artifactId>
</dependency>
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-hateoas</artifactId>
</dependency>
```

### Spring Boot Configuration

Most configuration is automatic with Spring Boot, but you can customize:

```properties
# Customize pagination parameters in application.properties
spring.data.web.pageable.page-parameter=pageNumber
spring.data.web.pageable.size-parameter=pageSize
spring.data.web.pageable.default-page-size=25
spring.data.web.pageable.max-page-size=100
spring.data.web.sort.sort-parameter=sortBy
```

## Common Issues and Solutions

### "Cannot Resolve Method" in linkTo(methodOn())

**Problem**: The IDE shows an error in `linkTo(methodOn(BookController.class).getBook(id))`

**Solution**: Add static imports:
```java
import static org.springframework.hateoas.server.mvc.WebMvcLinkBuilder.linkTo;
import static org.springframework.hateoas.server.mvc.WebMvcLinkBuilder.methodOn;
```

### Circular References in JSON Response

**Problem**: Entities with bidirectional relationships cause infinite recursion

**Solution**: Use DTOs or `@JsonIgnore` on back-references:
```java
@JsonIgnore
private Author author;
```

### Missing Pagination Links

**Problem**: Pagination links (next, prev) are not appearing

**Solution**: Check if your repository method returns `Page<T>` not `List<T>`:
```java
Page<Book> findByCategory(String category, Pageable pageable);
```

### Links Generated with Wrong Base URL

**Problem**: Links contain localhost instead of your domain

**Solution**: Configure a base URL in application.properties:
```properties
spring.hateoas.server-base-url=https://api.example.com
```

## References

1. [Spring HATEOAS Reference Documentation](https://docs.spring.io/spring-hateoas/docs/current/reference/html/)
2. [RESTful Web Services with Spring HATEOAS](https://spring.io/guides/gs/rest-hateoas/)
3. [Spring Data REST - Reference Documentation](https://docs.spring.io/spring-data/rest/docs/current/reference/html/)
4. [Understanding HATEOAS](https://spring.io/understanding/HATEOAS)
5. [Richardson Maturity Model](https://martinfowler.com/articles/richardsonMaturityModel.html)