## TommyTable
### A full featured table component written in Vue that suports multi-column searching, sorting, and more
---

## Before you go any further...

This component is used with preloaded JSON data. It doesn't do additional XHR requests to get more data.

It is intended for small to medium sized datasets and you would supply the data from the server on page load with something like this (PHP):

 ```
 window.data = @json($accounts);
 ```

## Quickstart

The only properties you really need to supply is the "tabledatavar" property, where you tell it the name of the variable that contains the JSON data, and the "detaillink" property,\
which is used to build hyperlinks to view the details of each row, like this:
```
<tommytable
    :tabledatavar="window.data"
    :detaillink="/account/show/"
```
That's all you need to get all the nice things like batching, multi-column search-as-you-type and sorting, etc.

I would recommend starting with that and adding some of the other properties below as needed.

## Properties

| Name                | Type              | Default                   | Description                                                       |
| ---                 | ---               | ---                       | ---                                                               |
| batchsizes          | Array (numbers)   | [25,50,100,200]           | How many records to show/load at a time                           |
| bitindicators       | String            | ['Y','N']                 | Labels to use for true/false indicators                           |
| choices             | Array             | []                        | Information that is used to generate a select element in header   |
| datatypes           | Array             | []                        | How the data is formatted                                         |
| debugmode           | Boolean           | false                     | Whether to show intialization information in console              |
| defaultsearches     | Array             | []                        | Searches to execute on mount                                      |
| defaultsorts        | Array             | []                        | Sorting to apply on mount                                         |
| detaillink          | String            | '/'                       | URL that is prepended to the id to view detail of row             |
| tabledatavar        | String            | 'window.tommyTableData'   | Variable to evaluate that supplies the data (JSON format)         |

## Additional information about the properties

### A note on all of the array properties

For all arrays, the position will correspond to that column in the table. Note this includes the first "id column" that isn't displayed.
Also you don't have to specify types for every column. For instance:
```
:datatypes="[,,,'money']
```
will use default type (string/numeric) for the first three columns, then 'money' for the fourth, then the default for the rest of the columns.

### batchsizes
The first element will be how many records are shown on mount. This also controls how many addtional records are added when you click 'Load more'.

Note that an 'All' option is automatically appended.

### bitindicators
Element 0 is the true label, element 1 is the false label.
This will be used for both the 'bit' datatype and also for display of 'choice' data if the dataTypeFormat is set (more on that below).

### choices

This is used to generate a select element in header. It consists of a label for the select option,
a callback function of the table cell data that determines if the criteria is met, and any optional formatting.
```
[{ options: [{ label: String, criteria: function(x: any): boolean }], dataTypeFormat?: 'bit' | 'money') }]
```

Note the criteria function of the first options element should always have the function:
```
() => true
```
(and a label of something to indicate no filtering, like '-select-') otherwise it won't work correctly when the search fields are reset

### datatypes
```
null | 'bit' | 'choice' | 'money'
```
null/empty this is the default and will be treated as a string (search as you type)\
'bit' is represented by 1 or 0 in your supplied JSON data\
'choice' this indicates a select to be used for search\
'money' is numeric data that will be money formatted

### defaultsearches
```
1 | 0 | { choiceCriteraIndex: Number}
```
1 and 0 is used for 'bit' datatypes.
The object is used for 'choice' datatypes (the index of the choice's criteria function to execute on mount - see the 'choices' property).

### defaultsorts
```
'asc' | 'desc'
```

Note for sorting it will do a numeric sort if the column value can be converted to a number (!isNaN) otherwise it will do a string sort

### detaillink
This is used to build the hyperlink for the data in the first column.
For instance, passing:
```
:detaillink="/account/show/"
```
Will result in data in the first column to be hyperlinked with this (assuming 123 is the id for that row): 
'account/show/123'

Note if the id is 0 (or negative) it will not be hyperlinked.

## About the data you supply

It assumes an array of Objects with key/value pairs (like you would get from a query).

All headers (keys) will be converted like this:\
account_number --> Account Number

There is an assumption that the element at position 0 is a numeric id field (but it doesn't have to be named 'id' - it just looks at position). It is assumed this is something like a primary key. This will not be displayed in the table.

There is also an assumption that the element at position 1 is the field that will be hyperlinked with the id from above (so this should be some identifier of the record,\
like an account number or person's name).
