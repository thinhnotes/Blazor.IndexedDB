# Tg.Blazor.IndexedDB
This is a Blazor library for accessing IndexedDB and uses Jake Archibald's [idb library](https://github.com/jakearchibald/idb) for handling access to IndexedDB on the JavaScript side. 

This version currently provides the following functionality:

* Open and upgrade an instance of IndexedDB, creating stores
* Add and update a record to/in a given store
* Delete a record from a store
* Retrieve all records from a given store
* Retrieve a record/or records from a store by index and value if the index exists
* Add a new store dynamically

It does not, at the moment, support aggregate keys, searches using a range and some of the more obscure features of IndexedDB.

## Change Logs

### 2019-08-21

* Updated to Blazor 3.0.0 preview 8 

### 2019-08-15
 
 * Updated to Blazor 3.0.0 preview 7.
 * Added means to add a new store dynamically.
 * Added function to get current version and store names of the underlying IndexedDB.
 * Minor changes. 

### 2019-06-25

* Upgraded to Blazor 3.0.0 preview 6.

### 2019-04-21

* Upgraded to Blazor 0.9.0-preview3-19154-02 (thanks Behnam Emamian).

## Using the library

1. Install the Nuget package TG.Blazor.IndexedDB (```Install-Package TG.Blazor.IndexedDB -Version 1.0.0-preview``)
2. create a new instance of DbStore
3. add one or more store definitions
4. Inject the created instance of IndexedDbManger into the component or page where you want to use it

The library provides a service extension to create a singleton instance of the DbStore.

Within the client application's ```startup.cs``` file, add the following to the ```ConfigureServices``` function.

```CSharp
services.AddIndexedDB(dbStore =>
            {
                dbStore.DbName = "TheFactory"; //example name
                dbStore.Version = 1;

            dbStore.Stores.Add(new StoreSchema
            {
                Name = "Employees",
                PrimaryKey = new IndexSpec { Name = "id", KeyPath = "id", Auto = true },
                Indexes = new List<IndexSpec>
                    {
                        new IndexSpec{Name="firstName", KeyPath = "firstName", Auto=false},
                        new IndexSpec{Name="lastName", KeyPath = "lastName", Auto=false}

                    }
            });
                dbStore.Stores.Add(new StoreSchema
                {
                    Name = "Outbox",
                    PrimaryKey = new IndexSpec { Auto = true }
                }
                    );
            });
```
### A breakdown of what this does

#### Step 1 - define the database
To define the database we need to first give it a name and set its version. IndexedDB uses the version to determine whether it needs to update the database. For example if you decide to add a new store then increment the version to ensure that the store is added to the database.

#### Step 2 - Add a store(table) to the database
In IndexedDB a store is equivalent to table. To create a store we create a new ```StoreSchema``` and add it to the collection of stores.

Within the ```StoreSchema``` we define the name of the store, the primary index key and optionally a set of foreign key indexes if required.

The ```IndexSpec``` is used to define the primary key and any foreign keys that are required. It has the following properties:

* Name - the name of the index
* KeyPath - the identifier for the property in the saved object/record that is to be indexed
* Unique - defines whether the key value must be unique
* Auto - determines whether the index value should be generated by IndexedDB.

In the example above for the "Employees" store the primary key is explicitly set to the keypath "id" and we want it automatically generated by IndexedDB. In the "Outbox" store the primary key just has ```Auto = true``` set. IndexedDB is left to handle the rest.

## Using IndexedDBManager

For the following examples we are going to assume that we have Person class which is defined as follows:

```CSharp
 public class Person
    {
        public long? Id { get; set; }
        public string FirstName { get; set; }
        public string LastName { get; set; }

    }
```
And the data store name is "Employees"

### Accessing IndexedDBManager

To use IndexedDB in a component or page first inject the IndexedDbManager instance.

```CSharp
@inject IndexedDBManager DbManager
``` 

### Setting up notifications

IndexedDBManager exposes ```ActionCompleted``` event that is raised when an action is completed. 

If you want to receive notifications in the ```OnInit()`` function subscribe to the event.

The function that handles the event should have the following signature:

```CSharp
 private void OnIndexedDbNotification(object sender, IndexedDBNotificationArgs args)
    {
        Message = args.Message;
    }
```

It is recommended that your page or component should also implement IDisposable to unsubscribe from the event.

### Adding a record to an IndexedDb store

Assuming we have a new instance of our sample ```Person``` class, to add to the "Employees" store doing the following:

```CSharp
var newRecord = new StoreRecord<Person>
        {
            Storename ="Employees",
            Data = NewPerson
        };

await DbManager.AddRecord(newRecord);
```

### Getting all records from a store

```CSharp
 var results = await DbManager.GetRecords<Person>("Employees");
 ```
### Get record by Id
To get a record using the id can be done as follows:

```CSharp
 CurrentPerson = await DbManager.GetRecordById<long, Person>(DbManager.Stores[0].Name, id);
```

### getting a record using the index

To search for a value on an index first create an instance of ```StoreIndexQuery<TInput>``` providing the name of the store to search in, the index to search on and the value to search for.

```CSharp
var indexSearch = new StoreIndexQuery<string>
        {
            Storename = DbManager.Stores[0].Name,
            IndexName = SelectedIndex,
            QueryValue = SearchString,
        };
```

The StoreIndexQuery is then passed to the function ```IndexedDbManager.GetRecordByIndex<TInput, TResult>()``` where TInput is the type of the value to search for and TRresult is the type of the return value.

```CSharp
 var result = await DbManager.GetRecordByIndex<string, Person>(indexSearch);

            if (result is null)
            {
                return;
            }
People.Add(result);
```

By default IndexedDB only returns the first record found that matches the query. If you want to get all of the records that match the query value use the following:

```CSharp
 var result = await DbManager.GetAllRecordsByIndex<string, Person>(indexSearch);

if (result is null)
{
    return;
}
People.AddRange(result);
```

### Updating a record

To update a record call ```IndexedDbManager.UpdateRecord<T>(StoreRecord<T> recordToUpdate)```.

### Deleting a record

To delete a record call ```IndexedDbManager.DeleteRecord<TInput>(string storeName, TInput id)```.

### Clear all records from a store

To clear all the records in a store call the following function ```IndexedDbManager.ClearStore(string storeName)```.

### Deleting a Database

If you are so inclined you can delete an entire database with the following function ```IndexedDbManager.DeleteDb(string dbName)```.

### Adding a new store dynamically

If you have occasion to what to add a store when the program is up and running. The following

```CSharp
var newStoreSchema = new StoreSchema
    {
        Name = NewStoreName,
        PrimaryKey = new IndexSpec { Name = "id", KeyPath = "id", Auto = true },
    };

    await DbManager.AddNewStore(newStoreSchema);
```

What this will do is, if the store doesn't already exist, is increment the database version number and add the store to the database.
