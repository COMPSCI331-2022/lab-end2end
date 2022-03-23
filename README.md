Lab - JPA and end-to-end web service development
==========

Before you start
----------
The purpose of this lab is to reinforce and build upon the course material, gaining more practice with JPA, Jackson, and JAX-RS in the end-to-end development of a simple Concert service.

**This lab does NOT allow you to change your teams (of 3) from what you formed for your project.** As usual, you must join your "group" on Canvas and your "team" on Github, and must provide a `Team.md` file. 

**Note:** There is a recording discussing **JPQL** on Canvas.

Exercise - Develop a stateless Concert Web service that uses ORM
----------
Develop a JAX-RS Web service for managing Concerts. The service should be stateless and use JPA to persist domain objects to a database.

The service is to provide a basic REST interface as follows:

- `GET /concerts/{id}`. Retrieves a representation of a `Concert`, identified by its unique ID. The HTTP response message should have a status code of either 200 or 404, depending on whether the specified concert is found.

- `POST /concerts`. Creates a `Concert`. The body of the HTTP request message contains a representation of the new concert (other than the unique ID) to create. The service generates the concert's ID via the database, and returns a HTTP response of 201 with a `Location` header storing the URI for the newly created concert.
  
- `PUT /concerts`. Updates an existing `Concert`. A representation of the modified concert is stored in the body of the HTTP request message. Being an existing concert that was earlier created by the Web service, it should include a unique ID value. The HTTP status code should be 204 on success, or 404 where the concert isn't known to the Web service.

- `DELETE /concerts/{id}`. Deletes a `Concert`, where the concert to delete is specified by a unique ID. This operation returns either 204 or 404, depending on whether the concert exists.

- `DELETE /concerts`. Deletes all `Concert`s, and returns a 204 status code.

#### About the domain model
The Web service is to store `Concert` and `Performer` entities. Similarly to the `lab-jpa` exercise, a `Concert` has a `Performer`, `Performer` may perform at many `Concert`s. For the REST interface, you don't need to use DTO classes - `Concert`s and `Performer`s are sufficiently simple and involve little data, so when a `Concert` is being exchanged between a client and the Web service, it should include its `Performer`. JSON should be supported as the data interchange format of choice for this web service.

#### Project structure
A partially complete project named `lab-end2end` can be found in this repository. The project is a multi-module project with `lab-end2end-concert-domain-model` and `lab-end2end-concert-web-service` as child projects.

- The parent project's POM file declares dependencies common to the two modules.

- The `lab-end2end-concert-domain-model` project includes complete implementations of the `Concert` and `Performer` classes, along with classes that are useful during JSON marshalling/unmarshalling. It also contains the unit test class `DomainModelTest`, which can help determine whether the domain classes have been correctly annotated for persistence (JPA) and data transfer (Jackson).

- Project `lab-end2end-concert-web-service` includes a JPA configuration file (`persistence.xml`), a class named `PersistenceManager`, and an integration test class (`ConcertResourceIT`).

#### (a) Develop the project
To complete the project, you need to do the following:

- Add all necessary metadata (annotations) to the domain classes `Concert` and `Performer`. You should use appropriate Jackson annotations to specify how instances are to be marshalled / unmarshalled to/from JSON. Similarly, you should add JPA annotations to specify how `Concert` and `Performer` instances should be persisted.
   - **You may transfer your `Concert` and `Performer` classes from the previous lab's `lab-jpa-database` exercise** to give you a head start on the JPA annotations.
   - **Note:** The `Concert` class doesn't have a `setPerformer()` method. Consider how this interacts with Jackson marshalling / unmarshalling, and how you might address this issue.

- Introduce a subclass of `javax.ws.rs.core.Application`. This has two start-up responsibilities. First, it should return a *resource-per-request* handler for processing HTTP requests. Second, it should instantiate the supplied `PersistenceManager` class.

- Implement the `Resource` class. As with any JAX-RS application, you need a `Resource` class that performs the actual processing of requests.

##### Stateless service &amp; resource-per-request
A key difference between this web service and the earlier ones that you've developed is that this one should be stateless - with all `Concert` and `Performer` data stored in a database. Rather than returning a singleton `Resource` handler *__object__*, the `Application` subclass should return a `Resource` handler *__class__*.

Where a `Resource` handler class is returned, the JAX-RS run-time instantiates it every time it's needed to process an incoming request. This is known as  *resource-per-request* handling. It differs to the *singleton-resource* handler that has been used until now, where one `Resource` object is created on start up and used to process all requests. When a web service stores state, obviously the same `Resource` object must be used to process all requests since the `Resource` object updates its in-memory state each time. Now that the web service is to be stateless, the *resource-per-request* model is more appropriate; the `Application` subclass' `getClasses()` method should return the `Resource` class.

##### PersistenceManager
Class `PersistenceManager` is a singleton class that provides the means to create an `EntityManager`. An `EntityManager` essentially provides the JPA interface for reading and writing persistent objects, and is required by the `Resource` class to retrieve and store data in the database. At construction time, the `PersistenceManager` creates a JPA `EntityManagerFactory` that reads the `persistence.xml` file, scans the entity classes for mapping metadata, and generates the relational schema. These actions should be performed *__once__*, on application startup (hence it's convenient for the `Application` subclass' `getSingletons()` method to instantiate the `PersistenceManager`). 

