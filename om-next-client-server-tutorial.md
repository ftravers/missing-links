<div id="table-of-contents">
<h2>Table of Contents</h2>
<div id="text-table-of-contents">
<ul>
<li><a href="#sec-1">1. Overview</a></li>
<li><a href="#sec-2">2. Get the Code</a></li>
<li><a href="#sec-3">3. In the Beginning</a></li>
<li><a href="#sec-4">4. Remove State from Component</a></li>
<li><a href="#sec-5">5. Add Query, Parser, Reader</a></li>
<li><a href="#sec-6">6. A Parameterized Query</a></li>
<li><a href="#sec-7">7. Adding in a remote</a></li>
<li><a href="#sec-8">8. The Architecture</a></li>
<li><a href="#sec-9">9. Om Next Server Basics</a>
<ul>
<li><a href="#sec-9-1">9.1. Om-next Server Parts</a></li>
</ul>
</li>
<li><a href="#sec-10">10. Full example</a>
<ul>
<li><a href="#sec-10-1">10.1. Start the backend</a></li>
<li><a href="#sec-10-2">10.2. Start the frontend</a></li>
</ul>
</li>
<li><a href="#sec-11">11. Additional and More in Depth Information</a>
<ul>
<li><a href="#sec-11-1">11.1. Om Next Lifecycle Stages</a>
<ul>
<li><a href="#sec-11-1-1">11.1.1. Load Root Component</a></li>
<li><a href="#sec-11-1-2">11.1.2. lifecycle logged to console</a></li>
<li><a href="#sec-11-1-3">11.1.3. Adding in a fake remote</a></li>
<li><a href="#sec-11-1-4">11.1.4. A real remote</a></li>
</ul>
</li>
<li><a href="#sec-11-2">11.2. My choice of transport</a></li>
<li><a href="#sec-11-3">11.3. Om Next Backend</a>
<ul>
<li><a href="#sec-11-3-1">11.3.1. The transport to om-next server boundary</a></li>
<li><a href="#sec-11-3-2">11.3.2. Backend Parser</a></li>
</ul>
</li>
</ul>
</li>
</ul>
</div>
</div>


# Overview<a id="sec-1" name="sec-1"></a>

This tutorial will try to explain how to make the simplest
client/server app using om-next and a simple backend.

We'll start the tutorial with building a simple login page.  The user
is prompted for a username and a password, and when they supply the
correct one, as verified by the backend database, the login form is
replaced by a success message.

We'll break down each step of this simple om-next app as it proceeds.

# Get the Code<a id="sec-2" name="sec-2"></a>

The code is contained in two git repositories:

The client code: 
<https://github.com/ftravers/omn1> - "Om Next, 1st tutorial"

The server code: <https://github.com/ftravers/omn1be> "Om Next Back End,
1st tutorial"

Throughout the tutorial we'll reference branches in these
repositories.  Check that branch out to follow along.

# In the Beginning<a id="sec-3" name="sec-3"></a>

*Git Repository*: `omn1`

*Git Branch*: `in-the-beginning`

Here is a super simple om-next web-app.  All it does is print "Hello
World".

```clojure
    (ns omn1.webpage
      (:require
       [om.next :as om :refer-macros [defui]]
       [om.dom :as dom :refer [div]]
       [goog.dom :as gdom]))
    
    (defui SimpleUI
      Object
      (render
       [this]
       (div nil "Hello World")))
    
    (om/add-root!
     (om/reconciler {})
     SimpleUI
     (gdom/getElement "app"))

```

The line:

```clojure
    (om/reconciler {})

```

is a bit redundant in this example, but its a required argument to the
`add-root!` function, so we include it.  It doesn't do anything at
this point, but later on we'll see what it does.

We refer to the user interface created by the `defui` macro as a
*Component*.

We've hard-coded "Hello World" into the UI component, which is bad
form, lets extract it now.

# Remove State from Component<a id="sec-4" name="sec-4"></a>

