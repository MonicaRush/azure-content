<properties linkid="mobile-services-how-to-html-client" urlDisplayName="HTML Client" pageTitle="How to use an HTML client - Windows Azure Mobile Services feature guide" metaKeywords="Windows Azure Mobile Services, Mobile Service HTML client, HTML client" metaDescription="Learn how to use an HTML client for Windows Azure Mobile Services." metaCanonical="" disqusComments="1" umbracoNaviHide="0" writer="krisragh" />



<div chunk="../chunks/article-left-menu-html.md" />

# How to use an HTML/JavaScript client for Windows Azure Mobile Services

<div class="dev-center-tutorial-selector"> 
	<a href="/en-us/develop/mobile/how-to-guides/how-to-dotnet-client" title=".NET Framework">.NET Framework</a>
	<a href="/en-us/develop/mobile/how-to-guides/how-to-js-client" title="JavaScript">HTML/JavaScript</a> 
	<a href="/en-us/develop/mobile/how-to-guides/how-to-ios-client" title="iOS">iOS</a> 
	<a href="/en-us/develop/mobile/how-to-guides/how-to-android-client" title="Android">Android</a>
</div> 


This guide shows you how to perform common scenarios using an HTML/JavaScript client for Windows Azure Mobile Services. The scenarios covered include querying for data, inserting, updating, and deleting data, authenticating users, and handling errors. If you are new to Mobile Services, you should consider first completing the Mobile Services [Windows Store JavaScript quickstart] or [HTML quickstart]. The quickstart tutorial helps you configure your account and create your first mobile service.


## Table of Contents

- [How to: Create the Mobile Services client]
- [How to: Query data from a mobile service]
	- [Filter returned data]
    - [Sort returned data]
	- [Return data in pages]
	- [Select specific columns]
	- [Look up data by ID]
- [How to: Insert data into a mobile service]
- [How to: Modify data in a mobile service]
- [How to: Delete data in a mobile service]
- [How to: Display data in the user interface]
- [How to: Cache authentication tokens]
- [How to: Handle errors]
- [How to: Use promises]
- [How to: Customize request headers]
- [How to: Use cross-origin resource sharing]

<div chunk="../chunks/mobile-services-concepts.md" />

<h2><a name="create-client"></a><span class="short-header">Create the Mobile Services Client</span>How to: Create the Mobile Services Client</h2>

The following code instantiates the mobile service client object. 

In your web editor, open the HTML file and add the following to the script references for the page:

	        <script src='//client/MobileServices.Web-1.0.0.min.js'></script>

You must replace the placeholder `AppUrl` with the application URL of your mobile service. In the editor, open or create a JavaScript file, and add the following code that defines the `MobileServiceClient` variable, and supply the application URL and application key from the mobile service in the `MobileServiceClient` constructor, in that order. To learn how to obtain the application URL and application key for the mobile service, consult the tutorial [Getting Started with Data in Windows Store JavaScript] or [Getting Started with Data in HTML/JavaScript).

			var MobileServiceClient = WindowsAzure.MobileServiceClient;
		    var client = new MobileServiceClient('AppUrl', 'AppKey');

<h2><a name="querying"></a><span class="short-header">Querying data</span>How to: Query data from a mobile service</h2>

All of the code that accesses or modifies data in the SQL Database table calls functions on the `MobileServiceTable` object. You get a reference to the table by calling the `getTable()` function on an instance of the `MobileServiceClient`.
	
		    var todoItemTable = client.getTable('todoitem');



### <a name="filtering"></a>How to: Filter returned data

The following code illustrates how to filter data by including a `where` clause in a query. It returns all items from `todoItemTable` whose complete field is equal to `false`. `todoItemTable` is the reference to the mobile service table that we created previously. The where function applies a row filtering predicate to the query against the table. It accepts as its argument a JSON object or function that defines the row filter, and returns a query that can be further composed. 
	
			var query = todoItemTable.where({
			    complete: false
			}).read().done(function (results) {
			    alert(JSON.stringify(results));
			}, function (err) {
			    alert("Error: " + err);
			});

