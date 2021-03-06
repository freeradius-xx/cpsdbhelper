# cpsdbhelper
    A db helper designed to allow code to be chained and simplified, sql call/parameter managing can be much easier with the help of it and it's auto mapping ability
    
## Get Started
	Search for "CpsDbHelper" in Nuget Package manager:)
	
## Code generating
	An OPTIONAL build tool to auto generate c# data models and data access classes from a Dacpac package (output of a SqldbProject in visual studio)
### Code generator Usages:
  - Code generator generated code can give a good insight about how the helper works, scroll down to find the code example of using the db helper without code generator.
  - Install-Package CpsDbHelper.CodeGenerator.BuildTask in a csproject 
  - Config db project path in CodeGeneratorSettings.xml
  - Config the output path in CodeGeneratorSettings.xml
  - Specify the build configuration which will generate code, default Release and Debug
  - Specify the file name patterns and the name spaces.
  - Specify whether to break the build if dacpac is not found
  - Config whether to process primary key/foreign key/unique index/non-unique index, default all true
  - It will generate models out of each table
  - It will generate GetModel methods from each unique index/primary key
  - It will generate GetModels methods from each non-unique index
  - It will generate SaveModel/DeleteModel methods from each unique index/primary key
  - It will generate SaveModel/DeleteModel methods from each foreign key, and load entities to a IList member in foreign entity
  - It will generate classes as partial for you to extend.
  - Identity columns will have their identify scope value returned if a new row is inserted
  - Further, follow the examples in CodeGeneratorSettings.xml shipped with the package to do the following
  	- Define whether to use async for Get/Save/Delete methods overall.
  	- Overall async setting can be overrided by an "AsyncMappings" entry for specific index/primary key with the full name.
  		- If a specific enum class is needed, put a 'EnumMappings' map the type fullname with the column full name in settings, you can actually put any types you want.
  		- If a specific column needs to be overrided, e.g.: dataType/nullable different from db, put a 'ColumnOverrides' entry with the column's full name.
  		- If a Model name's Plural form is not '-s', put 'PluralMappings' entry to make the 'GetModels' method prettier.
  		- If a column/table/index needs to be ignored, put an 'ObjectsToIgnore' entry with its full name.
  		- If a column needs to be readonly, put it to ignore list and define it mannually with the same column name and compatible type.
  	- After generating, the code files will be in the path specified in CodeGeneratorSettings.xml, include them in the project to use.
  	- And we are all good!

## Example DbHelper Usages:

