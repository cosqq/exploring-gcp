# Introduction 

In my humble experience of using Google Cloud Platform, the use of datastore was used to capture ops meta (either in streaming or micro-batching). 

The important part of datastore in my opinion is the index planning. In this section, I will touch on how I create my indexes in datastore using a yaml file and explain why it was done. 

# Concepts 

There are two kinds of index entities in Cloud Datastore:
1. Built-in-Indexes : single type quries for simple usuage
2. Composite-Indexes : complex type qurires for complex usuage 

Indexes can consist of data entities property or entity's ancestors. All queries that uses a certain index will only return entities that contain that property. By default datastore will index these properties but if an entity does not possess this property, Datastore will mark the proprty as  `none` and it will not be considered a part of the results. 

However, if you explicitly define `NULL` it might return as a part of the results 


## Index Creation Concept 

Adapted from : **https://cloud.google.com/datastore/docs/concepts/indexes**

According to GCP, the index shall be properties that used:
1. equality filters
- For example, if you want to query transactions where the customer's ID is a certain value, having an index on the customer_id property allows for efficient querying of such transactions.
2. At most one inequality filter
-  For instance, if you want to find transactions where the transaction amount is greater than a specific value, having an index on the transaction_amount property supports this type of query efficiently.

3. Sort orders
- For instance, if you want to retrieve transactions sorted by date in descending order, having an index on the transaction_date property ensures efficient sorting.

4. In projections : Selecting only a subset of properties if they are not already a part of the sorted columns.
- The idea is "is there a column where the property well captures other interested columns"
- For example, if you are interested in analyzing only transaction amounts without needing other details like customer ID, transaction date, or product information, you can create a projection index on the Transaction Amount property.
-  It can be considered the composite. For example, If you want to project only transaction_amount and transaction_date, and these properties are not part of the sort order, having indexes on them can optimize projection queries.

### Composite Index 

Created to form search for complex queries. The combination of the above criteras can be used to improve search. 

1. Composite Key for Transaction History of a Customer:
- Composite Key: (Customer ID, Transaction Date). This composite key enables efficient querying of all transactions for a specific customer, sorted by date. It's useful for scenarios where you want to retrieve a customer's transaction history in chronological order.

### Ancestor Relationships 
Affect the entities group, all entities under the same ancestors are part of the same entity group. 

They are important becasue : 
- Entities of the same group allow read and write to the group in a single transactions (plannign your groups are important)
- This transactional capability ensures strong consistency for the data within the entity group, meaning that all reads within a transaction see a consistent snapshot of the data.
- Any entity not of the same entity group will not be considered a part of the read and write transactions to that group 

Thus, spend time planning the groups of entities and how properties could be linked through ancestors to support complex queries. 

For example : 

- Transaction ID (Ancestor): Each transaction with a unique Transaction ID could serve as an ancestor for related entities, such as transaction details or payment information associated with that specific transaction.

- Merchant ID (Ancestor): Similarly, transactions associated with specific merchants could share the same merchant entity as their ancestor. This allows all transactions related to a particular merchant to be in the same entity group.

- Composite Keys (Ancestor): For composite keys like (Customer ID, Transaction Date) or (Product Name, Transaction Date), the entities corresponding to these composite keys would serve as ancestors. 
    - These are ancestors because, for example, in the first composite key, Customer ancestor and Transaction ancestor could represent a set of records representning customer ID tied to a transaction date where read and writes could be frequently accessing.



## Creation of Index using YAML 
 
Important terminlogy: 
- kind : "collection"
- ancestor : as explained 
- properties : key-value pair 

Example Yaml : consisting of transaction columns 

````
indexes:
- kind: Transaction
  ancestor: yes
  properties:
  - name: customerId
  - name: transactionDate

- kind: Transaction
  ancestor: yes
  properties:
  - name: productId
  - name: transactionDate

- kind: Transaction
  ancestor: yes
  properties:
  - name: transactionAmount

- kind: Customer
  ancestor: yes
  properties:
  - name: customerName

- kind: Product
  ancestor: yes
  properties:
  - name: productName

````