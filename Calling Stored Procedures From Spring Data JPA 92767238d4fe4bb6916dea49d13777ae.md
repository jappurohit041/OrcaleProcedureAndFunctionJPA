# Calling Stored Procedures From Spring Data JPA

Created by: Jap Purohit
Created time: March 21, 2023 9:02 AM
Last edited by: Jap Purohit
Last edited time: March 21, 2023 9:05 AM

Consider the following stored procedure:

```jsx
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

```jsx
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

```jsx
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

```jsx
public interface MyTableRepositoryCustom {
    void inOnlyTest(String inParam1);
 }
```

We then make sure our main repository extends this interface:

```jsx
public interface MyTableRepository extends CrudRepository<MyTable, Long>, MyTableRepositoryCustom {
 }
```

We then create our custom repository implementation:

```jsx
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

```jsx
@Autowired
MyTableRepository myTableRepository;

// And to call the method -
myTableRepository.inOnlyTest(inParam1);
```