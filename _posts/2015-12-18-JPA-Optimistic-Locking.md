---
layout: post
title: Implementing Optimistic Locking with JPA
published: true
categories: 
            - Java
            - JPA

---

#Motivation
It is possible to have multiple processes trying to modify the same record in the database. If a process manages to modify a stale record
(i.e. one which has been updated by another process but not refreshed), then it could lead to the record being overwritten by the second process.

In order to solve this problem, we tend to implement optimistic locking. This is a strategy where we hold a version column in the schema for
the entity we wish to control updates to. The idea behind the version column is that a record will read the version number along with the rest of the entity
and when updating the record, it will ensure it only updates the record if the version number is still the same.

Another process could have modified the same record in a different transaction and moved the version number on. In this case the first process should fail to update the record
and result in a exception condition.

##Example
Consider the following JPA entity:

{% highlight java linenos=table%}
import javax.persistence.Entity;

@Entity
public class Customer{
  private String firstname;
  private String lastname;
  private int state;
  
  public void setState(int state){
    this.state = state;
  }
}
{% endhighlight %}

1. Customers can be in various states e.g. 1, 2, 3, 4

2. The state field must be updated correctly. Parallel process could be running and both could attempt to update the state to a different value.

##Test
Lets write a failing test to prove the fix that we will do.

{% highlight java linenos=table%}

@Test(expected = OptimisticLockException.class)
public void CustomerTest(){
  //just imagine we have a DAO somewhere that has finder methods on it.
  final Customer customer1 = dao.findCustomerByCustomerNo(1);
  final Customer customer2 = dao.findCustomerByCustomerNo(1);

  customer1.setState(76);
  customer2.setState(98);

  try{
      dao.update(customer1); //transaction created to persist customer
      dao.update(customer2); //another transaction created but should not allow update due to optimistic locking
     }
     catch(final OptimisticLockException lockException){
        final Customer modifiedCustomer = dao.findCustomerByCustomerNo(1);
        assertThat(modifiedCustomer.getState(), is(76));
        throw lockException;
    }
}
{% endhighlight %}

1. This test makes two calls to the DAO to fetch the same customer twice.
2. It then updates the first customer entity with a state of 76 and the second with a state of 98
3. The test currently fails because the assertion for checking the state is now 76 fails. This is because the second update overwrote the first.
4. The test also fails because we are expecting to get a OptimisticLockException.

##How to fix it
Firstly we need to ensure we create column on our CUSTOMER table that we can use for the version. You can use any datatype for this
column, however hibernate recommends using Long.

Secondly we need to add the version column to our customer entity and annotate it with @Version.

The Customer entity now looks like:

{% highlight java linenos=table%}
import javax.persistence.Entity;

@Entity
public class Customer{
  private String firstname;
  private String lastname;
  private int state;
  
  @Version
  private Long version;
  
  public void setState(int state){
    this.state = state;
  }
}
{% endhighlight %}

And that's all we have to do to make it work. The test should now pass because the version column would have prevented the second update
from being successful and threw a OptimisticLockException.
