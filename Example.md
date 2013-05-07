```
+===================+
|| Legend          ||
+-------------------+
|| + Primary Key   ||
|| * Foreign Key   ||
+===================+


Assume the following database structure:
                                                       +-----------------+
     +-----------------+      +-----------------+      |     Store       |
     |      Sale       |      |    Customer     |      +-----------------+
     +-----------------+      +-----------------+  +---| + Store_ID      |--+  
     | + Sale_ID       |  +---| + Customer_ID   |  |   |   Name          |  |
  +--| * Store_ID      |  |   | * Pref_Store_ID |--+   |                 |  |
  |  | * Customer_ID   |--+   |   Name          |      |                 |  |
  |  |   Total         |      |                 |      |                 |  |
  |  |                 |      |                 |      +-----------------+  |
  |  +-----------------+      +-----------------+                           |
  +-------------------------------------------------------------------------+           
```

  Assume that sale with Sale_ID = 25 has the following properties:
  	Sale_ID     =    25
  	Store_ID    =     1
  	Customer_ID =     1
  	Total       = 99.99

  Assume that we want to completely change the sale like this:
  	Sale_ID     =    25
  	Store_ID    =     5
  	Customer_ID =     6
  	Total       = 23.99

  Assume the following:
	1. The user needs a WRITE Sale ACL to update the sale (change the sale total).
	2. The user needs a special ACL to decrease the sale total.
	3. The user needs READ access to the old store to load the sale
	4. The user needs READ access to the new store to change the store in the sale
	5. The user needs WRITE access to the new customer to change the customer in the sale. However, the customer object bases its permissions on the user's privileges to the store, which means that the user needs WRITE access to the store to change the customer in the sale

  Assume, additionally, that the user wants to insert a new sale
  Assume the following:
	6. The user needs READ access to the store to insert the sale.
	7. The user needs READ access to the customer to insert the sale.
	8. A WRITE Sale ACL should be created for the user for the new sale.
	9. A READ Sale ACL should be created for the "audit" role for the new sale.
	10. A DELETE Sale ACL should be created for the user and for the "manager" role for the new sale.

  We also have the following assumptions that are not specific to the above tasks:
	11. When a user attempts to load a sale with a specific ID, pre-authorize based on the ID parameter.
	12. When a user attempts to load a sale any other way, user needs a READ Sale ACL.


  Assume, finally, that the user wants to delete a sale
    13. A user needs DELETE access to the sale to delete it.


  Other specifications:
	1. When a method uses @CasHmacPreAuth, we may need to force skipping of other authorization checks to speed things up.


