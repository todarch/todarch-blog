---
title: "Application-side Id Generation"
date: 2018-10-29T16:26:49+02:00
description: "application-side id generation description"
tags: [ "uuid", "hibernate", "java", "ddd", "sequence", "postgresql" ]
categories: ["java", "ddd"]
author: selimssevgi
---

On the weekend, I needed to implement edit functionality for todo item.
Partial update, full update, transactional boundary, aggregates. And here I am
with the same problem that was bothering me for some time.
Someone please do tell me how to get my IDs without a database so I can
link my aggregates and have it there instead of magically waiting for the id to appear
somewhere in the line?  I would like to have
the id of the aggreate even before it will be persisted to the database. Every time
you read something on **DDD**, there will be "*design your models first without
thinking about database concern*" and "*use unidirectional linking*" and finally
"*use IDs instead of reference for linking*". They may not be exact words, but
same thinking.


At this point I would like to point out that I also prefer my IDs to have their
own types instead of primitives such as `Long`. For example, `TodoId` and `TagId`.

### Ask Database for nextval

`UUID` is the first thing comes to mind. However, some of us are not brave enough.
I was already convinced to use a custom sequence for IDs. Later on, I was like why
not ask for database to give me the next id so the entity can be constructed.

The following was my first approach: using `Spring Data Repository`. Java 8 default
method as helper and my sequence on database side named `taq_seq`.

```java
@Query(value = "SELECT tag_seq.nextval FROM dual", nativeQuery = true)
Long getNextSeriesId();

default TagId nextId() {
  return TagId.of(getNextSeriesId());
}
```

I even pushed this piece of code to production. Of course, after all of the
tests were green. However, when I tried to create a new todo. It failed. Failing in production is not the best thing (*going to find other means to prevent this happening soon*). The problem
was that the sequence defined under `todarch_db` schema in **PostgreSQL**.
And anyway that was not the correct way to ask for nextval for PostgreSQL.

```java
@Query(
  value = "SELECT nextval('todarch_td.\"tag_seq\"')",
  nativeQuery = true)
```

Even after the query was fixed to work for the production database. Now we have failing
tests. Because `H2` does not have `todarch_td`. Some fight with `Flyway`, H2, and
Spring configuration did not feel right or easy. Specially, when I keep looking
at the `nativeQuery = true` part. What if tomorrow we changed to MySQL(even in
our fantasies), what if we changed the schema in external configuration? Two
methods for the same operation on interface? Many annoyances with this approach so
that made me brave enough to think about the other girl, I mean approach.

### Use UUID

Another option is to use UUID. You do not have to trip to database to get a
simple value. And imagine it is not just going to be unique for the table but
universally. That has some advantages in itself. Unfortunately, some people are
so discouraging that I started to have nightmares on the way of implementing
it. Some blog posts:

- [Do not store UUID as String](https://tomharrisonjr.com/uuid-or-guid-as-primary-keys-be-careful-7b2aa3dcb439)

- [Encouraging, then referencing something that is not](https://blog.codinghorror.com/primary-keys-ids-versus-guids/)

- [How to store UUID](https://dba.stackexchange.com/questions/69254/whats-the-most-efficient-uuid-column-type)

- [Benchmarks](https://www.percona.com/blog/2014/12/19/store-uuid-optimized-way/)

- [JPA, PostgreSQL, UUID](https://medium.com/@swhp/work-with-uuid-in-jpa-and-postgresql-86a59ea989cd)

Again even though some databases support UUIDs natively, could not expect that
from all of them. That means you will not have the best support to store it.
I admit that UUID format looks cool but integer values seems to be more normal for id for me.
Another reason was that I already had some data in the database with normal
integer IDs. I would have some headache on that side (*this means that there
should be import/export functionality for the application*). What if on the way
of using it, it would cause more trouble than good? Afterall, I was not so
comfortable and brave to follow that path with the information I have gathered,
the situation I have had. Maybe next time.

### Dummy way

While searching for the options, I found it on a SO answer(could not find the link, will add when found). It was in
the early of my searching hours. And I was like "*%$x, this is a stupid way*". Then
I ended up implementing it. And for now I am happy with it. Sharing with you
all. So, you could tell me where it is wrong.

I will explain in simple steps:

1. Create a new sequence on database if you do not want to use existing one.

2. Define a new dummy entity, which has a single field annotated with @Id.

3. Whenever you need an id for the original entity, you could save the dummy
   entity, get its id, and use for the original one.

Some code may explain a bit better:

A dummy entity definition, easy to instantiate. We just need `id` field, nothing
more.

```java
@Table(name = "todo_id")
@Entity
class TodoIdEntity {
  @Id
  @GeneratedValue(
    strategy = GenerationType.SEQUENCE,
    generator = "todo_id_generator")
  @SequenceGenerator(
      name = "todo_id_generator",
      sequenceName = "todo_id_seq",
      initialValue = 300,
      allocationSize = 100)
  private Long id;

  protected TodoIdEntity() {
    // for jpa/hibernate
  }

  public Long id() {
    return id;
  }
}
```

A repository for just saving a dummy entity is an overkill but simple.

```java
@Repository
interface TodoIdRepository extends JpaRepository<TodoIdEntity, Long> {
}
```

Public facing `IdGenerator` that nobody should know what crazy work we are doing
behind the scenes.

```java
@Component
@AllArgsConstructor
public class TodoIdGenerator {

  private final TodoIdRepository todoIdRepository;

  public TodoId next() {
    return TodoId.of(todoIdRepository.save(new TodoIdEntity()).id());
  }
}
```

I created different dummy entities for business different entities. You may check out the whole change [here](https://github.com/todarch/todarch-td/pull/42/files).

The dummy entity and its repository are encapsulated to the package itself. The
only public part is the next() method. Inject the Generator and start using it.

```java
private final TodoIdGenerator todoIdGenerator;

TodoFactory.from(todoCreationCommand, todoIdGenerator.next());
```

#### Advantages

0. Had the id for the business entity *before saving it*.

1. Postponed the nightmares of *storing UUID* in a performant and vendor-independent way.

2. Did not have to write the *native query*. Hibernate will take care of that part (hopefully).

3. Did not have to make *changes on the existing data* in the database.


#### Disadvantages

0. Unnecessary *table and sequence creation*. This may not be possible in some
   setting like oh let's create a dummy table.

1. *Not %100 sure* if there will be any problem with such way of generating and
   using the id in another table. It is my lack of knowledge with all Hibernate magic.

2. There is the *extra trip* to database, and *space* to store dummy entity. I
   believe that it will not hurt to delete dummy entries at some interval.

3. *Some extra code* to write. Here Spring helps a lot. Does not even worth writing
   test or another. Encapsulation also helps in case of a change with way of
   getting the id.

For now, I am actually happy with the result. The application is up and running,
I can create new todos. I did not have to invest deeply with UUID to have IDs
before saving the entity. It fit perfectly with what already existed. The
whole point of the project (aside from building something useful) is to try out
new things actively in practice. This was a crazy idea to try. We will see how
it will work out.

You may want to join us to build the next beatiful thing. Check out our
[Github](https://github.com/todarch/)
or [application itself](https://todarch.com)

[Long UUID Generation](https://www.callicoder.com/distributed-unique-id-sequence-number-generator/)
