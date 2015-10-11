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
{% highlight java linenos %}


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
{% highlight java linenos %}





| Qantum  | Consumed  | Rate      |
|:-------:|-----------|-----------|
| Plain Jdbc                   | 11887.212    | 775.680 |
| Jooq IntoGroup               | 1186.168     | 38.805  |
| Jooq Old Fashioned GroupBy   | 3217.562     | 132.897 |
| Jooq Streamed GroupBy        | 1323.197     | 30.200  |
| Hibernate Criteria           | 974.292      | 23.291  |
| Hibernate Named Query        | 1015.088     | 16.014  |
| Hibernate Spring Data        | 1017.145     | 17.038  |