By adding calling `where` on the Query object and passing an object as a parameter, we are  instructing Mobile Services to return only the rows whose `complete` column contains the `false` value. Also, look at the request URI below, and notice that we are modifying the query string  itself:

			GET /tables/todoitem?$filter=(complete+eq+false) HTTP/1.1			

You can view the URI of the request sent to the mobile service by using message inspection software, such as browser developer tools or Fiddler.

This request would normally be translated roughly into the following SQL query on the server side:
			
			SELECT * 
			FROM TodoItem 			
			WHERE ISNULL(complete, 0) = 0

The object which is passed to the `where` method can have an arbitrary number of parameters, and they’ll all be interpreted as AND clauses to the query. For example, the line below:

			query.where({
			   complete: false,
			   assignee: "david",
			   difficulty: "medium"
			}).read().done(function (results) {
			   alert(JSON.stringify(results));
			}, function (err) {
			   alert("Error: " + err);
			});

Would be roughly translated (for the same request shown before) as
			
			SELECT * 
			FROM TodoItem 
			WHERE ISNULL(complete, 0) = 0
			      AND assignee = 'david'
			      AND difficulty = 'medium'

The `where` statement above and the SQL query above find incomplete items assigned to "David" of "medium" difficulty.

There is, however, another way to write the same query. A `.where` call on the Query object will add an `AND` expression to the `WHERE` clause, so we could have written that in three lines instead:

			query.where({
			   complete: false
			});
			query.where({
			   assignee: "david"
			});
			query.where({
			   difficulty: "medium"
			});

Or using the fluent API:

			query.where({
			   complete: false
			})
			   .where({
			   assignee: "david"
			})
			   .where({
			   difficulty: "medium"
			});

The two methods are equivalent and may be used interchangeably. All the `where` calls so far take an object with some parameters, and are compared for equality against the data from the database. There is, however, another overload for the query method, which takes a function instead of the object. In this function we can then write more complex expressions, using operators such as inequality and other relational operations. In these functions, the keyword `this` binds to the server object.

The body of the function is translated into an OData boolean expression which is passed to a query string parameter. It is possible to pass in a function that takes no parameters, like so:

		    query.where(function () {
		       return this.assignee == "david" && (this.difficulty == "medium" || this.difficulty == "low");
		    }).read().done(function (results) {
		       alert(JSON.stringify(results));
		    }, function (err) {
		       alert("Error: " + err);
		    });


If passing in a function with parameters, any arguments after the `where` clause are bound to the function parameters in order. Any objects which come from the outside of the function scope MUST be passed as parameters – the function cannot capture any external variables. In the next two examples, the argument "david" is bound to the parameter `name` and in the first example, the argument "Medium" is also bound to the parameter `level`. Also, the function must consist of a single `return` statement with a supported expression, like so:
					
			 query.where(function (name, level) {
			    return this.assignee == name && this.difficulty == level;
			 }, "david", "Medium").read().done(function (results) {
			    alert(JSON.stringify(results));
			 }, function (err) {
			    alert("Error: " + err);
			 });

So, as long as we follow the rules, we can add more complex filters to our database queries, like so:

		    query.where(function (name) {
		       return this.assignee == name &&
		          (this.difficulty == "Medium" || this.difficulty == "Low");
		    }, "david").read().done(function (results) {
		       alert(JSON.stringify(results));
		    }, function (err) {
		       alert("Error: " + err);
		    });

It is possible to combine `where` with `orderBy`, `take`, and `skip`. See the next section for details.

### <a name="sorting"></a>How to: Sort returned data

The following code illustrates how to sort data by including an `orderBy` or `orderByDescending` function in the query. It returns items from `todoItemTable` sorted ascending by the `text` field. By default, the server returns only the first 50 elements. 

