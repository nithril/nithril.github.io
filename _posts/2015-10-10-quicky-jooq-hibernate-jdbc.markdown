---
layout: post
title:  "Quicky: Jooq / Hibernate / JDBC"
date:   2015-10-10 10:18:46
categories: AMQP
comments: true
---




## The query

{% highlight sql linenos %}
SELECT AUTHOR.*, BOOK.* FROM AUTHOR LEFT OUTER JOIN BOOK ON AUTHOR.ID = BOOK.AUTHOR_ID
{% endhighlight %}


## Plain Old JDBC

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


## Jooq Into Group

Jooq into group function `Return a {@link Map} with the result grouped by the given key table.`.

{% highlight java linenos %}
@Transactional(readOnly = true)
public Collection<AuthorWithBooks> findAuthorsWithBooksJooqIntoGroup() {
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


## Jooq with old fashion group by

{% highlight java linenos %}
@Transactional(readOnly = true)
public Object findAuthorsWithBooksJooqOldFashionGroupBy() {

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




## JPA

{% highlight java linenos %}
private List<AuthorWithBooks> toAuthor(List<Author> authors) {
    return authors.stream()
            .distinct()
            .map(author -> new AuthorWithBooks(author, author.getBooks())).collect(Collectors.toList());
}
{% endhighlight %}


## Hibernate Named Query

{% highlight java linenos %}
@Transactional(readOnly = true)
public List<AuthorWithBooks> findAuthorsWithBooksUsingNamedQuery() {
    TypedQuery<Author> query = entityManager.createNamedQuery("Author.findAllWithBooks", Author.class);
    return toAuthor(query.getResultList());
}
{% endhighlight %}

{% highlight java linenos %}
@NamedQueries(
		@NamedQuery(name = "Author.findAllWithBooks" , query = "FROM Author a LEFT JOIN FETCH a.books")
)
{% endhighlight %}

## Spring Data

{% highlight java linenos %}
@Transactional(readOnly = true)
public List<AuthorWithBooks> findAuthorsWithBooksUsingSpringData() {
    return toAuthor(authorRepository.findAllWithBooks());
}
{% endhighlight %}



| Qantum  | Consumed  | Rate      |
|:-------:|-----------|-----------|
| Plain Jdbc                   | 11887.212    | 775.680 |
| Hibernate Named Query        | 1015.088     | 16.014  |
| Hibernate Spring Data        | 1017.145     | 17.038  |
| Jooq IntoGroup               | 1186.168     | 38.805  |
| Jooq Old Fashioned GroupBy   | 3217.562     | 132.897 |

