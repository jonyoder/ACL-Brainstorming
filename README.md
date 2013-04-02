ACL-Brainstorming
=================

I like the idea of group-based permissions for domain objects. What if we created a "groups" table in our SSO database and then created a "Group_ACL" table:

```
    (existing roles table)
    ------------
    |   Roles  |
    ------------
    | Role_ID |
    | User_ID |
    | Role    |
    
    (create ACLs)
    -------------
    | Group_ACL |
    -------------
    | Group_ID  |
    | User_ID   | (from our "users" table; allows null)
    | Role_ID   | (from our "roles" table; allows null)
    | Access    | (read, write)
    | Class     | (object class)
    | PK        | (object PK value)
    -------------
```

With this model, we could handle "update" functions like this:

1.  Get a list of ACLs for the object (class+PK) that match the user's User_ID
2.  Check the list of "user" ACLs for one with the "access" level specified in the method annotation.
3.  If no ACLs with "write" access are found in step #2, then
    - Look up the user's roles.
    - Get a list of ACLs for the object (class+PK) that match the user's roles.
    - Check for an ACL with the "access" level specified in the method annotation.

We could handle "create" functions like this:

1. Look up the user's roles.
2. Check the method's annotations for a required role that matches one of the user's roles.
3. Create the new object and persist it in the database.
4. Create the following ACLs for the object
    - ACLs that associate the method's annotated roles with with appropriate access level for the object
    - And/Or an ACL that associates the user's User_ID with the appropriate access level for the object
    - And any additional ACLs that associate varying access levels with annotated or otherwise specified groups.


Examples in a RESTEasy service implementation:
-----------------------------------------------

```java
    @GET
    @RolesRequired({"LCDB-Reader", "LCDB-Writer"})
    @AccessLevel("read")
    public Patient getPatient(…) {
    
        /*
    
        Users with the LCDB-Reader or LCDB-Writer roles who have a "read" ACL could run this function
    
        For example, any of the following would match:
        
        A:
            User = Robert;
            Role = LCDB-Writer;
            ACL = (Role_ID = LCDB-Writer, Access=read)
        
        B:
            User = Bob;
            Role = LCDB-Reader;
            ACL = (User_ID = Bob; Access=read)
        /*
        
    
    }
    
    @PUT
    @RolesRequired({"LCDB-Writer"})
    public Boolean createPatient(Patient patient) {
    
        /*
    
        Only users with the LCDB-Writer role could run this function. No ACL is required, since we're creating a new object that doesn't exist.
    
        For example:
        
        A.
            User = Robert;
            Role = LCDB-Writer;
        
        */
    
        patientDao.persist(patient);
        createACL(patient, "read"); // Defaults to @RolesRequired annotation's annotated roles.
        createACL(patient, "write", {"LCDB-Reader"}); // Specify an additional role with write access.
        createUserACL(patient, "write"); // Create additional user-level ACL
        
    
    }
    
    @POST
    @RolesRequired({"LCDB-Writer"})
    @AccessLevel("write")
    public Boolean updatePatient(Patient patient) {
    
        /*
    
        Only users with the LCDB-Writer role AND a "write" access ACL could run this function.
    
        For example:
        
        A.
            User = Robert;
            Role = LCDB-Writer;
            ACL = (Role_ID = LCDB-Writer, Access=write)
        
        B.
            User = Tom;
            Role = LCDB-Writer;
            ACL = (User_ID = Tom, Access=writer)
        */
        
    
    }
    
    @POST
    Public Boolean createOrUpdatePatient(Patient Patient) {
    
        /*
    
        For this method, we can't use annotations, since permissions vary based on the operation
    
        */
    
        if (operation == "create") {
            confirmRole("LCDB-Writer");
            return createPatient(patient);
        }
        
        else if (operation == "update") {
            confirmRoles({"LCDB-Writer"}); // optionally confirm that our user has a specific role or roles
            confirmACL(patient, {"writer"}); // confirm that the user has at least one ACL associated with his/her user and/or roles
            return updatePatient(patient);
        }
        
    
    }
```