### with the following example models:
```csharp
public class Address
{
	public string City { get; set; }
	public string Country { get; set; }
}
 public class Company
{
	public string Name { get; set; }
	public Address Address { get; set; }
	public string PropertyWithNoMatchingColumn { get; set; }
}
public enum Gender
{
	Male = 0,
	Female = 1
}
public class Person
{
	public int Id { get; set; }
	public string Name { get; set; }
	public Gender Gender { get; set; }
}
```	
### Using Readers:
```csharp
public class UsingReader
  {
      private readonly DbHelperFactory _db = new DbHelperFactory("dummy connection string");

      /// <summary>
      /// executing a stored procedure which returns one set of query and the columns matches the entity property definition
      /// </summary>
      /// <param name="id">an optional id parameter</param>
      /// <returns></returns>
      public IEnumerable<Person> GetPersion(int? id)
      {
          var reader = _db.BeginReader("sp_getPeople")
                       .AddIntInParam("Id", id)
                       .AddOutParam("totalCount", SqlDbType.Int)
                       .AutoMapResult<Person>() //the result key is used to identify result set, the default value can be used once
                       //the same as this way: 
                       //.BeginMapResult<Person>().AutoMap().FinishMap()
                       .Execute();
          //retrieve the result. the reader always use a List<T> to store the returned table. 
          var ret = reader.GetResult<IList<Person>>(); //the result key is default as the one when mapping.
          //retrive the output parameter with it's name.
          var count = reader.GetParamResult<int>("totalCount");
          return ret;
      }

      /// <summary>
      /// executing a stored procedure which returns two sets of results, people and address, of which the columns matches the property definitions
      /// </summary>
      /// <param name="id"></param>
      /// <returns></returns>
      public KeyValuePair<Person, Address> GetPersionAndAddres(int id)
      {
          var reader = _db.BeginReader("sp_getPersonAndAddress")
                       .AddIntInParam("personId", id)
                       .AutoMapResult<Person>("Persion") //we can name the result set this way. or we can still use the default parameter
                       .AutoMapResult<Address>("Address") //map the second reader result, if the previous line uses the default value. this line must specify a different result key
                       .Execute();
          //retrieve the results with result key
          var ret = reader.GetResult<IList<Person>>("Persion");
          var addressRet = reader.GetResult<IList<Address>>("Address");
          return new KeyValuePair<Person, Address>(ret.FirstOrDefault(), addressRet.FirstOrDefault());
      }

      /// <summary>
      /// executing a stored procedure which returns a company with it's address in one set
      /// </summary>
      /// <param name="id"></param>
      /// <returns></returns>
      public Company GetCompanyWithAddress(int id)
      {
          var presavedColumnIndexToImprovePerformance = 0;
          var reader = _db.BeginReader("sp_getCompanyWithAddress")
                       .AddIntInParam("companyId", id)
                       .PreProcessResult(rd => presavedColumnIndexToImprovePerformance = rd.GetOrdinal("Country"))
                       .BeginMapResult<Company>("Company") //it is also ok to leave the result key default 
                       .AutoMap() //can partially use the automap ability, the mapper will map the columns with properties with same names and leave the others to you
                       .MapField<string>("AdditionalColumnName", (item, columnValue) => item.PropertyWithNoMatchingColumn = columnValue)
                       .MapField((item, rd) =>
                           { //customizing a map logic to assign value to non-automapable field.
                               item.Address = new Address()
                                   {
                                       City = rd.Get<string>("City"), //operating at current row. use the reader extension to read value of Column 'City'
                                       Country = rd.Get<string>(presavedColumnIndexToImprovePerformance) //or we can pre-get the ordinal and use it to get better performance
                                   };
                           })
                       .FinishMap()
                       .Execute();
          //retrieve the results with result key
          var ret = reader.GetResult<IList<Company>>("Company");
          return ret.FirstOrDefault();
      }

      /// <summary>
      /// executing a stored procedure which returns a set of addresses
      /// this time we demonstrate controlling the reader without any mapping functionality, and a transaction
      /// </summary>
      /// <returns></returns>
      public Address GetAddresses()
      {
          return _db.BeginReader("sp_getAddresses")
             .DefineResult((rd, helper) =>
                 {
                     var ret = new List<Address>();
                     var ordinalCity = rd.GetOrdinal("City");
                     while (rd.Read())
                     {
                         ret.Add(new Address()
                             {
                                 City = rd.Get<string>(ordinalCity),
                                 Country = rd.Get<string>("Country")
                             });
                     }
                     //this return value is exactly result which the reader stores. we will get it later at the end of this method
                     return ret.FirstOrDefault();
                 })
             .BeginTransaction()
             .Execute(false)
             .CommitTransaction()
             .End()
             .GetResult<Address>();
      }

      /// <summary>
      /// executing a stored procedure which returns a set of addresses
      /// using define list result
      /// </summary>
      /// <returns></returns>
      public Address GetAddresses2()
      {
          return _db.BeginReader("sp_getAddresses")
             .DefineListResult((rd, helper) => new Address()
                 {
                     //the reader will loop the result set and apply this action and put the return value into a list 
                     //the result will be List<Address>
                     City = rd.Get<string>("City"),
                     Country = rd.Get<string>("Country")
                 })
             .BeginTransaction()
             .Execute(false)
             .CommitTransaction()
             .End()
             .GetResult<IList<Address>>()
             .FirstOrDefault();
      }
  }
```
### User NonReaders:
```csharp
public class UsingNonReader
{
	private readonly DbHelperFactory _db = new DbHelperFactory("dummy connection string");
	
	/// <summary>
	/// Save a set of addresses with a user-defined table-valued structure type parameter
	/// and finally selects a count out
	/// </summary>
	/// <param name="addresses"></param>
	public int SaveAddresses(IEnumerable<Address> addresses)
	{
	  return _db.BeginScalar<int>("sp_saveAddresses")
	     .BeginAddStructParam<ScalarHelper<int>, Address>("Addresses")
	     .MapField("City", item => item.City)
	     .MapField("Country", item => item.Country)
	     .FinishMap(addresses)
	      //can also use the auto mapper if the columns are name match and sequence match
	      .AutoMapStructParam("Addresses", addresses)
	     .Execute()
	     .GetResult();
	}
	
	/// <summary>
	/// Save a person and return the id value
	/// </summary>
	/// <param name="person"></param>
	/// <returns></returns>
	public int SavePerson(Person person)
	{
	  return _db.BeginNonQuery("sp_SavePerson")
	            .AddVarcharInParam("name", person.Name)
	            .AddIntInParam("gender", (int) person.Gender)
	            .Execute()
	            .GetReturnValue<int>();
	}
	
	public XElement UsingXmlReader()
	{
	  return _db.BeginXmlReader("sp_returnXml")
	            .Execute()
	            .GetResult();
	}
}
```