*Git Branch*: `remove-state`

Now we move the data from being hard coded inside the component to an
external light weight database.  Parts that are redundant from
previous examples are elided.

```clojure
    ;; ...
    
    (defui SimpleUI
      Object
      (render
       [this]
       (div nil (:greeting (om/props this)))))
    
    (def app-state
      (atom {:greeting "Hello World"}))
    
    (def reconciler
      (om/reconciler
       {:state app-state}))
    
    ;; ...

```

# Add Query, Parser, Reader<a id="sec-5" name="sec-5"></a>

*Git Branch*: `add-reader-query-parser`

```clojure
    ;; ...
    (defui SimpleUI
      static om/IQuery
      (query [_] [:greeting])
    
      ;; ...
      )
    
    ;; ...
    
    (defn my-reader
      [env kee parms]
      (.log js/console (:target env))
      {:value "abc"})
    
    (def parser
      (om/parser {:read my-reader}))
    
    (def reconciler
      (om/reconciler
       {:state app-state
        :parser parser}))
    
    ;; ...

```

Now the application looks quite a bit more complicated.  We've added a
query to the component, a reader function and a parser.

Run the program and inspect the console.  The code:

```clojure
    (.log js/console (:target env))

```

causes the following output:

```clojure
    null
    :remote

```

Om-next will run the reader function once for a local query, and once
for any remotes that are defined.  We haven't define any remote end
points, but om-next out of the box provides one remote called:
`:remote`.  A remote is a mechanism to wire in calls to a backend
server. 

Our reader function `my-reader`, has the function parameter `kee`, set
to the keyword `:greeting`.  Then the reader result is a map with a
key `:value` set to the string `abc`.

Reader functions should always return a map with a `:value` key, that
is set to whatever the value for the passed in `kee` is.

As you can see `{:greeting "abc"}` gets printed out on the webpage.

So we have a lot of ceremony already, and it is a bit hard to percieve
the benefits of this approach at this point.  Unfortunately, we'll
just need to chug through this and hopefully in the end you can start
to appreciate the benefits.

# A Parameterized Query<a id="sec-6" name="sec-6"></a>

Our eventual goal is to create a login page that passes a username and
password to a backend database, and if the username/password pair
matches what is in the database, then we display a "login successful"
page. 

Our query is going to be: `:user/authenticated`.  This value will
initially be `false`, but eventually, when the correct
username/password pair is supplied, be changed to be `true`.

*Git Branch*: `parameterize-query`

```clojure
     1    (defui SimpleUI
     2      static om/IQuery
     3      (query [_]
     4             '[(:user/authenticated
     5                {:user/name ?name
     6                 :user/password ?pword})])
     7  
     8      static om/IQueryParams
     9      (params [this]
    10              {:name "" :pword ""})
    11      ;; ...
    12      )
    13  
    14  (defn my-reader
    15    [env kee parms]
    16    (.log js/console parms)
    17    ;; ...
    18    )

```

The `IQueryParams` indicate which parameters are available to this
component and query.  Our `IQuery` section has been updated to make
use of these parameters.

Line abc We are dumping the `parms` parameter of the reader
function to the console.  Go inspect the console to see the shape of
the data.

# Adding in a remote<a id="sec-7" name="sec-7"></a>

*Git Branch*: `add-remote`

Now we have added a function that is stubbing out what will eventually
be an actual call to a remote server.  Our `remote-connection`
function responds with the key `:user/authenticate` to `true`.

When we want to trigger a remote read we add a key to the map returned
by a reader that is the name of the remote, in our case the default
om-next remote `:remote`, and set it's value to be true.

We also wire it up in the reconciler by passing the function to the
`:send` keyword of the reconciler constructor map.

Finally lets hardcode in a username password pair.  If you look at the
console of the browser then, you'll see the following data spit out:

```clojure
    [(:user/authenticated
      {:user/name "fenton"
       :user/password "passwErd"})]

```

So this is the data that our client will send to our server.  This is
EDN.  