```java
@Inject SaleRestService saleRestService;

// Load a sale
Sale sale = saleRestService.loadSale(25);

// Update the sale
sale.setStoreID(5);
sale.setCustomerID(6);
sale.setTotal(23.99);
saleRestService.putSale(sale);

// Create a new sale
Sale newSale = new Sale();
sale.setStoreID(3);
sale.setCustomerID(4);
sale.setTotal(100.00);
saleRestService.putSale(sale);

// Delete a sale
saleRestService.deleteSale(25);

public static class SaleRestService {

	@Inject	SaleDao saleDao;

	/* Loads an existing sale */
	@CasHmacPreAuth(READ, Sale.class, "saleid")      // Fulfill Assumption #11
	public Sale loadSale(Integer saleId) {
		saleDao.load(saleId);
	}

	/* Adds or updates a sale */
	public putSale(Sale sale) {
		Sale saleOrig = saleDao.load(sale.saleId);
		                                             
		if (saleOrig.total > sale.total) {
			CasHmacValidation.verifyCustomAcl(           // Fulfill Assumption #2
        "DECREASE", Sale.class, sale
      );
		}
		saleDao.put(sale);
	}

	/*
     Deletes a sale. We could force a quick
     pre-auth here with:
     `@CasHmacPreAuth(DELETE, Sale.class, "saleId")`.
  */
	public deleteSale(Integer saleId) {
		saleDao.delete(saleId);
	}

}

@CasHmacObjectAcl(Sale.class)                      // Fulfill Assumption #12
@CasHmacObjectRead(READ, Sale.class)               // Fulfill Assumption #12
@CasHmacObjectUpdate(WRITE, Sale.class)            // Fulfill Assumption #1
@CasHmacObjectDelete(DELETE, Sale.class)           // Fulfill Assumption #13 
public static class Sale {
	@CasHmacPKField                                  // Fulfill Assumption #1
	@CasHmacWriteACL({
		{ READ },
		{ WRITE },                                     // Fulfill Assumption #8
		{ READ, { "audit" } },                         // Fulfill Assumption #9
		{ DELETE { SELF, "manager" } },                // Fulfill Assumption #10
		{ "DECREASE", { "manager" } }                  // Fulfill Assumption #2
	})                                               
	public Integer saleID;
	@CasHmacForeignFieldRead(READ, Store.class)      // Fulfill Assumption #3
	@CasHmacForeignFieldUpdate(READ, Store.class)    // Fulfill Assumption #4
	@CasHmacForeignFieldCreate(READ, Store.class)    // Fulfill Assumption #6
	public Integer storeID;
	@CasHmacForeignFieldUpdate(WRITE, Customer.class)// Fulfill Assumption #5
	@CasHmacForeignFieldCreate(READ, Customer.class) // Fulfill Assumption #7
	public Integer customerID;
	public Float   total;
}

/*
   We don't need the @CasHmacObjectAcl annotation
   here, since we don't have any ACLs stored
   specifically with the Customer object
   (ACL calculated through store)
*/
public static class Customer {
	@CasHmacPKField                                  // Fulfill Assumption #5
	public Integer customerID;
	@CasHmacForeignFieldUpdate(WRITE, Store.class)   // Fulfill Assumption #5
	@CasHmacForeignFieldRead(READ, Store.class)      // Fulfill Assumption #7
	public Integer preferredStoreID;
	public String  name;
}

@CasHmacObjectAcl(Store.class)                     // Fulfill Assumptions #3,4,6,7
public static class Store {
	@CasHmacPKField                                  // Fulfill Assumptions #3,4
	public Integer storeID;
	public String name;
}

```


