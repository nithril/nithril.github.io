---
layout: post
title:  "Quicky: jOOQ / Hibernate / JDBC"
date:   2015-10-10 10:18:46
categories: AMQP
comments: true
---

This quicky tests how jOOQ, Hibernate and JDBC perform against each other on a simple query / scenario:
- Plain old SQL
- jOOQ
- Hibernate Named Query
- Spring Data



<!--more-->


# The database

The database is H2 1.4.188. The DB schema contains an `AUTHOR` table with a one to many relation to a `BOOK` table. For simplicity, an author has at least one book.

The query involves a left outer join on `BOOK` from `AUTHOR`.

{% highlight sql linenos %}
SELECT AUTHOR.*, BOOK.* FROM AUTHOR LEFT OUTER JOIN BOOK ON AUTHOR.ID = BOOK.AUTHOR_ID
{% endhighlight %}

All query must returns a POJO containing the author associated to its books

{% highlight java linenos %}
public class AuthorWithBooks {
	private Author author;
	private List<Book> books = Collections.emptyList();
}
{% endhighlight %}

The DB contains 100 authors with a mean of 5 books per author.



# JDBC / jOOQ

## Plain Old JDBC

The mapping is done by hand

{% highlight java linenos %}
@Transactional(readOnly = true)
public Collection<AuthorWithBooks> findAuthorsWithBooksJdbc() {
    Map<Long, AuthorWithBooks> booksMap = new HashMap<>();
    jdbcTemplate.query("SELECT AUTHOR.*, BOOK.* FROM AUTHOR LEFT OUTER JOIN BOOK ON AUTHOR.ID = BOOK.AUTHOR_ID", r -> {
        Long authorId = r.getLong("AUTHOR.ID");
        AuthorWithBooks authorWithBooks = booksMap.get(authorId);
        if (authorWithBooks == null) {
            authorWithBooks = new AuthorWithBooks();
            authorWithBooks.setAuthor(new Author(authorId, r.getString("AUTHOR.NAME")));
            authorWithBooks.setBooks(new ArrayList<>());
            booksMap.put(authorId, authorWithBooks);
        }
        Book book = new Book(r.getLong("BOOK.ID"), r.getString("BOOK.TITLE"), authorId);
        authorWithBooks.getBooks().add(book);

    });
    return booksMap.values();
}
{% endhighlight %}


## jOOQ Into Group

jOOQ `intoGroups` function return a {@link Map} with the result grouped by the given key table (here Author).
The returned map contains instances of [Record](http://www.jOOQ.org/javadoc/3.7.x/org/jOOQ/Record.html),
a database result record which is not a pojo but an array of object wrapped into an adapter class. `Record` instance are converted to POJO
using the jOOQ [RecordMapper](http://www.jOOQ.org/javadoc/3.7.x/index.html?org/jOOQ/RecordMapper.html).

{% highlight java linenos %}
@Transactional(readOnly = true)
public Collection<AuthorWithBooks> findAuthorsWithBooksjOOQIntoGroup() {
    return dslContext.select()
            .from(AUTHOR.leftOuterJoin(BOOK).on(BOOK.AUTHOR_ID.equal(AUTHOR.ID)))
            .fetch().intoGroups(TAuthor.AUTHOR)
            .entrySet()
            .stream()
            .map(e -> {
                Author author = authorRepository.mapper().map(e.getKey());
                List<Book> stream = e.getValue().stream()
                        .map(r -> bookRepository.mapper().map(r.into(TBook.BOOK))).collect(Collectors.toList());
                return new AuthorWithBooks(author, stream);
            }).collect(Collectors.toList());
}
{% endhighlight %}


## jOOQ with hand made group by / mapping

This function will allow to test the cost of jOOQ groupBy and mapper:

{% highlight java linenos %}
@Transactional(readOnly = true)
public Collection<AuthorWithBooks> findAuthorsWithBooksjOOQOldFashionGroupBy() {

    Result<Record> records = dslContext.select()
            .from(AUTHOR.leftOuterJoin(BOOK).on(BOOK.AUTHOR_ID.equal(AUTHOR.ID)))
            .fetch();
    Map<Long, AuthorWithBooks> booksMap = new HashMap<>(records.size() / 4);
    records.stream()
            .forEach(r -> {
                Long authorId = r.getValue(TAuthor.AUTHOR.ID);
                AuthorWithBooks authorWithBooks = booksMap.get(authorId);
                if (authorWithBooks == null) {
                    authorWithBooks = new AuthorWithBooks();
                    authorWithBooks.setAuthor(new Author(authorId, r.getValue(TAuthor.AUTHOR.NAME)));
                    authorWithBooks.setBooks(new ArrayList<>());
                    booksMap.put(authorId, authorWithBooks);
                }
                Book book = new Book(r.getValue(TBook.BOOK.ID), r.getValue(TBook.BOOK.TITLE), authorId);
                authorWithBooks.getBooks().add(book);
            });
    return booksMap.values();
}
{% endhighlight %}



# JPA

All JPQ queries use this function to transform a list of duplicated list of `Author` to a list of distinct `AuthorWithBooks`:

{% highlight java linenos %}
private List<AuthorWithBooks> toAuthor(List<Author> authors) {
    return authors.stream()
            .distinct()
            .map(author -> new AuthorWithBooks(author, author.getBooks())).collect(Collectors.toList());
}
{% endhighlight %}


## Hibernate Named Query

The named query set on `Author` entity:

{% highlight java linenos %}
@NamedQueries(
		@NamedQuery(name = "Author.findAllWithBooks" , query = "FROM Author a LEFT JOIN FETCH a.books")
)
{% endhighlight %}

The associated query:

{% highlight java linenos %}
@Transactional(readOnly = true)
public List<AuthorWithBooks> findAuthorsWithBooksUsingNamedQuery() {
    TypedQuery<Author> query = entityManager.createNamedQuery("Author.findAllWithBooks", Author.class);
    return toAuthor(query.getResultList());
}
{% endhighlight %}



## Spring Data

The method from the repository interface:

{% highlight java linenos %}
@Query("FROM Author a LEFT JOIN FETCH a.books")
List<Author> findAllWithBooks();
{% endhighlight %}


{% highlight java linenos %}
@Transactional(readOnly = true)
public List<AuthorWithBooks> findAuthorsWithBooksUsingSpringData() {
    return toAuthor(authorRepository.findAllWithBooks());
}
{% endhighlight %}


# Results


| Scenario  | ops/s   | Error      |
|-----------|-----------|-----------|
| Plain Jdbc                   | 11887.212    | 775.680 |
| Hibernate Named Query        | 1015.088     | 16.014  |
| Hibernate Spring Data        | 1017.145     | 17.038  |
| jOOQ IntoGroup               | 1186.168     | 38.805  |
| jOOQ Old Fashioned GroupBy   | 3217.562     | 132.897 |

