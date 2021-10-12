# Pagination
In this article, we will be discussing Pagination.
## Server Side Pagination

Assume your request is to return a large number of results. For example, a return list of employees in a company, Microsoft, will result in 100K records. This incurs a lot of load on the server and the client, but our focus is on the server. The server would return this huge result set size in multiple paginated calls.
### Semantics
First, let's have a look at semantics for a better understanding

#### **Sample Response**
```
{"value": [
    {"firstname": "Alex",
     "lastname": "Tom"
    }
]
  "nextLink": "https://all.companydata.com/microsoft/employees?$skipToken=23453642rerwqwe"
}
```
#### **Sample psuedo code in client to process paginated data from server.**
```
nextLink = "https://all.companydata.com/microsoft/employees"
do {
    result = makeApiCall(nextLink);
    process(result.value); // process the list of items returned by the API
    nextLink = result.nextLink; // update the nextLink to fetch more records
} while (nextLink != null)
```
This is just like, single linkedlist.

# **implementation**
Lets assume the data is in MySQL server.
## 1. Limit offset
The first thing that comes to mind is to use MySQL limit and offset.
*Sample Psuedo code*
```
serverpagesize = 100
offsetval = retrieveOffSetValueFromSkipToken(skipToken)
listofrecords = execute_sql(select * from allemployes where company = 'Microsoft' order by id LIMIT serverpagesize OFFSET offsetval)
if (len(listofrecords) < serverpagesize) {
    nextLink = null
} else {
    nextLink = generateNextLinkWithOffset(offsetval + len(listofrecords))
}
return constructResponse(listofrecords, nextLink)
```
The nextLink for the above code is `https://all.companydata.com/microsoft/employees?$skipToken=151`, where 151 is the offset value.

*Employees table*
|id | firstname | lastname | company | datejoined|
|-|-|-|-|-|
|1  | Alex|Tom|Microsoft|xxx|
2  | Nik|M|Microsoft||
4  | Vik|B|Microsoft||
5  | Vik1|B|Microsoft||
6  | Vik2|B|Microsoft||
7  | Vik3|B|Microsoft||


### **Let's identify the problem with above code (race conditions)**
 #### **Let's assume new employee got added**
- Page size is 3, so the first call would return Alex, Nik, and Vik with a skipToken offset as 3
- Now with id 3 Bhan got added

|id | firstname | lastname | company | datejoined
|-|-|-|-|-|
1  | Alex|Tom|Microsoft||
2  | Nik|M|Microsoft||
3  | Bhan|A|Microsoft||
4  | Vik|B|Microsoft||
5  | Vik1|B|Microsoft||
6  | Vik2|B|Microsoft||
7  | Vik3|B|Microsoft||

- Second call with the next link from first call would return Vik, Vik1, Vik2  with a skipToken offset as 6
- *Problem:* `It might return duplicate records of the data in the subsequent pagination calls.(return Vik in above example)`

#### **Let's assume new employee got deleted after first page is returned(Alex, Nik, Vik). Alex record is deleted.**
- Second page would return vik2, vik3, but vik1 got missed.
- *Problem:* `It might miss the records in the pagination call.(In above example Vik1 record got missed)`

## 2. Id as Skip Token (order by id)
*Sample Psuedo code*
```
serverpagesize = 100
idFromSkipToken = retrieveIdValueFromSkipToken(skipToken)
listofrecords = execute_sql(select * from allemployes where company = 'Microsoft' where id > idFromSkipToken order by id LIMIT serverpagesize)
if (len(listofrecords) < serverpagesize) {
    nextLink = null
} else {
    nextLink = generateNextLinkWithMaxId(max(fetchIds(listofrecords)))
}
return constructResponse(listofrecords, nextLink)
```
### **Let's assume new employee got added afer first call**
- First Call: Page size is 3, so the first call would return Alex, Nik, Vik with skipToken id as 4
- Now with id 3 Bhan got added
- Second Call: Page Size is 3, so the second call would return Vik1, Vik2, Vik3 with skipToken id as 7
- As you can see, we are not returning duplicate records, as we did in the *OFFSET* approach.

### **Let's assume new employee got deleted after first page is returned(Alex, Nik, Vik). Alex record is deleted.**
- First Call: Page Size is 3, so first call would return Alex, Nik, Vik with skipToken id as 4
- Now with Alex(id: 1) is deleted
- Second Call: Page Size is 3, so second call would return Vik1, Vik2, Vik3 with skipToken id as 7
- As you can see, we are not missing any records, as we did in the *OFFSET* approach.

## 3. How to handle server pagination with different order by firstName
- With Id as skip Token approach (where id > idFromSkipToken) would work because id is unique.
- With this, order by firstName, scenario (where firstName > firstFromSkipToken) might not work, because first names may have duplicates.
- Always try to make sure order by clause would return a unique row(deterministic ordering), ex `order by firstName` will be `order by firstName, id`
- Now skipToken will have firstName, and id.
- Now modify the where condition to `(where firstName > firstFromSkipToken or (where firstName == firstNameFromSkipToken and id > idFromSkipToken))`
- If the order by is on firstName, LastName then order by clause would be `order by firstName, LastName, Id`.
where condition will be `where firstName > firstNameFromSkipToken or (firstName == firstNameFromSkipToken and LastName > lastNameFromSkipToken) or (firstName == firstNameFromSkipToken and LastName == lastNameFromSkipToken and id > idFromSkipToken)` 
- With this long where condition, make sure you have right indexes on the table.

## Note
The delete scenario described above does not have to be a hard delete; it could be data missing from your original query after certain pagination calls. An example, Assume your API returns a list of pending transactions; after the first paginated call, a pending transaction may become a posted transaction.


# References
https://docs.microsoft.com/en-us/odata/webapi/skiptoken-for-server-side-paging