Scenarios Based on Assumptions:
---------------

 1. The user is attempting to update a sale.  The "@CasHmacObjectUpdate(WRITE, Sale.class)" annotation tells us that the user needs a WRITE ACL for the Sale object in order to update it.  We use reflection to look for the Sale class field annotated with "@CasHmacPKField", and then we look for a match in the database (to the user or one of the user's roles).
 
 2. Also while the user is attempting to update the sale, we include custom code to verify that the user has a custom "DECREASE" ACL for the Sale class and object.  We use reflection, as in #1, to look for the Sale class PK field and look for a match in the database.
 
 3. Also while the user is attempting to update the sale, the "@CasHmacForeignFieldRead(READ, Store.class)" annotation on the "storeID" field tells us that we need a READ ACL for the Store object in order to read the original Sale into an object.  As in #1 and #2, we use reflection to find the PK of the Store object and look up the ACL for the store
 
 4. Next, the "@CasHmacForeignFieldUpdate(READ, Store.class)" annotation tells us that we need a READ ACL for the Store object in order to update the existing Sale object.  We use reflection, like we do in 3, to find the ACL field and look of the store ACL
 
 5. Next, the "@CasHmacForeignFieldWrite(WRITE, Customer.class)" annotation tells us that we need a WRITE ACL for the Customer object in order to update the existing Sale object.  We use reflection to look at the Customer class (using the customerID field value from the Sale object).  However, we notice that the Customer class is not annotated with the "@CasHmacObjectAcl" annotation, which means that there will not be any ACLs specific to the Customer class in the database.  But, while introspecting, we also notice that the Customer class contains a field annotated with "@CasHmacForeignFieldRead(READ, Store.class)".  This annotation tells us that we need a READ ACL on the Store object specified by the "preferredStoreID" foreign key.  So, we use reflection to inspect the Store class, find its "@CasHmacPKField" annotated primary key field, and look up the ACL for the PK value specified in the Customer class.  NOTE: We also need to look up the Customer object, which we can easily do by finding the Customer class's "@CasHmacPKField" annotated field. even though the Customer class isn't annotated with @CasHmacObjectAcl.
 
 6. Here, the user is attempting to insert a new sale.  While using reflection to introspect on the fields of the Sale class, we notice the "@CasHmacForeignFieldCreate(READ, Store.class)" annotation on the "storeID" field.  This tells us that we need a READ ACL for the Store object referenced by the storeID field. 
 
 7. We also notice, while attempting to insert a sale, the "@CasHmacForeignFieldCreate(READ, Customer.class)" annotation on the Sale's customerID field.  So, we know that we also need a READ ACL for the Customer object referenced by the Sale's customerID field.

 8. For assumptions 8, 9, and 10, we notice the multi-line annotation beginning with "@CasHmacWriteACL({" that is applied to the Sale class's PK field (saleID).  The "@CasHmacWriteACL" annotation tells us that, when a new Sale object is saved, we need to create the specified ACLs.  Here, we define READ and WRITE ACLs for the current user, a READ ACL for the "audit" role, DELETE ACLs for the current user and the "manager" role, and a DECREASE custom ACL for the "manager" role.

 9. See 8.

10. See 8.

11. Since it may be inefficient to completely load a persisted object before checking an ACL, especially when we're in a situation where an attempt to load an item without the proper ACLs is likely, we create a "@CasHmacPreAuth" annotation that allows us to specify an ACCESS level, a class, and a parameter name.  An interceptor or filter runs pre-authorization code before running the method, ensuring that the specified ACL exists before even allowing the method to run.

    NOTE: When we pre-authorize, we need to cache the ACL for the preauthorized object.  Since the object is likely to have its ACL specified otherwise, we can check the cache on subsequent ACL lookups to avoid multiple lookups for the same ACL.  (See "Other specifications" #1.)

12. When a user attempts to load a sale in a method without the @CasHmacPreAuth annotation, we still need to ensure that the user has the appropriate ACL for the sale.  Since the Sale class is annotated with the @CasHmacObjectAcl annotation, we're aware that we need to look up ACLs for the Sale class. Then, the "@CasHmacObjectRead(READ, Sale.class)" annotation tells us that we need a READ ACL for the Sale class in order to read a Sale object.

13. Very similar to #12, the "@CasHmacObjectDelete(DELETE, Sale.class)" annotation tells us that we need a DELETE ACL for the Sale class in order to delete the Sale object.



Classes, Methods Needed:
--------------------------

Class Annotations:
```java
  @CasHmacObjectAcl(Store.class)                                    // Fulfill Assumptions #3,4,6,7,12
  @CasHmacObjectCreate(WRITE, Sale.class)                           // 
  @CasHmacObjectRead(READ, Sale.class)                              // Fulfill Assumption #12
  @CasHmacObjectUpdate(WRITE, Sale.class)                           // Fulfill Assumption #1
  @CasHmacObjectDelete(DELETE, Sale.class)                          // Fulfill Assumption #13
```

  These class annotations are considered when performing CRUD operations.  We'll need to create an interface that gets   implemented in our entity classes, and we'll need to ensure that the appropriate helper methods are called to perform the ACL checks specified by these annotations.

PK Field Annotations:
```java
  @CasHmacPKField                                                   // Fulfill Assumption #1,3,4,5
  @CasHmacWriteACL({                 
    { READ },                 
    { WRITE },                                                      // Fulfill Assumption #8
    { READ, { "audit" } },                                          // Fulfill Assumption #9
    { DELETE { SELF, "manager" } },                                 // Fulfill Assumption #10
    { "DECREASE", { "manager" } }                                   // Fulfill Assumption #2
  })
```

  These field annotations are considered, as well, when  performing CRUD operations, and will be used by the helper methods called when we implement the interface referenced above (under Class Annotations).

Foreign Key Field Annotations:
```java
  @CasHmacForeignFieldCreate(READ, Store.class)                     // Fulfill Assumption #6,7
  @CasHmacForeignFieldRead(READ, Store.class)                       // Fulfill Assumption #3,7
  @CasHmacForeignFieldUpdate(READ, Store.class)                     // Fulfill Assumption #4,5
  @CasHmacForeignFieldDelete(READ, Store.class)                     // 
```

  These field annotations, too, will be considered
  during CRUD operations.

Public RESTful Method Annotations:
```java
  @CasHmacPreAuth(READ, Sale.class, "saleid")                       // Fulfill Assumption #11
```

  The pre-auth annotations will be considered in a filter (RESTEasy 3) or interceptor (RESTEasy 2).

Helper Functions:
```java
  CasHmacValidation.verifyCustomAcl("DECREASE", Sale.class, sale);  // Fulfill Assumption #2
  CasHmacValidation.addCustomAcl("DECREASE", Sale.class, sale);     // 
  CasHmacValidation.deleteCustomAcl("DECREASE", Sale.class, sale);  // 
```