<div class="dev-callout"><strong>Note</strong> <p>A server-driven page size us used by default to prevent all elements from being returned. This keeps default requests for large data sets from negatively impacting the service. </p> </div>
>
You may increase the number of items to be returned by calling `take` as described in the next section. `todoItemTable` is the reference to the mobile service table that we created previously.

			var ascendingSortedTable = todoItemTable.orderBy("text").read().done(function (results) {
			   alert(JSON.stringify(results));
			}, function (err) {
			   alert("Error: " + err);
			});

			var descendingSortedTable = todoItemTable.orderByDescending("text").read().done(function (results) {
			   alert(JSON.stringify(results));
			}, function (err) {
			   alert("Error: " + err);
			});

			var descendingSortedTable = todoItemTable.orderBy("text").orderByDescending("text").read().done(function (results) {
			   alert(JSON.stringify(results));
			}, function (err) {
			   alert("Error: " + err);
			});

### <a name="paging"></a>How to: Return data in pages

The following code shows how to implement paging in returned data by using the `take` and `skip` clauses in the query.  The following query, when executed, returns the top three items in the table. 

			var query = todoItemTable.take(3).read().done(function (results) {
			   alert(JSON.stringify(results));
			}, function (err) {
			   alert("Error: " + err);
			});

Notice that the `take(3)` method was translated into the query option `$top=3` in the query URI.

The following revised query skips the first three results and returns the next three after that. This is effectively the second "page" of data, where the page size is three items.

			var query = todoItemTable.skip(3).take(3).read().done(function (results) {
			   alert(JSON.stringify(results));
			}, function (err) {
			   alert("Error: " + err);
			});

Again, you can view the URI of the request sent to the mobile service. Notice that the `skip(3)` method was translated into the query option `$skip=3` in the query URI.

This is a simplified scenario of passing hard-coded paging values to the `take` and `skip` functions. In a real-world app, you can use queries similar to the above with a pager control or comparable UI to let users navigate to previous and next pages. 

### <a name="selecting"></a>How to: Select specific columns

You can specify which set of properties to include in the results by adding a `select` clause to your query. For example, the following code returns the `id`, `complete`, and `text` properties from each row in the `todoItemTable`:

			var query = todoItemTable.select("id", "complete", "text").read().done(function (results) {
			   alert(JSON.stringify(results));
			}, function (err) {
			   alert("Error: " + err);
			})
	
Here the parameters to the select function are the names of the table's columns that you want to return. 