# The Architecture<a id="sec-8" name="sec-8"></a>

Om-next has nothing to say about how you would communicate with a
backend server.  So you can use any of the methods available to a
browser to do this.  Some examples of technologies you could use:
http, REST, json, websockets, EDN, transit, blah, blah, blah.

The key to understand is that the client has a piece of Clojure EDN
data that it will give to you, and you have to send that back to the
server somehow.  This example happens to use EDN over websockets.
Transit with REST might be another good way.

In our example we are using this data:

```clojure
    [(:user/authenticated
      {:user/name "fenton"
       :user/password "passwErd"})]

```

Please keep this front and center in your mind.  Any good integration
is going to be all about data and only data.  Here we have a classic
piece of Clojure EDN.  In classic clojure style, data is KING!

Once the data is received by your tech stack on the server side, you
pump it through om-next server.  In our example we make use of a
reader function and the om-next parser to handle this data from the
client.  In a full example you'd also have mutators too most likely.

So lets switch gears and head over and build up an om-next server.

# Om Next Server Basics<a id="sec-9" name="sec-9"></a>

So continuing on with our example, by some mechanism, the piece of
data:

```clojure
    [(:user/authenticated
      {:user/name "fenton"
       :user/password "passwErd"})]

```

is going to arrive.  We will fill in the plumbing between the client
and server later.  Remember that is not the focus of this tutorial, so
it will not be explored in detail.

## Om-next Server Parts<a id="sec-9-1" name="sec-9-1"></a>

In om-next, it is the job of the *Parser*, to figure out what to do
with both queries and mutations.  Checkout the following github
project if you haven't already done so:

Github Project: <https://github.com/ftravers/omn1be>

*Git Branch*: `step1-backend`

Checkout the project and branch and launch your REPL.

Now try some tests in the REPL:

```clojure
    omn1be.core> (parser {:state users}
                         '[(:user/authenticated
                            {:user/name "fenton"
                             :user/password "passwerd"})])
    #:user{:authenticated false}
    
    omn1be.core> (parser {:state users}
                         '[(:user/authenticated
                            {:user/name "fenton"
                             :user/password "passwErd"})])
    #:user{:authenticated true}

```

Lets quickly look at our reader function, even though it doesn't
present any new ideas.  The input params are the same as on the
client, and just like the client we simply return a map with the
answer attached to the `:value` key.

```clojure
    (defn reader
      [env kee params]
      (let [userz (:state env)
            username (:user/name params)
            password (:user/password params)]
        {:value (valid-user userz username password)}))

```

And our parser is dead simple:

```clojure
    (def parser (om/parser {:read reader}))

```

Thats all there is to a basic om-next server.

# Full example<a id="sec-10" name="sec-10"></a>

For the full working sample checkout the master branches of the two
projects, `omn1` and `omn1be`.

## Start the backend<a id="sec-10-1" name="sec-10-1"></a>

Start the backend at the command prompt:

```clojure
    cd omn1be; lein repl
    (load "websocket") 
    (in-ns 'omn1be.websocket)
    (start)
    (in-ns 'omn1be.router)

```

## Start the frontend<a id="sec-10-2" name="sec-10-2"></a>

```clojure
    cd omn1; lein figwheel

```

Navigate to:

<http://localhost:3449/>

Of course you'll need to have datomic installed for this complete
example to work.

# Additional and More in Depth Information<a id="sec-11" name="sec-11"></a>

## Om Next Lifecycle Stages<a id="sec-11-1" name="sec-11-1"></a>

Our code has one root UI component.  This component has a query for
one field, `:user/authenticated`.  The query for this field accept two
parameters, `:user/name` and `:user/password`.

The basic idea is that we send this query for the
`:user/authenticated` value, passing along the username and password
of the user.  This gets looked up in the database and if the pair is
valid, then `:user/authenticated` gets set to the value `true`
otherwise it is set `false`.

### Load Root Component<a id="sec-11-1-1" name="sec-11-1-1"></a>