##### Use of an EntityManager	
In implementing the `Resource` class' handler methods, you'll need to use the `EntityManager`. The typical usage pattern for `EntityManager` is shown below. Particular `EntityManager` methods that will be useful for this task include:

- `find(Class, primary-key)`. `find()` looks up an object in the database based on the type of object and primary-key value arguments. If a match is found, this method returns an object of the specified type and with the given primary key. If there's no match the method returns `null`.

- `persist(Object)`. This persists a new object in the database when the enclosing transaction commits.

- `merge(Object)`. A `merge()` call updates an object in the database. When the enclosing transaction commits, the database is updated based on the state of the in-memory object to which the `merge()` call applies.

- `remove(Object)`. This deletes an object from the database when the transaction commits.

For this task, the above `EntityManager` methods are sufficient. The `EntityManager` interface will be discussed in more detail this week; for more information in the meantime consult the Javadoc for `javax.persistence.EntityManager` (<https://docs.oracle.com/javaee/7/api/javax/persistence/EntityManager.html>).

A simple JPQL query to return all `Concert`s might be useful for this task, and this can be expressed simply as:

```java
EntityManager em = PersistenceManager.instance().createEntityManager();
TypedQuery<Concert> concertQuery = em.createQuery("select c from Concert c", Concert.class);
List<Concert> concerts = concertQuery.getResultList();
```

Resources, including an add-on lecture video, for JPQL have been provided on Canvas. For further information, the Java Enterprise tutorial includes a chapter on JPQL: <https://docs.oracle.com/javaee/7/tutorial/persistence-querylanguage.htm>.

###### EntityManager usage scenario

```java
// Acquire an EntityManager (creating a new persistence context).
EntityManager em = PersistenceManager.instance().createEntityManager();
try {
    
    // Start a new transaction.
    em.getTransaction().begin();
    
    // Use the EntityManager to retrieve, persist or delete object(s).
    // Use em.find(), em.persist(), em.merge(), etc...
    
    // Commit the transaction.
    em.getTransaction().commit();
    
} finally {
    // When you're done using the EntityManager, close it to free up resources.
    em.close();
}
```

#### (b) Run the project
Once you've added the metadata, `Application` and `Resource` classes, build and run the project, by running the Maven `verify` goal on the `lab-end2end` project. This will run both the domain model unit tests, and the web service integration tests. These should all pass.

You may run just the domain model unit tests (`DomainModelTest`) directly from your IDE, if you would like to make sure your annotations are correct before proceeding to implement the web service.

You might want to use the H2 console to see the tables that have been generated and populated. 

#### Github Classroom
The submission for this lab will follow the same final step previous labs, which means you will be creating a **"submission"** branch and push your final code to that branch for submission. Note that as previous labs this must be done (as a team) once you are fully satisfied with your code and have successfully tested your code locally. Again, only 3 attempts for pushing to "submission" branch are allowed. Except for this, there are no other rules for how you manage your braching for this lab. After you finish all exercises and push the code to the repository on Github, please check if the execution of `Github classroom workflow` runs successfully without any test fails.

Points to consider
----------

- With regard to web services, what are the benefits to having stateless services?

- What are some of the benefits provided by an ORM?

- What are the benefits and drawbacks with using your domain model classes for both data transfer and persistence, as opposed to using separate DTO classes for data transfer?

## Assessment

The marking of this lab will be based on your team repository as of **Monday 11 April 1700hrs NZ time**. Note that extra three days are provided to accomodate your preparation for Quiz 1, scheduled for Friday of Week 6 1800hrs. As well as the changes made to it for the above exercises, you must include a file <tt>Team.md</tt> containing the list of members in your
team along with UPIs and a brief summary of what role each member played. Please also include, for each member, what their Github account name is. If this file is not provided then there will be a **50% penalty**. As notified via Week02Lab General Feedback and Week 04 Study Plan annoucements, you must enter richer discussions/issues and commit messages supporting your work. Providing clearly visible evidence of your team contribution and coorperation is your responsibility.

**You must enter your initial team communication via GitHub Discussions before Thursday, 31 March 17:00hrs NZ time**. Some examples of this evidence could be: who does what, by when you plan to finish the exercises, constraints, planning notes, etc. Additionally, feel free to use GitHub Issues to report and discuss specific issues that you face while working on the exercises. If you end up using both Discussions and Issues, clearly state this in Team.md. If you fail to add any notes to Discussions before the Initial Team Communication deadline, **25% penalty** will be applied to the lab marks.

**Note that your team (one of the team members to push on behalf of the team) must NOT push to the "submission" branch on remote (GitHub repo) more than 3 times**. Executing workflows incurs cost, and you must make sure you do not violate this rule while making your submissions. **Violating this rule will bring penalties too**. To be safe, never push to submission branch unless you have fully tested your final code. 

Overall, the assessment will be performed by examining the **code, commit logs, Team.md, comments recorded in GitHub Discussions and other
information associated with your team repository**. You must demonstrate that
you have engaged with the lab material and fully participated with the
team. This means we expect to see non-trivial commits, with meaningful commit
messages, corresponding to each exercise. Different team members will do
different things and different times, but we will be looking for evidence that
there was team cooperation and collaboration. Examples including making useful
commits, communications via GitHub Discussions e.g. commenting on actions by other team members, resolving conflicts, etc. 

**Note:** Your team may **NOT** be different to that of project. See more details about this on the Canvas Assignments page.