All the functions described so far are additive, so we can just keep calling them and we’ll each time affect more of the query. One more example:


		    query.where({
		       complete: false
		    })
		       .select('id', 'assignee')
		       .orderBy('assignee')
		       .take(10)
		       .read().done(function (results) {
		       alert(JSON.stringify(results));
		    }, function (err) {
		       alert("Error: " + err);

### <a name="lookingup"></a>How to: Look up data by ID

The `lookup` function takes only the `id` value, and returns the object from the database with that ID.

			todoItemTable.lookup(3).done(function (result) {
			   alert(JSON.stringify(result));
			}, function (err) {
			   alert("Error: " + err);
			})

<h2><a name="inserting"></a><span class="short-header">Inserting data</span>How to: Insert data into a mobile service</h2>

The following code illustrates how to insert new rows into a table. The client requests that a row of data be inserted by sending a POST request to the mobile service. The request body contains the data to be inserted, as a JSON object. 

			todoItemTable.insert({
			   text: "New Item",
			   complete: false
			})

This inserts data from the supplied JSON object into the table. You can also specify a callback function to be invoked when the insertion is complete:

			todoItemTable.insert({
			   text: "New Item",
			   complete: false
			}).read().done(function (result) {
			   alert(JSON.stringify(result));
			}, function (err) {
			   alert("Error: " + err);
			});

<h2><a name="modifying"></a><span class="short-header">Modifying data</span>How to: Modify data in a mobile service</h2>

The following code illustrates how to update data in a table. The client requests that a row of data be updated by sending a PATCH request to the mobile service. The request body contains the specific fields to be updated, as a JSON object. It updates an existing item in the table `todoItemTable`. 

			todoItemTable.update({
			   id: idToUpdate,
			   text: newText
			})

The first parameter specifies the instance to update in the table, as specified by its ID. 

You can also specify a callback function to be invoked when the update is complete:

			todoItemTable.update({
			   id: idToUpdate,
			   text: newText
			}).read().done(function (result) {
			   alert(JSON.stringify(result));
			}, function (err) {
			   alert("Error: " + err);
			});
	
<h2><a name="deleting"></a><span class="short-header">Deleting data</span>How to: Delete data in a mobile service</h2>

The following code illustrates how to delete data from a table. The client requests that a row of data be deleted by sending a DELETE request to the mobile service. It deletes an existing item in the table todoItemTable. 

			todoItemTable.del({
			   id: idToDelete
			})

The first parameter specifies the instance to delete in the table, as specified by its ID. 

You can also specify a callback function to be invoked when the delete is complete:	
	
			todoItemTable.del({
			   id: idToDelete
			}).read().done(function () {
			   /* Do something */
			}, function (err) {
			   alert("Error: " + err);
			});	

<h2><a name="binding"></a><span class="short-header">Displaying data</span>How to: Display data in the user interface</h2>

This section shows how to display returned data objects using UI elements. To query items in `todoItemTable` and display it in a very simple list, you can run the following example code. No selection, filtering or sorting of any kind is done. 

			var query = todoItemTable;
		
			query.read().then(function (todoItems) {
			   // The space specified by 'placeToInsert' is an unordered list element <ul> ... </ul>
			   var listOfItems = document.getElementById('placeToInsert');
			   for (var i = 0; i < todoItems.length; i++) {
			      var li = document.createElement('li');
			      var div = document.createElement('div');
			      div.innerText = todoItems[i].text;
			      li.appendChild(div);
			      listOfItems.appendChild(li);
			   }
			}).read().done(function (results) {
			   alert(JSON.stringify(results));
			}, function (err) {
			   alert("Error: " + err);
			});

In a Windows Store app, the results of a query can be used to create a [WinJS.Binding.List] object, which can be bound as the data source for a [ListView] object. For more information, see [Data binding (Windows Store apps using JavaScript and HTML)].

<h2><a name="caching"></a><span class="short-header">Authentication</span>How to: Cache authentication tokens</h2>

Mobile Services supports the following existing identity providers that you can use to authenticate users:

- Facebook
- Google 
- Microsoft Account
- Twitter

You can set permissions on tables to restrict access for specific operations to only authenticated users. You can also use the ID of an authenticated user to modify requests. For more information, see [Get started with authentication].

To login, you can use the following method:

	        client.login("facebook");

With the above code snippet, the variable login will be set to a promise that completes when the login process is finished. The user is authenticated by using a Facebook login. 

<div class="dev-callout"><strong>Note</strong> <p>If you are using an identity provider other than Facebook, change the value passed to the `login` method above to one of the following: `microsoftaccount`, `facebook`, `twitter`, or `google`.</p> </div>
>

To enable single sign-on, where the client has already logged-in with one of the identity providers, it is possible to submit an access token obtained using an SDK. The format for the token which should be passed to the method is `{"access_token": **the-actual-access-token**}` if the access token is from the Facebook or Google SDK. If the access token is from a Live SDK, it must be specified instead as `{"authenticationToken": **accessToken**}`. What needs to be passed in place of "accessToken" is the session.authentication_token property of the result to the call to [WL.login].

The "actual-access-token" is the value you receive from the SDK. Your code should look somehow like the following:
	
			var login = client.login(“facebook”, jsonSerializedToken).done(function (results) {
			   alert(JSON.stringify(results));
			}, function (err) {
			   alert("Error: " + err);
			});

In general, it is simple to persist a user token in [sessionStorage] (or [localStorage]), use it to prepopulate currentUser when the app starts up, and to log out:

		    // After logging in
		    sessionStorage.loggedInUser = JSON.stringify(client.currentUser);

		     // Just after instantiating the client instance
		    if (sessionStorage.loggedInUser) {
		       client.currentUser = JSON.parse(sessionStorage.loggedInUser);
		    }

		     // Log out
		    client.logout();
		    sessionStorage.loggedInUser = null;

You can cache an authentication token. Do this to prevent users from having to authenticate again while the token is still valid. All you have to do is manually ‘log the user in’ rather than use the built in login methods. To do this, you just need to set the user object on the Mobile Service client instance, here’s how:
		
			client.currentUser = {
			   userId: "123456789",
			   mobileServiceAuthenticationToken: "<your-users-JSON-web-token>"
			};

<h2><a name="errors"></a><span class="short-header">Error handling</span>How to: Handle errors</h2>

There are several ways to encounter, validate, and work around errors in Mobile Services. 

As an example, server scripts are registered in a mobile service and can be used to perform a wide range of operations on data being inserted and updated, including validation and data modification. Imagine defining and registering a server script that validate and modify data, like so:

			function insert(item, user, request) {
			   if (item.text.length > 10) {
				  request.respond(statusCodes.BAD_REQUEST, { error: "Text cannot exceed 10 characters" });
			   } else {
			      request.execute();
			   }
			}

This server-side script validates the length of string data sent to the mobile service and rejects strings that are too long, in this case longer than 10 characters.

Now that the mobile service is validating data and sending error responses on the server-side, you can update your HTML app to be able to handle error responses from validation.

		todoItemTable.insert({
		   text: itemText,
		   complete: false
		})
		   .then(function (results) {
		   alert(JSON.stringify(results));
		}, function (error) {
		   alert(JSON.parse(error.request.responseText).error);
		});


To tease this out even further, you pass in the error handler as the second argument each time you perform data access: 
			
			function handleError(message) {
			   if (window.console && window.console.error) {
			      window.console.error(message);
			   }
			}

			client.getTable("tablename").read().then(function (data) { /* do something */ }, handleError);

<h2><a name="promises"></a><span class="short-header">Promises</span>How to: Use promises</h2>

Promises provide a mechanism to schedule work to be done on a value that has not yet been computed. It is an abstraction for managing interactions with asynchronous APIs. 

The `done` promise allows you to specify the work to be done on the fulfillment of the promised value, and the error handling to be performed if the promise fails to fulfill a value. After the handlers have finished executing, this function throws any error that would have been returned from then as a promise in the error state. For more information, see [`done`].

			promise.done(onComplete, onError);

Like so:
	
			var query = todoItemTable;
			query.read().done(function (results) {
			   alert(JSON.stringify(results));
			}, function (err) {
			   alert("Error: " + err);
			});

The `then` promise allows you to specify the work to be done on the fulfillment of the promised value, and the error handling to be performed if the promise fails to fulfill a value. For more information, see [`then`].

			promise.then(onComplete, onError).done( /* Your success and error handlers */ );

Like so:

			var query = todoItemTable;
			query.read().done(function (results) {
			   alert(JSON.stringify(results));
			}, function (err) {
			   alert("Error: " + err);
			});

You can use promises in a number of different ways. You can chain promise operations by calling `then` or `done` on the promise that is returned by the previous `then` function. Use `then` for an intermediate stage of the operation (for example `.then().then()`), and `done` for the final stage of the operation (for example `.then().then().done()`). 
	
 			todoItemTable.insert({
 			   text: ’foo’
 			}).then(function (inserted) {
 			   inserted.newField = 123;
 			   return todoItemTable.update(inserted);
 			}).done(function (insertedAndUpdated) {
 			   alert(JSON.stringify(insertedAndUpdated));
 			})

[Learn more about the  differences between then and done]. The error callback is invoked if there is an error while processing a promise. Learn more about [how to handle errors in promises].

<h2><a name="customizing"></a><span class="short-header">Customize request headers</span>How to: Customize client request headers</h2>

You can send custom request headers using the `withFilter` function, reading and writing arbitrary properties of the request about to be sent within the filter. You may want to add such a custom HTTP header if a server-side script needs it or may be enhanced by it. 

			var client = new WindowsAzure.MobileServiceClient('https://your-app-url', 'your-key')
			   .withFilter(function (request, next, callback) {
			   request.headers.MyCustomHttpHeader = "Some value";
			   next(request, callback);
			});

Filters are used for a lot more than customizing request headers. They can be used to examine or change requests, examine or change  responses, bypass networking calls, send multiple calls, etc.

<h2><a name="hostnames"></a><span class="short-header">Use CORS</span>How to: Use cross-origin resource sharing</h2>

To control which web sites are allowed to interact with and send requests to your mobile service, you should configure a whitelist of allowed hostnames. By default, new Mobile Services instruct browsers to permit access only from `localhost`.  This configuration is not necessary for WinJS applications.

Cross-Origin Resource Sharing (CORS) allows JavaScript code running in a browser on an external hostname to interact with your Mobile Service. When deploying an HTML5 front-end app to a production environment, make sure to add the host name of the website you use to host it to the Cross-Origin Resource Sharing (CORS) whitelist using the Configure tab. You can use wildcards if required.

<!-- Anchors. -->
[How to: Create the Mobile Services client]: #create-client
[How to: Query data from a mobile service]: #querying
[Filter returned data]: #filtering
[Sort returned data]: #sorting
[Return data in pages]: #paging
[Select specific columns]: #selecting
[Look up data by ID]: #lookingup
[How to: Display data in the user interface]: #binding
[How to: Insert data into a mobile service]: #inserting
[How to: Modify data in a mobile service]: #modifying
[How to: Delete data in a mobile service]: #deleting
[How to: Authenticate users]: #authentication
[How to: Cache authentication tokens]: #caching
[How to: Handle errors]: #errors
[How to: Use promises]: #promises
[How to: Customize request headers]: #customizing
[How to: Use cross-origin resource sharing]: #hostnames
[How to: Create the Mobile Services client]: #create-client
[How to: Query data from a mobile service]: #querying
[Filter returned data]: #filtering
[Sort returned data]: #sorting
[Return data in pages]: #paging
[Select specific columns]: #selecting
[Look up data by ID]: #lookingup
[How to: Display data in the user interface]: #binding
[How to: Insert data into a mobile service]: #inserting
[How to: Modify data in a mobile service]: #modifying
[How to: Delete data in a mobile service]: #deleting
[How to: Authenticate users]: #authentication
[How to: Cache authentication tokens]: #caching
[How to: Handle errors]: #errors
[How to: Use promises]: #promises
[How to: Customize request headers]: #customizing
[How to: Use cross-origin resource sharing]: #hostnames

<!-- URLs. -->
[Get started with Mobile Services]: ../tutorials/mobile-services-get-started-html.md
[Mobile Services SDK]: http://go.microsoft.com/fwlink/?LinkId=257545
[Getting Started with Data]: http://www.windowsazure.com/en-us/develop/mobile/tutorials/get-started-with-data-html/
[Get started with authentication]: ./mobile-services-get-started-with-users-html.md
[then]: http://msdn.microsoft.com/en-us/library/windows/apps/br229728.aspx
[done]: http://msdn.microsoft.com/en-us/library/windows/apps/hh701079.aspx
[Learn more about the  differences between then and done]: http://msdn.microsoft.com/en-us/library/windows/apps/hh700334.aspx
[how to handle errors in promises]: http://msdn.microsoft.com/en-us/library/windows/apps/hh700337.aspx
[WL.login]: http://msdn.microsoft.com/en-us/library/live/hh550845.aspx
[sesssionStorage]: http://msdn.microsoft.com/en-us/library/cc197062(v=vs.85).aspx
[localStorage]: http://msdn.microsoft.com/en-us/library/cc197062(v=vs.85).aspx
[WinJS.Binding.List]: http://msdn.microsoft.com/en-us/library/windows/apps/Hh700774.aspx
[ListView]: http://msdn.microsoft.com/en-us/library/windows/apps/br211837.aspx
[Data binding (Windows Store apps using JavaScript and HTML)]: http://msdn.microsoft.com/en-us/library/windows/apps/hh758311.aspx
[Windows Store JavaScript quickstart]: http://www.windowsazure.com/en-us/develop/mobile/tutorials/get-started
[HTML quickstart]: http://www.windowsazure.com/en-us/develop/mobile/tutorials/get-started-html
[Getting Started with Data in Windows Store JavaScript]: http://www.windowsazure.com/en-us/develop/mobile/tutorials/get-started-with-data-js
[Getting Started with Data in HTML/JavaScript): http://www.windowsazure.com/en-us/develop/mobile/tutorials/get-started-with-data-html/