# Calling Oracle Stored Procedures And Function From Spring Data JPA

Created by: Jap Purohit
Created time: March 21, 2023 9:14 AM
Last edited by: Jap Purohit
Last edited time: March 21, 2023 9:15 AM

Consider the following stored procedure:

```sql
CREATE OR REPLACE PACKAGE test_pkg AS
   PROCEDURE in_only_test (inParam1 IN VARCHAR2);
   PROCEDURE in_and_out_test (inParam1 IN VARCHAR2, outParam1 OUT VARCHAR2);
END test_pkg;
/

CREATE OR REPLACE PACKAGE BODY test_pkg AS
   PROCEDURE in_only_test(inParam1 IN VARCHAR2) AS
   BEGIN
      DBMS_OUTPUT.PUT_LINE('in_only_test');
   END in_only_test;

   PROCEDURE in_and_ou
```

Here we have two different procedures:

- in_only_test - takes an input parameter(inParam1), but doesn't return a value.
- in_and_out_test - takes an input parameter(inParam1), and returns a value(outParam1).

We can then call the stored procedures using the @NamedStoredProcedureQueries annotation:

```java
@Entity
@Table(name = "MYTABLE")
@NamedStoredProcedureQueries({
   @NamedStoredProcedureQuery(name = "in_only_test", 
                              procedureName = "test_pkg.in_only_test",
                              parameters = {
                                 @StoredProcedureParameter(mode = ParameterMode.IN, name = "inParam1", type = String.class)
                              }),
   @NamedStoredProcedureQuery(name = "in_and_out_test", 
                              procedureName = "test_pkg.in_and_out_test",
                              parameters = {
                                 @StoredProcedureParameter(mode = ParameterMode.IN, name = "inParam1", type = String.class),
                                 @StoredProcedureParameter(mode = ParameterMode.OUT, name = "outParam1", type = String.class)
                              })
})
public class MyTable implements Serializable {
}
```

The key points are:

- The Stored Procedure uses the annotation @NamedStoredProcedureQuery and is bound to a JPA table.
- procedureName – This is the name of the stored procedure.
- name – This is the name of the StoredProcedure in the JPA ecosystem.
- Define the IN/OUT parameter using @StoredProcedureParameter.

We then create the Spring Data JPA repository:

```java
public interface MyTableRepository extends CrudRepository<MyTable, Long> {
    @Procedure(name = "in_only_test")
    void inOnlyTest(@Param("inParam1") String inParam1);

    @Procedure(name = "in_and_out_test")
    String inAndOutTest(@Param("inParam1") String inParam1);
 }
```

The key points are:

- @Procedure – the name parameter must match the name on @NamedStoredProcedureQuery
- @Param – Must match @StoredProcedureParameter name parameter
- Return types must match - so in_only_test is void, and in_and_out_test returns String

We can then call them:

```jsx
// This version shows how a param can go in an be returned from a stored procedure
 String inParam = "Hi Im an inputParam";
 String outParam = myTableRepository.inAndOutTest(inParam);
 Assert.assertEquals(outParam, "Woohoo Im an outparam, and this is my inparam Hi Im an inputParam");

 // This version shows how to call a Stored Procedure which doesnt return any parameter -
 myTableRepository.inOnlyTest(inParam);
```

## Other Tricks

The wide range of possiblities for stored procedures has resulted in a few occasions when the above approach hasnt worked. I've solved these problems by defining a custom repository to call the stored procedures as a native query.

This is done by defining a custom repository:

```java
public interface MyTableRepositoryCustom {
    void inOnlyTest(String inParam1);
 }
```

We then make sure our main repository extends this interface:

```java
public interface MyTableRepository extends CrudRepository<MyTable, Long>, MyTableRepositoryCustom {
 }
```

We then create our custom repository implementation:

```java
public class MyTableRepositoryImpl implements MyTableRepositoryCustom {

   @PersistenceContext
   private EntityManager em;

   @Override
   public void inOnlyTest(String inParam1) {
      this.em.createNativeQuery("BEGIN in_only_test(:inParam1); END;")
          .setParameter("inParam1", inParam1)
          .executeUpdate();
   }

}
```

This can then be called in the normal way:

```java
@Autowired
MyTableRepository myTableRepository;

// And to call the method -
myTableRepository.inOnlyTest(inParam1);
```

# For Function Calling

You can call your function via native query and get result from dual.

```java
public interface HelloWorldRepository extends JpaRepository<HelloWorld, Long> {
    @Query(nativeQuery = true, value = "SELECT PKG_TEST.HELLO_WORLD(:text) FROM dual")
    String callHelloWorld(@Param("text") String text);

}
```

Note that it won't work if your function is using DML statements. In this case you'll need to use `@Modifying` annotation over query, but then the function itself must return number due to `@Modifying` return type restrictions.

You can also implement your `CustomRepository`and use `SimpleJdbcCall`:

```java
import org.springframework.jdbc.core.JdbcTemplate;
import org.springframework.jdbc.core.namedparam.MapSqlParameterSource;
import org.springframework.jdbc.core.namedparam.SqlParameterSource;
import org.springframework.jdbc.core.simple.SimpleJdbcCall;
import org.springframework.stereotype.Repository;

@Repository
public class HelloWorldRepositoryImpl implements HelloWorldRepositoryCustom {

    @Autowired
    private JdbcTemplate jdbcTemplate;

    @Override
    public String callHelloWorld() {
        SimpleJdbcCall jdbcCall = new SimpleJdbcCall(jdbcTemplate)
                .withCatalogName("PKG_TEST") //package name
                .withFunctionName("HELLO_WORLD");
        SqlParameterSource paramMap = new MapSqlParameterSource()
                .addValue("param", "value"));
        //First parameter is function output parameter type.
        return jdbcCall.executeFunction(String.class, paramMap));
    }

}
```

This document serves as a comprehensive guide on how to call stored procedures and functions from Spring Data JPA. It provides step-by-step instructions and examples, making it easy to follow and understand.

The guide begins by introducing the concept of stored procedures and their role in database management. It then goes on to demonstrate how to define stored procedures using the @NamedStoredProcedureQuery and @StoredProcedureParameter annotations. These annotations are used to define the stored procedure and its parameters, respectively.

The guide then moves on to show how to create a Spring Data JPA repository to call these stored procedures using the @Procedure and @Param annotations. These annotations are used to map the stored procedure parameters to the repository method parameters. By doing so, the repository methods can be used to call the stored procedures.

Furthermore, the guide also covers how to define a custom repository to call stored procedures as a native query. This can be useful in situations where the standard approach using annotations is not feasible. The guide provides a detailed example of how to create a custom repository and use SimpleJdbcCall to call the stored procedures as a native query.

Lastly, the guide also explains how to call functions using native queries or SimpleJdbcCall. It provides examples of how to define a function using the CREATE FUNCTION statement and how to call the function using native queries or SimpleJdbcCall.

Overall, this document is an informative and useful guide for developers who want to learn how to call stored procedures and functions from Spring Data JPA. It covers a range of topics and provides clear examples, making it an excellent resource for both beginners and experienced developers.