The first stage to an om next application is to load the Root
component.  This is dictated by the following line:

```clojure
    (om/add-root! reconciler Login (gdom/getElement "app"))

```

Here the second param, root-class, is set to the `Login` component.
The third param, `target`, is the div in the `index.html` where to
mount or locate this component.  Finally the first argument is the
reconciler to use for this application.  The reconciler hold together
all the function and state required to handle data flows in the
application. 

1.  Our Query

    Our root component, `Login`, has a query of the form:
    
    ```clojure
        static om/IQuery
        (query
         [_]
         '[(:user/authenticated
            {:user/name ?name
             :user/password ?password})])
    
    ```
    
    Basically this says, get the value of `:user/authenticated` supplying
    as parameters to the query the values for the `:user/name` and
    `:user/password` fields.

2.  Query Parameters

    `?name` and `?password` are query parameter variables that hold the
    values for the username and password that this query will eventually
    use in its query for `:user/authenticated`.  We initially set their
    value to be the empty string:
    
    ```clojure
        static om/IQueryParams
        (params [this]
                {:name "" :password ""})
    
    ```

3.  Component State

    In react we can have local state variables.  The code:
    
    ```clojure
        (initLocalState
         [this]
         {:username "fenton"
          :password "passwErd"})
    
    ```
    
    creates two parameters: `:username:` and `:password` and sets their
    initial values.
    
    In the `:onChange` handlers for our two input elements we set the
    values of these two react state variables to be whatever the user
    types into the name and password input boxes.
    
    ```clojure
        (input
         #js
         {:name "uname"
          :type "text"
          :placeholder "Enter Username"
          :required true :value username
          :onChange
          (fn [ev]
            (let [value (.. ev -target -value)]
              (om/update-state! this assoc :username value)))})
    
    ```

4.  Submitting username/password to backend

    Finally when the user clicks the submit button to send the username
    and password to the backend we take the values from the react
    component state, and use those values to update the values of the
    query parameters.  Updating a query's parameter values causes the
    query to be rerun.
    
    Next we'll see how this state all runs by logging out to the console
    each time the reader is run.  The reader is the function that is run
    to handle processing the queries.

### lifecycle logged to console<a id="sec-11-1-2" name="sec-11-1-2"></a>

We can see everytime a query is run by putting a log statement into
our reader function.

```clojure
    (defmethod reader :default
      [{st :state :as env} key _]
      (log "default reader" key "env:target" (:target env))
      {:value (key (om/db->tree [key] @st @st))
       ;; :remote true
       :remote false
       })

```

Here we see a log statement at the top of the reader function.  Lets
see what a dump of the browser console looks like and try to
understand it.

```clojure
    1  [default reader]: :user/authenticated env:target null
    2  [props]: {:user/authenticated false}
    3  [default reader]: :user/authenticated env:target :remote

```

In line nil: the query of the component is run before the
component is first loaded.

In line 2: as the component is rendered we dump the react
properties that have been passed into the component, in this case it
is simply the `@app-state`.

This is done with line:

```clojure
    (log "props" (om/props this))

```

In the component rendering.

The line: 3, comes again from our `:default` reader, but this
time it is passed for the remote called `:remote`.  By default out of
the box in om-next we get a remote named `:remote`.  So the reader
will get called once for a local call, and once for each remote we
have defined.

So we have traced a basic flow of a simple component.  Now lets see
how to trigger a remote read.  When our reader is getting called with
the `:target` a remote, if we then also return `:remote true` in our
returned map from the reader, then our remote functions will also be
called. 

### Adding in a fake remote<a id="sec-11-1-3" name="sec-11-1-3"></a>

Git Repository: <https://github.com/ftravers/omn1>

*Git Branch*: `simple-remote`

So we want to send our stuff to a backend server.  Om next creates a
default hook for this.  So basically what happens again, is that our
reader will get called twice, once for trying to satisfy our query
from our local state, and once for trying to get the information from
the backend.

