---
layout: post
title:  "Oracle APEX Lesson: Free to Play Games"
date:   2021-01-23
categories: lesson
author: Jose
---
## Introduction

In this lesson, you will learn how to:

* Query a web API
* Parse the results and store them in a table
* Display the information as Cards on a web page

At the end of the lesson, your page will look like this:

<a href="/assets/free-to-play/page_01.png" target="_blank"><img src="/assets/free-to-play/page_01.png"/></a>

## The API

For this project, we will download data from a free web API called [Free to Play Game API](https://www.freetogame.com/api/games). I chose this API for three reasons:

1.  It does not require authentication, which makes it very easy to integrate.
2.  The output is straightforward; it contains all the game information in one object with no sub arrays, making it very easy to parse.
3.  I like video games, even though I do not have time to play them.

## Tools

To complete this tutorial, you need an Oracle APEX workspace. You can also install additional tools to make your life easier.

#### Oracle APEX (Required)

If you do not have an Oracle APEX workspace, <a href="https://www.oracle.com/cloud/free/" target="_blank">sign up for an Always Free account in Oracle Cloud</a> and follow these instructions: <a href="https://docs.oracle.com/en/cloud/paas/autonomous-database/adbsa/apex-access-admin-services.html#GUID-D078638E-4F59-46CA-A14D-DAEBE1514BE8" target="_blank">Access Oracle Application Express Administration Services</a>

#### SQL Developer (Optional)

You can do all the work from this tutorial directly in APEX, but I recommend writing and testing your PL/SQL code in <a href="https://www.oracle.com/tools/downloads/sqldev-downloads.html" target="_blank">SQL Developer</a>.

#### REST API Client (Optional)

You should also install a REST API client to help you explore the web API before coding. In this lesson, I use <a href="https://insomnia.rest/download/core" target="_blank">Insonmia Core</a>. You can also use <a href="https://www.postman.com" target="_blank">Postman</a>.

## Exploring the API

Let us start by exploring the web API and coming up with a plan to create the application. We will use the "Live games list" API call to retrieve a list of games. We do this by running a GET call against <a href="https://www.freetogame.com/api/games" target="_blank">https://www.freetogame.com/api/games</a>.

Fire up Insomnia Core. Click on the + button to create a New Request.

<a href="/assets/free-to-play/insomnia_01.png" target="_blank"><img src="/assets/free-to-play/insomnia_01.png"/></a>

Name the request "Live games list," make sure you are using a "GET" request, and click "Create."

<a href="/assets/free-to-play/insomnia_02.png" target="_blank"><img src="/assets/free-to-play/insomnia_02.png"/></a>

Then, paste the URL in the box and click "Send." You should now see a preview of the results on the right-hand pane.

<a href="/assets/free-to-play/insomnia_03.png" target="_blank"><img src="/assets/free-to-play/insomnia_03.png"/></a>

Inspecting the output gives us two essential pieces of information. First, the results contain over 183,000 characters, which will help us determine the data type we need to hold them. And second, we can see the name-value pairs (like title: "Dota 2"), which tells us how to model the table.

## Creating the table

To create the table, we will use <a href="https://docs.oracle.com/en/database/oracle/application-express/20.2/aeutl/using-quick-sql.html#GUID-21EE36C2-F814-48C0-90EA-7D464E9014FD" target="_blank">Quick SQL</a>, a markdown-like syntax that generates SQL code and makes life easier. Quick SQL has many features, and you should take some time to learn them. But for this lesson, all we need to know is we must define strings as `vc` (varchar2) and numbers as `int`. 

When defining varchars, set the length in parenthesis. For example, define a column that is 25 characters in length like this: `my_column vc(25)`.

By examining the API’s output, we can define our table like so:

```sql
game_id int
title vc(225)
thumbnail vc(500)
short_description vc(4000)
game_url vc(500)
gender vc(125)
platform vc(225)
publisher vc(225)
developer vc(225)
release_date vc(25)
freetogame_profile_url vc(500)
```

I want to clarify some things about the model. First, note that the game record has a unique id named id. We will change this column's name to `game_id` and let Quick SQL create an identity column called id (an auto-generated sequential number).

Note the `release_date` is a date, but I decided to use a varchar2 because I ran into an error trying to insert the data into the table. The reason is that a few games have a release day of 00, which is an invalid day number in Oracle (and in real life).

Now, log in to your Oracle APEX workspace, click on "SQL Workshop" -> "Utilities" and "Quick SQL."

<a href="/assets/free-to-play/quick_sql_01.png" target="_blank"><img src="/assets/free-to-play/quick_sql_01.png"/></a>

Type the table name `free_to_games`, and create the columns under it. Please note you must indent the column definitions for Quick SQL to recognize them.

<a href="/assets/free-to-play/quick_sql_02.png" target="_blank"><img src="/assets/free-to-play/quick_sql_02.png"/></a>

Click on "Save SQL Script," give it a name like "Free to Games Table," and finally click on "Review and Run."

You should now see the final code. Click on "Run" to execute it. 

<a href="/assets/free-to-play/quick_sql_03.png" target="_blank"><img src="/assets/free-to-play/quick_sql_03.png"/></a>

On the next screen, click on "Run Now" to confirm. 

<a href="/assets/free-to-play/quick_sql_04.png" target="_blank"><img src="/assets/free-to-play/quick_sql_04.png"/></a>

If everything goes right, you should have 0 errors.

<a href="/assets/free-to-play/quick_sql_05.png" target="_blank"><img src="/assets/free-to-play/quick_sql_05.png"/></a>

Take a look at your table by going to "SQL Workshop" -> "Object Browser." Make sure that everything looks correct.

<a href="/assets/free-to-play/quick_sql_06.png" target="_blank"><img src="/assets/free-to-play/quick_sql_06.png"/></a>

## Connecting to the API

Fire up SQL Developer and connect to your Oracle database to get started. For instructions on how to connect SQL Developer to your Oracle Cloud database <a href="https://docs.oracle.com/en/cloud/paas/atp-cloud/atpgs/autonomous-connect-sql-developer.html#GUID-14217939-3E8F-4782-BFF2-021199A908FD" target="_blank">click here</a>.

Open a new Worksheet and declare a few variables. 

First, we need to declare the URL we are going to call.

```sql
declare
    l_url varchar2(225) := 'https://www.freetogame.com/api/games';
```
Second, we need two variables to hold the API's response and the response code.

```sql
    ...
    l_response clob;    
    l_response_code int;
```

Remember that our JSON output is over 183K characters. That is why we define the response variable as a CLOB, or Character Large Object, which can hold 4GB of data. The VARCHAR2 data type can only have 32k, so it is not suitable for this operation.

Next, we are going to execute our API call using the <a href="https://docs.oracle.com/en/database/oracle/application-express/20.2/aeapi/MAKE_REQUEST-Function.html#GUID-2973A635-6A7D-426B-8ED0-F458D96FD520" target="_blank">make_request()</a> function of the apex_web_service API. Pass the `l_url` variable to the `p_url` argument, and the value `GET` to the `p_http_method` argument.

```sql
...
begin
    -- Send the API request
    l_response := apex_web_service.make_rest_request ( 
        p_url => l_url, 
        p_http_method => 'GET' 
    );
```

We are also going to store the HTTP response code in the `l_response_code` variable.

```sql
    l_response_code := apex_web_service.g_status_code;
```
And finally, let us print the code to see if this works.

```sql
    dbms_output.put_line('Response code: ' || l_response_code);
end;
```

Now you can run the script and see the result. 

<a href="/assets/free-to-play/plsql_01.png" target="_blank"><img src="/assets/free-to-play/plsql_01.png"/></a>

If everything went well, your response code should be 200 (OK).

## Converting the output to JSON

The next steps are to convert the JSON output stored in the CLOB into a JSON data type and iterate through the new object to test the parsing function.

Let us declare a few more variables.

```sql
    l_values apex_json.t_values;
    l_count int;
    l_json_index int default 1;
```

The `l_values` variable will hold the new JSON object. The `l_count` variable will store the number of games. And the `l_json_index` variable will help us iterate through the games.

Let us start piecing this together. But first, we want to ensure that our code only runs if our response code equals 200. Otherwise, the code will fail, and Oracle will return an error.

Add an if statement that evaluates the response code and exits if it does not equal 200.

```sql
    ...
    l_response_code := apex_web_service.g_status_code;
    dbms_output.put_line('Response code: ' || l_response_code);
    -- Exit if the call fails
    if l_response_code != 200 then
        return;
    end if;
```

Now, convert the `l_response` CLOB to a JSON object and store it in `l_values`. For this, use the `parse()` function of the `apex_json package`.

```sql
    ...
    -- Convert the output to a JSON object    
    apex_json.parse(
        p_values => l_values, 
        p_source => l_response
    );
```

To get the number of games, use the `get_count()` function from the same package.

```sql
    ...
    -- Count the number of games 
    l_count := apex_json.get_count(
        p_path => '.',
        p_values => l_values
    );
```

Print the number of games to see if it works.

```sql
    ...
    dbms_output.put_line('Count games: ' || l_count);   
```

Run the script to see if everything looks good.

<a href="/assets/free-to-play/plsql_02.png" target="_blank"><img src="/assets/free-to-play/plsql_02.png"/></a>

When I ran it, I received 349 games. Do not worry if the count is different for you, as the games list may have changed since I published this lesson.

## Parsing the JSON

Now that we stored the games in a neat JSON object, it is time to parse. We are going to use two functions: <a href="https://docs.oracle.com/en/database/oracle/application-express/20.2/aeapi/GET_VARCHAR2-Function.html#GUID-BC908E35-57DC-46B3-BFB2-EFF092855D25" target="_blank"> and <a href="https://docs.oracle.com/en/database/oracle/application-express/20.2/aeapi/GET_NUMBER-Function.html#GUID-272C06D5-17F8-474D-B1A3-6ED0675F89E2" target="_blank">get_number</a>.

Declare a new variable to hold an index number. This variable will help us loop through all the games.

```sql
    ...
    l_json_index int default 1;
    ...
```

But first, add some error trapping to ensure we do not try to parse an empty object, which would throw an ugly exception error.

```sql
    -- Exit if we didn't receive any games
    if l_count = 0 then
        return;
    end if;
```

After setting your error trap, add a while loop.

```sql
    ...
    -- Loop through the JSON object
    while l_count >= l_json_index
    loop
        -- TODO: Do some parsing here
        l_json_index := l_json_index + 1;
    end loop;
```

At this point, we want to test parsing one data element. Once we get it working, we can flesh out the section. So, for now, add a `put_line()` that prints the game title. Since the title is text, use the `get_varchar2()`</a> function from the `apex_json` package.

```sql
        ...
        dbms_output.put_line(
            l_count || ' ' ||
            apex_json.get_varchar2(
                p_path => '[%d].title',
                p0 => l_json_index,
                p_values => l_values
            )
        );  
        ...
```

Run the script, and you should see a list of all the games.

<a href="/assets/free-to-play/plsql_03.png" target="_blank"><img src="/assets/free-to-play/plsql_03.png"/></a>

The next step is to parse all the data elements for each game and insert them in our table.

## Collecting the data

Now that we can call the API, convert the results to JSON, and loop through each game, we can build each row to insert them in our table. For this, we are going to declare two new types.

First, declare the `game_record` type, which will hold the column definitions.

```sql
    type game_record is record(
        -- Add the record columns here
    );
```

To declare the columns and their data types, we will reference the table we created earlier. All we need is to do is type the column names and use the following syntax for the data type:

```sql
column_name schema.table_name.column_name%type
```

So for example, to declare game_id, type:

```sql
game_id dev.free_to_games.game_id%type,
```

Your new `record` type should look like this:

```sql
    type game_record is record(
        game_id dev.free_to_games.game_id%type,
        title dev.free_to_games.title%type,
        thumbnail dev.free_to_games.thumbnail%type,
        short_description dev.free_to_games.short_description%type,
        game_url dev.free_to_games.game_url%type,
        platform dev.free_to_games.platform%type,
        publisher dev.free_to_games.publisher%type,
        developer dev.free_to_games.developer%type,
        release_date dev.free_to_games.release_date%type,
        freetogame_profile_url dev.free_to_games.freetogame_profile_url%type
    );
    ...
```
The next step is to declare a `table` type to hold the collection of records. 

```sql
    type games_table is a table of game_record index by simple_integer;
    ...
```
Then declare a variable to hold the table of game records.

```sql
    l_games_table games_table;
    ...
``` 

And finally, declare a variable to hold the table index, which will help us set and retrieve values from the table.

```sql
    l_table_index int default 0;
    ...
```

Note that we have two index variables: l_json_index and l_table_index. The first is to loop through the JSON object, which starts at position 1, and the second is to loop through the table, starting at position 0. The reason is that when you parse JSON, your first node is at position 1, but when you access a table, the first position is at 0.

Now let us test our new types and variables. Instead of printing the title directly from the JSON object, set the title in a `game_record` in the `games_table` variable and then print it from the table variable. To do this, use the `get_varchar2()` function from the `apex_json` package. Pass the value of `[%d].title` to the `p_path` argument, the `l_json_index` variable to `p0`, and the `l_values` variable to the `p_values`.

```sql
        l_games_table(l_table_index).title := apex_json.get_varchar2(
            p_path => '[%d].title', 
            p0 => l_json_index, 
            p_values => l_values
        ); 
        dbms_output.put_line(l_games_table(l_table_index).title);           
        l_table_index := l_table_index + 1;
        l_json_index := l_json_index + 1;
    ...
```

Run the script to see the result.

<a href="/assets/free-to-play/plsql_04.png" target="_blank"><img src="/assets/free-to-play/plsql_04.png"/></a>

At this point, we can try to store all the data in our table variable, which will make it very easy to insert into a real table. Remove the `put_line()`, and set all the values in your table records.

```sql
            l_games_table(l_table_index).game_id := apex_json.get_number(p_path => '[%d].id', p0 => l_json_index, p_values => l_values); 
            l_games_table(l_table_index).title := apex_json.get_varchar2(p_path => '[%d].title', p0 => l_json_index, p_values => l_values); 
            l_games_table(l_table_index).thumbnail := apex_json.get_varchar2(p_path => '[%d].thumbnail', p0 => l_json_index, p_values => l_values); 
            l_games_table(l_table_index).short_description := apex_json.get_varchar2(p_path => '[%d].short_description', p0 => l_json_index, p_values => l_values); 
            l_games_table(l_table_index).game_url := apex_json.get_varchar2(p_path => '[%d].game_url', p0 => l_json_index, p_values => l_values); 
            l_games_table(l_table_index).genre := apex_json.get_varchar2(p_path => '[%d].genre', p0 => l_json_index, p_values => l_values);
            l_games_table(l_table_index).platform := apex_json.get_varchar2(p_path => '[%d].platform', p0 => l_json_index, p_values => l_values);     
            l_games_table(l_table_index).publisher := apex_json.get_varchar2(p_path => '[%d].publisher', p0 => l_json_index, p_values => l_values);           
            l_games_table(l_table_index).developer := apex_json.get_varchar2(p_path => '[%d].developer', p0 => l_json_index, p_values => l_values);           
            l_games_table(l_table_index).release_date := apex_json.get_varchar2(p_path => '[%d].release_date', p0 => l_json_index, p_values => l_values);           
            l_games_table(l_table_index).freetogame_profile_url := apex_json.get_varchar2(p_path => '[%d].freetogame_profile_url', p0 => l_json_index, p_values => l_values);
            ...
```

I know this is not lovely. To clean this up, we could create a function that handles the parsing to simplify the code. But for now, this is enough to complete the next steps.

## Populating the table

Now we are ready to insert the data into our table. But first, let us add some error trapping to catch bad values from the web API. To do so, wrap the parsing section in a begin/end statement, and add an exception handler for `value_error`.

To test this, change the `game_id `value to `[%d].title`. That will raise the error because the title is not a number. Add a `return` after the `put_line()` to halt the script and prevent a buffer overflow from all the raised errors.

```sql
        begin
            l_games_table(l_table_index).game_id := apex_json.get_number(p_path => '[%d].title', p0 => l_json_index, p_values => l_values); 
            -- etc.
            ...     
        exception when value_error then
            dbms_output.put_line('Raised VALUE_ERROR');
            return;
        end;
```

Run the script to test your code. If everything works, we can insert the data without worrying about insert errors due to wrong data types.

<a href="/assets/free-to-play/plsql_05.png" target="_blank"><img src="/assets/free-to-play/plsql_05.png"/></a>

Now that we have all our games in the `l_games_table` variable, we can insert all the rows in the table. To do so, we are using the <a href="https://docs.oracle.com/en/database/oracle/oracle-database/21/lnpls/FORALL-statement.html#GUID-C45B8241-F9DF-4C93-8577-C840A25963DB">FORALL statement</a>, which speeds things up by inserting in batches instead of one record at a time.

```sql
    ...
    forall x in l_games_table.first..l_games_table.last
```

You can name the index_name whatever you want. I use `x` for simplicity. Note the bounds clause after the `in` statement. It means we are going to insert all the rows from first to last.

Next, write the `insert` statement.

```sql
    ...
    insert into dev.free_to_games
    (
        game_id,
        title,
        thumbnail,
        short_description,
        game_url,
        genre,
        platform,
        publisher,
        developer,
        release_date,
        freetogame_profile_url
    )
    values (
        l_games_table(x).game_id,
        l_games_table(x).title,
        l_games_table(x).thumbnail,
        l_games_table(x).short_description,
        l_games_table(x).game_url,
        l_games_table(x).genre,
        l_games_table(x).platform,
        l_games_table(x).publisher,           
        l_games_table(x).developer,
        l_games_table(x).release_date,          
        l_games_table(x).freetogame_profile_url
    );
    commit;      
```

Run the script. 

If you do not receive any errors, check your table with a select statement.

```sql
    select * from dev.free_to_games;
```

You should see all the games on the table.

<a href="/assets/free-to-play/plsql_06.png" target="_blank"><img src="/assets/free-to-play/plsql_06.png"/></a>

## Creating the APEX app

Now log in to your APEX workspace. From the App Builder, click on "Create."

<a href="/assets/free-to-play/apex_01.png" target="_blank"><img src="/assets/free-to-play/apex_01.png"/></a>

Then, click on "New Application."

<a href="/assets/free-to-play/apex_02.png" target="_blank"><img src="/assets/free-to-play/apex_02.png"/></a>

On the New Application settings, enter a title. For example, "Free to Play Games." Then, click on "Create Application."

<a href="/assets/free-to-play/apex_03.png" target="_blank"><img src="/assets/free-to-play/apex_03.png"/></a> 

To keep things simple, we are going to use the Home page for the application. Click on the Home page to open the "Page Designer."

<a href="/assets/free-to-play/apex_04.png" target="_blank"><img src="/assets/free-to-play/apex_04.png"/></a>

Click on the new Breadcrumb Bar Region and give it a name. I named it "Best Free to Play Games!."

<a href="/assets/free-to-play/apex_05.png" target="_blank"><img src="/assets/free-to-play/apex_05.png"/></a>

Now, we are going to create a Region for the page body. Right-click on the Content Body section, and then on "Create Region."

<a href="/assets/free-to-play/apex_06.png" target="_blank"><img src="/assets/free-to-play/apex_06.png"/></a>

With the new Content Body Region selected, let us set some basic settings. Under the Identification section, name your Region "Games List", and set the Type to "Cards." Under Source, set the Location to "Local Database." And under Table Name, select the "FREE_TO_GAMES" table that we created earlier.

<a href="/assets/free-to-play/apex_07.png" target="_blank"><img src="/assets/free-to-play/apex_07.png"/></a>

Now, switch the Type from "Local Database" to "SQL Query." Doing so will auto-generate a SQL query with all the columns from your table (a lot faster than typing it from scratch). We want to use SQL query because later on, we will add a new column to spice up the UI.

After setting all this up, click on "Attributes."

<a href="/assets/free-to-play/apex_08.png" target="_blank"><img src="/assets/free-to-play/apex_08.png"/></a>

Under Attributes, configure the Card Region:
1. Set the Primary Key to the table's `ID` column.
2. Set the Title to the `TITLE` column.
3. Set the Subtitle to the `PUBLISHER`.
4. Set the Body to the `SHORT_DESCRIPTION`.

<a href="/assets/free-to-play/apex_09.png" target="_blank"><img src="/assets/free-to-play/apex_09.png"/></a>

We are ready to test our page. Click on the "Run" button to preview your work!

<a href="/assets/free-to-play/apex_10.png" target="_blank"><img src="/assets/free-to-play/apex_10.png"/></a>

## Final Details

To wrap up the page, we will add a color badge to display the game's genre. Also, we will add the game's thumbnail image to the card. And finally, we will add a link to play the game.

To create the color badge, we need a new column that will hold the CSS class. I will show you a simple way to accomplish this. Oracle provides the <a href="https://docs.oracle.com/en/database/oracle/oracle-database/21/sqlrf/ORA_HASH.html#GUID-0349AFF5-0268-43CE-8118-4F96D752FDE6" targer="_blank">ora_hash()</a> function that takes a string and returns a number. With this method, you do not have to worry about assigning colors to each genre.

If you look at the APEX Universal Theme <a href="https://apex.oracle.com/pls/apex/apex_pm/r/ut/color-and-status-modifiers" target="_blank">Color Modifiers</a>, you will see a total of 45 color classes that you can use. We can then build a color class based on the genre like so:

```sql
    ...
   'u-color-'||ora_hash(GENRE,45) as "GENRE_COLOR",    
    ...
```

Add the new column to your SQL query.

<a href="/assets/free-to-play/apex_11.png" target="_blank"><img src="/assets/free-to-play/apex_11.png"/></a>

Go back to the Region Attributes, and under Icon and Badge, set the Badge Column to `GENRE`, and the BADGE CSS Classes to `&GENRE_COLOR.`.

Note that `GENRE_COLOR` starts with '&' and ends in '.' to denote the value of a "Column Region."

<a href="/assets/free-to-play/apex_12.png" target="_blank"><img src="/assets/free-to-play/apex_12.png"/></a>

Re-run the page, and you should see that each card now has a badge displaying the genre, and each genre has a unique color.

<a href="/assets/free-to-play/apex_13.png" target="_blank"><img src="/assets/free-to-play/apex_13.png"/></a>

Now let us set up the game's thumbnail image. Go back to the Region Attributes, and under Media, set up the Source as "Image URL," the URL as the `&THUMBNAIL.` column region, the Position as "Body," the Appearance as "Widescreen," and the "Sizing" as "Cover."

<a href="/assets/free-to-play/apex_14.png" target="_blank"><img src="/assets/free-to-play/apex_14.png"/></a>

Finally, set up a link on the card to go to the game's page when the users click on it. Right-click on the Actions section under the Games List region and then click on "Create Action."

<a href="/assets/free-to-play/apex_15.png" target="_blank"><img src="/assets/free-to-play/apex_15.png"/></a>

On the new "Action" pane, set up the Identification as "Full Card." Now set up the Link as "Redirect to URL," and the Target as the `&GAME_URL` column region. Set up the Link Attributes to `target_"_blank"`, so the game's page opens in a new tab.

<a href="/assets/free-to-play/apex_16.png" target="_blank"><img src="/assets/free-to-play/apex_16.png"/></a>

Now run the page and try it out!

<a href="/assets/free-to-play/apex_16.png" target="_blank"><img src="/assets/free-to-play/apex_17.png"/></a>

## Congratulations

You completed the lesson. Let me know if you have questions or suggestions.