If we return `:remote true` in our reader response map, the remote
hooks will get triggered.  So lets see this in action.  First lets
wire up some basic 'remotes'.

First we must write a function that will be our remote query hook:

```clojure
    (defn my-remoter
      [qry cb]
      (log "remote query" (str qry))
      (cb {:some-param "some value"}))

```

And lets wire this into the reconciler.

```clojure
    (def reconciler
      (om/reconciler
       {:state app-state
        :parser parser
        :send my-remoter}))

```

And finally our reader needs to return `:remote true` for the remote
to run:

```clojure
    (defmethod reader :default
      [{st :state :as env} key _]
      (log "default reader" key "env:target" (:target env))
      {:value (key (om/db->tree [key] @st @st))
       :remote true})

```

Now lets see what happens as we trace the programs execution with some
logging statements

```clojure
    1  [default reader]: :some-param env:target null
    2  [props]: {:some-param "not much"}meta
    3  [default reader]: :some-param env:target :remote
    4  [remote query]: {:remote [:some-param]}
    5  [app state]: {:some-param "not much"}
    6  [default reader]: :some-param env:target null
    7  [props]: {:some-param "value gotten from remote!"}meta
    8  [app state]: {:some-param "value gotten from remote!"}
    9  [default reader]: :some-param env:target null

```

The first three lines remain unchanged.

Line 4: we see we've entered into the hook for the
remote function.  We dump the `@app-state`

Line 5: before we call the callback, `cb`, with our
new data, which should merge the data into our `@app-state` map.  The
callback is called and we can see that the `@app-state` is updated and
the component is re-rendered.

I'm not quite sure why the reader is called at the end&#x2026;but maybe
someone who knows om-next better can explain that.

### A real remote<a id="sec-11-1-4" name="sec-11-1-4"></a>

At this point we aren't hooking into any backend, we are just stubbing
out the call to the backend.  To have a real call to a backend
involves taking our request and sending via `http`, `json`,
`websockets`, `edn`, or some other way to our backend.  Receiving the
data, doing something with it and creating a response and sending it
back, then getting it back on the client, and updating the local
client data and therefore updating the client webpage.

So that is a lot of stuff.  Don't dispair, I will demonstrate real
code that does this, but the scope of this tutorial is to demonstrate
how to use om-next with a remote.  How exactly data is exchanged with
a remote is actually a separate concern.  This is actually a wonderful
thing.  As clojuristas we dont like monolithic frameworks that package
the entire world into an opinionated whole.  Perhaps like a rails
project.  We would rather pick the pieces that best suit our needs,
and data transport between client and server is not something that om
next has an opinion on and it lets you fill in that blank however you
would like.

What we need to be clear on is the boundaries between the transport
segment and om next.  So lets reiterate that now to be absolutely
clear.

This boundary or responsibility handoff occurs in our `my-remoter`
function.  Om next hands us the data of the query that we've put into
the `qry` parameter, then it expects us to call the callback, `cb`,
with the results of our remote query.  We'll look into detail of what
the shape of the data is that om next expects us to return the result
in.

Here is a sample of data in and data out that om next would be happy
with:

IN:

```clojure
    [:some-param]

```

OUT:

```clojure
    {:some-param "Some New Value"}

```

## My choice of transport<a id="sec-11-2" name="sec-11-2"></a>

I have written simple websocket client and server libraries that I
use.  They are located at:

<https://github.com/ftravers/websocket-client>

and

<https://github.com/ftravers/websocket-server>

I have chosen to send EDN over this websocket connection.

Another perhaps better choice would be to send JSON over Transit.
Perhaps using a Ring server or some other type of web server.  My
websocket server uses http-kit to act as the websocket server.

Again, what you use is really beyond the scope of this tutorial, and I
dont want this tutorial to get bogged down in those details, since it
would detract from this tutorials purpose which is solely to educate a
user on how to create a typical client server app using om-next.

Truely this tutorial is about how to use om-next in a client/server
setup, somewhat agnostic to whatever the backend database of choice
is.

So with those caveats declared lets look into what an om-next backend
might look like.

## Om Next Backend<a id="sec-11-3" name="sec-11-3"></a>

The project for the om next backend is a git project located here, go
ahead and clone it:

<https://github.com/ftravers/omn1be>

The project name, omn1be, is the abbreviation of Om Next version 1
Back End.

In our example we are asking if a user has supplied the correct
username password combination, and if so, to set the flag
`:user/authenticated` to `true`, otherwise set it to `false`.

Our complete example contains more pieces than what this tutorial is
aiming to teach about.  Here is a word diagram about the flow and
architecture of the system:

```clojure
    [(:user/authenticated
      {:user/name "fenton"
       :user/password "passwErd"})]

```

Again here we need to be clear of where the handoff occurs from the
choice of wire or transport architecture occurs and where we enter the
land of om-next for the backend.  Lets inspect the file layout for the
project first:

```clojure
    ╭─fenton@ss9 ~/projects ‹system› ‹master*› 
    ╰─➤  cd omn1be
    ╭─fenton@ss9 ~/projects/omn1be ‹system› ‹upper-case› 
    ╰─➤  tree src
    src
    `-- omn1be
        |-- core.clj
        |-- router.clj
        `-- websocket.clj

```

The `core.clj` file has all the information about the datomic
database.  It has the schema, the testdata, etc.  If you need more
help understanding how datomic works, please checkout my tutorial at: 

[Beginner Datomic Tutorial](https://github.com/ftravers/missing-links/blob/master/datomic-tutorial.md)

Again, I will highlight the boundaries of the durability layer
(i.e. the database), and om-next server side.

The file: `websocket.clj`, is the servers side of the transport
layer.  Again you could sub this out with whatever type of transport
you wanted to do.

Finally, the file: `router.clj` is truely the om-next server side.  If
you want to do om-next on the server side then this file will be the
most interesting for you.

### The transport to om-next server boundary<a id="sec-11-3-1" name="sec-11-3-1"></a>

Lets point out where the boundary of the server end of the transport
layer to the om-next server is.

Have a look at the

*Git Branch*: `full-working-basic-backend`

To fire up the backend you could do:

```clojure
    $ cd omn1be; lein repl
    (load "websocket") 
    (in-ns 'omn1be.websocket)
    (start)
    (in-ns 'omn1be.router)

```

Then to test it without our front end, we could use the "Simple
Websocket Client" chrome extension.

The websocket URL end point is: `ws://localhost:7890`

Then we can send the following data in it:

```clojure
    [(:user/authenticated
      {:user/name "fenton"
       :user/password "passwErd"})]

```

Here is a log of some sent requests and their response from the
server:

```clojure
    [(:user/authenticated
      {:user/name "fenton"
       :user/password "passwErd"})]
    {:user/authenticated true}
    
    [(:user/authenticated
      {:user/name "fenton"
       :user/password "password"})]
    {:user/authenticated false}

```

### Backend Parser<a id="sec-11-3-2" name="sec-11-3-2"></a>

So we can see that all we are sending over the wire is an om next
parameterized query.  

```clojure
    [(:user/authenticated
      {:user/name "fenton"
       :user/password "passwErd"})]

```

A good reference for the different types of queries can be found at:
[Query Syntax Explained](https://anmonteiro.com/2016/01/om-next-query-syntax/).

If we create a server side reader and parser, we can pass this query
to it and it will act almost the same as the front end.

When we develop an om next backend there is a symmetry to the front
end.  Again we will create a reader function and create a parser with
this reader function.  So we pass from the transport layer, into the
om-next server layer in this code:

```clojure
    (defn process-data [data]
      (->> data
           read-string
           (router/parser {:database (be/db)})
           prn-str))

```

Particularly when we call the `parser` with the data we recieved.  The
result of calling the parser is passed back into the transport layer.
