<div id="table-of-contents">
<h2>Table of Contents</h2>
<div id="text-table-of-contents">
<ul>
<li><a href="#sec-1">1. About These Tutorials</a></li>
<li><a href="#sec-2">2. Overview</a></li>
<li><a href="#sec-3">3. Daily Setup</a>
<ul>
<li><a href="#sec-3-1">3.1. Start the backend</a></li>
<li><a href="#sec-3-2">3.2. Start frontend</a></li>
<li><a href="#sec-3-3">3.3. Testing from front end</a></li>
<li><a href="#sec-3-4">3.4. Testing from backend</a></li>
</ul>
</li>
<li><a href="#sec-4">4. Om Next lifecycle stages</a>
<ul>
<li><a href="#sec-4-1">4.1. Load Root Component</a>
<ul>
<li><a href="#sec-4-1-1">4.1.1. Our Query</a></li>
<li><a href="#sec-4-1-2">4.1.2. Query Parameters</a></li>
<li><a href="#sec-4-1-3">4.1.3. Component State</a></li>
<li><a href="#sec-4-1-4">4.1.4. Submitting username/password to backend</a></li>
</ul>
</li>
<li><a href="#sec-4-2">4.2. lifecycle logged to console</a></li>
<li><a href="#sec-4-3">4.3. Adding in a fake remote</a></li>
<li><a href="#sec-4-4">4.4. A real remote</a></li>
</ul>
</li>
<li><a href="#sec-5">5. My choice of transport</a></li>
<li><a href="#sec-6">6. Om Next Backend</a>
<ul>
<li><a href="#sec-6-1">6.1. The transport to om-next server boundary</a></li>
<li><a href="#sec-6-2">6.2. Backend Parser</a></li>
</ul>
</li>
<li><a href="#sec-7">7. Send username &amp; password</a></li>
<li><a href="#sec-8">8. Datomic - the backend</a>
<ul>
<li><a href="#sec-8-1">8.1. Data Structure</a></li>
</ul>
</li>
<li><a href="#sec-9">9. Front End (om-next)</a></li>
<li><a href="#sec-10">10. Om-Next Remotes</a></li>
<li><a href="#sec-11">11. Database Structure</a></li>
</ul>
</div>
</div>


# About These Tutorials<a id="sec-1" name="sec-1"></a>

These tutorials are aimed at a beginner audience.  They try to be
brief yet complete.  Less can be more.  However I try not to leave any
detail out either.  Gaps in explanation can lead to confusion.

In summary I hope for these tutorials to be short, complete and
approachable for beginners.  I appreciate constructive feedback, so
let me know if, in your opinion, I fail or succeed to achieve these
goals.

# Overview<a id="sec-2" name="sec-2"></a>

This tutorial will try to explain how to make the simplest
client/server app using om-next and datomic respectively.

If you are not familiar with Datomic, I suggest you checkout [my
 tutorial](https://www.reddit.com/r/Clojure/comments/5zu1oc/my_datomic_tutorial_feedback_sought/) on it.

We'll start the tutorial with building a simple login page.  The user
is prompted for a username and a password, and when they supply the
correct one, as verified by the backend database, the login form is
replaced by a success message.

# Daily Setup<a id="sec-3" name="sec-3"></a>

## Start the backend<a id="sec-3-1" name="sec-3-1"></a>

Start the backend outside of emacs at the command prompt:

    cd ~/projects/omn1be; lein repl
    
    (load "websocket") (in-ns 'omn1be.websocket) (start) (in-ns 'omn1be.router)

## Start frontend<a id="sec-3-2" name="sec-3-2"></a>

Open project `omn1` and file `src/omn1/webpage.cljs`.  

Start a REPL there.  Then run figwheel.

Open your browser at: <http://localhost:3449/>

## Testing from front end<a id="sec-3-3" name="sec-3-3"></a>

    omn1.webpage>   @app-state
    {:authenticated false}
    
    ;; a mutation
    omn1.webpage> (om/transact! reconciler `[(user/login {:user/name "fenton" :user/password "passwErd"})])
    omn1.webpage> @app-state
    {:authenticated false, :user/name "fenton", :user/password "welcome1"}
    
    ;; a parameterized query
    omn1.webpage> (om/transact! reconciler '[(:user/authenticated {:user/name "fenton" :user/password "passwErd"})])

## Testing from backend<a id="sec-3-4" name="sec-3-4"></a>

    omn1be.router=> (parser {:database (core/db)} '[(user/login {:user/name "fenton", :user/password "passwErd"})])
    #:user{login {:keys (:user/name :user/password), :valid-user true}}
    omn1be.router=> (api {:function :user/login :params {:user/name "fenton", :user/password "passwErd"}})
    {:valid-user true}

# Om Next lifecycle stages<a id="sec-4" name="sec-4"></a>

Our code has one root UI component.  This component has a query for
one field, `:user/authenticated`.  The query for this field accept two
parameters, `:user/name` and `:user/password`.

The basic idea is that we send this query for the
`:user/authenticated` value, passing along the username and password
of the user.  This gets looked up in the database and if the pair is
valid, then `:user/authenticated` gets set to the value `true`
otherwise it is set `false`.

## Load Root Component<a id="sec-4-1" name="sec-4-1"></a>

The first stage to an om next application is to load the Root
component.  This is dictated by the following line:

    (om/add-root! reconciler Login (gdom/getElement "app"))

Here the second param, root-class, is set to the `Login` component.
The third param, `target`, is the div in the `index.html` where to
mount or locate this component.  Finally the first argument is the
reconciler to use for this application.  The reconciler hold together
all the function and state required to handle data flows in the
application. 

### Our Query<a id="sec-4-1-1" name="sec-4-1-1"></a>

Our root component, `Login`, has a query of the form:

    static om/IQuery
    (query  [_] '[(:user/authenticated {:user/name ?name :user/password ?password})])

Basically this says, get the value of `:user/authenticated` supplying
as parameters to the query the values for the `:user/name` and
`:user/password` fields.

### Query Parameters<a id="sec-4-1-2" name="sec-4-1-2"></a>

`?name` and `?password` are query parameter variables that hold the
values for the username and password that this query will eventually
use in its query for `:user/authenticated`.  We initially set their
value to be the empty string:

    static om/IQueryParams
    (params [this]
            {:name "" :password ""})

### Component State<a id="sec-4-1-3" name="sec-4-1-3"></a>

In react we can have local state variables.  The code:

    (initLocalState [this] {:username "fenton" :password "passwErd"})

creates two parameters: `:username:` and `:password` and sets their
initial values.

In the `:onChange` handlers for our two input elements we set the
values of these two react state variables to be whatever the user
types into the name and password input boxes.

    (input
     #js
     {:name "uname"
      :type "text"
      :placeholder "Enter Username"
      :required true :value username
      :onChange (fn [ev]
                  (let [value (.. ev -target -value)]
                    (om/update-state! this assoc :username value)))})

### Submitting username/password to backend<a id="sec-4-1-4" name="sec-4-1-4"></a>

Finally when the user clicks the submit button to send the username
and password to the backend we take the values from the react
component state, and use those values to update the values of the
query parameters.  Updating a query's parameter values causes the
query to be rerun.

Next we'll see how this state all runs by logging out to the console
each time the reader is run.  The reader is the function that is run
to handle processing the queries.

## lifecycle logged to console<a id="sec-4-2" name="sec-4-2"></a>

We can see everytime a query is run by putting a log statement into
our reader function.

    (defmethod reader :default
      [{st :state :as env} key _]
      (log "default reader" key "env:target" (:target env))
      {:value (key (om/db->tree [key] @st @st))
       ;; :remote true
       :remote false
       })

Here we see a log statement at the top of the reader function.  Lets
see what a dump of the browser console looks like and try to
understand it.

    1  [default reader]: :user/authenticated env:target null
    2  [props]: {:user/authenticated false}
    3  [default reader]: :user/authenticated env:target :remote

In line: nil the query of the component is run before the
component is first loaded.

In line: 2, as the component is rendered we dump the react
properties that have been passed into the component, in this case it
is simply the `@app-state`.

This is done with line:

    (log "props" (om/props this))

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

## Adding in a fake remote<a id="sec-4-3" name="sec-4-3"></a>

GIT REPO: <https://github.com/ftravers/omn1>
BRANCH: simple-remote

So we want to send our stuff to a backend server.  Om next creates a
default hook for this.  So basically what happens again, is that our
reader will get called twice, once for trying to satisfy our query
from our local state, and once for trying to get the information from
the backend.

If we return `:remote true` in our reader response map, the remote
hooks will get triggered.  So lets see this in action.  First lets
wire up some basic 'remotes'.

First we must write a function that will be our remote query hook:

    (defn my-remoter
      [qry cb]
      (log "remote query" (str qry))
      (cb {:some-param "some value"}))

And lets wire this into the reconciler.

    (def reconciler
      (om/reconciler
       {:state app-state
        :parser parser
        :send my-remoter}))

And finally our reader needs to return `:remote true` for the remote
to run:

    (defmethod reader :default
      [{st :state :as env} key _]
      (log "default reader" key "env:target" (:target env))
      {:value (key (om/db->tree [key] @st @st))
       :remote true})

Now lets see what happens as we trace the programs execution with some
logging statements

    1  [default reader]: :some-param env:target null
    2  [props]: {:some-param "not much"}meta
    3  [default reader]: :some-param env:target :remote
    4  [remote query]: {:remote [:some-param]}
    5  [app state]: {:some-param "not much"}
    6  [default reader]: :some-param env:target null
    7  [props]: {:some-param "value gotten from remote!"}meta
    8  [app state]: {:some-param "value gotten from remote!"}
    9  [default reader]: :some-param env:target null

The first three lines remain unchanged.

On line: 4, we see we've entered into the hook for the
remote function.  We dump the `@app-state` on line:
5, before we call the callback, `cb`, with our
new data, which should merge the data into our `@app-state` map.  The
callback is called and we can see that the `@app-state` is updated and
the component is re-rendered.

I'm not quite sure why the reader is called at the end&#x2026;but maybe
someone who knows om-next better can explain that.

## A real remote<a id="sec-4-4" name="sec-4-4"></a>

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

    [:some-param]

OUT:

    {:some-param "Some New Value"}

# My choice of transport<a id="sec-5" name="sec-5"></a>

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
user on how to create a semi-typical client server app using om-next
and datomic.

Even the use of datomic should be minimized.  Truely this tutorial is
about how to use om-next in a client server setup, somewhat agnostic
to whatever the backend database of choice is.

So with those caveats declared lets look into what an om-next backend
might look like.

# Om Next Backend<a id="sec-6" name="sec-6"></a>

Clone the git project:

<https://github.com/ftravers/omn1be>

The project name, omn1be, is the abbreviation of Om Next version 1
Back End.

Again here we need to be clear of where the handoff occurs from the
choice of wire or transport architecture occurs and where we enter the
land of om-next for the backend.  Lets inspect the file layout for the
project first:

    ╭─fenton@ss9 ~/projects ‹system› ‹master*› 
    ╰─➤  cd omn1be
    ╭─fenton@ss9 ~/projects/omn1be ‹system› ‹upper-case› 
    ╰─➤  tree src
    src
    `-- omn1be
        |-- core.clj
        |-- router.clj
        `-- websocket.clj

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

## The transport to om-next server boundary<a id="sec-6-1" name="sec-6-1"></a>

Lets point out where the boundary of the server end of the transport
layer to the om-next server is.

Have a look at the

GIT BRANCH: full-working-basic-backend

To fire up the backend you could do:

    $ cd omn1be; lein repl
    (load "websocket") (in-ns 'omn1be.websocket) (start) (in-ns 'omn1be.router)

Then to test it without our front end, we could use the "Simple
Websocket Client" chrome extension.

The websocket URL end point is: `ws://localhost:7890`

Then we can send the following data in it:

    [(:user/authenticated {:user/name "fenton" :user/password "passwErd"})]

Here is a log of some sent requests and their response from the
server:

    [(:user/authenticated {:user/name "fenton" :user/password "passwErd"})]
    {:user/authenticated true}
    [(:user/authenticated {:user/name "fenton" :user/password "password"})]
    {:user/authenticated false}

## Backend Parser<a id="sec-6-2" name="sec-6-2"></a>

So we can see that all we are sending over the wire is an om next
parameterized query.

    [(:user/authenticated {:user/name "fenton" :user/password "passwErd"})]

If we create a server side reader and parser, we can pass this query
to it and it will act almost the same as the front end.

When we develop an om next backend there is a symmetry to the front
end.  Again we will create a reader function 

# Send username & password<a id="sec-7" name="sec-7"></a>

So we can send something to the backend using:

    (om/transact!
     reconciler
     `[(user/login
        {:user/name "fenton"
         :user/password "passwErd"})])

# Datomic - the backend<a id="sec-8" name="sec-8"></a>

## Data Structure<a id="sec-8-1" name="sec-8-1"></a>

Imagine we have datomic data that looks like:

    {:db/id 1
     :car/make "Toyota"
     :car/model "Tacoma"
     :year 2013}
    
    {:db/id 2
     :car/make "BMW"
     :car/model "325xi"
     :year 2001}
    
    {:db/id 3 
     :user/user "ftravers"
     :user/age 54
     :user/cars [{:db/id 1}
                 {:db/id 2}]}

We can see the `:user/cars` field points to an array of cars that
fenton owns.

**NOTE:** I'm simplifying some aspects of this tutorial because I want
to keep focused on conceptual simplicity over absolute syntactical
correctness.

In datomic we can use **pull** syntax to indicate which data we want.  A
valid pull *query* would look like:

    [:user/name
     :user/age
     {:user/cars
      [:db/id :car/make :car/model :year]}]

which datomic will happily *hydrate*, or fill out the fields of this
*query* to become:

    #:user{:email "ftravers",
           :age 43,
           :cars
           [{:db/id 1,
             :car/make "Toyota",
             :car/model "Tacoma",
             :year 2013}
            {:db/id 2,
             :car/make "BMW",
             :car/model "325xi",
             :year 2001}]}

So we have (kind of) demonstrated how we can extract data from
datomic, lets see if we can get this to jive with the om-next front
end now.

# Front End (om-next)<a id="sec-9" name="sec-9"></a>

Now we need to structure the front end so we can easily get this info
from the backend.

So in om-next we'd normally construct this as two components.  One is
the parent component which would display the email and age of the
user, then we'd have a table that shows the cars they own.  The two
rows of the table are another component that is reused for each row.

Om-next components declare the data they require, in our example we'd
have something like:

    (defui Car
      (query [this] [:db/id :car/make :car/model :year]))
    
    (defui MyCars
      (query [this] [:user/name :user/age {:user/cars (om/get-query Car)}]))

Now om-next will create one combined query which we can pass along to
datomic to *hydrate*.

So the final full query looks like:

    [:user/name
     :user/age
     {:user/cars
      [:db/id :car/make :car/model :year]}]

Which is **identical** to the query we were able to pass to datomic, so
on the surface this looks great.  Next we'll look at how om-next
queries a backend and how it merges the results into the local data
structure.

# Om-Next Remotes<a id="sec-10" name="sec-10"></a>

Om-next remotes are handled in a function definition.  This function
can be passed anything that the reader has.  A good place to look at
what it is sent is this URL: [Alan Kay om-next tutorial - State Reads](https://awkay.github.io/om-tutorial/#!/om_tutorial.E_State_Reads_and_Parsing).
Its a long page without anchors so go to section: "Implementing Read",
then scroll down to the numbered list where he describes the three
parameters passed into `read`.

The last parameter your remote handling function will recieve is a
callback function that you should be called with the results of your
actual server side response.  Critical here is the shape of your local
state store, and the data that you are passing into the callback.

Lets look at the aspect of returning data from a remote function and
seeing how/what is able to be merged into your local state store.

# Database Structure<a id="sec-11" name="sec-11"></a>

A big part of om-next is that it creates norms about how to store your
application data.  Often we have pieces of data that appear in
multiple places in our UI.  Naturally, if it is the same data in
multiple places, we'd like to only have one real copy of it.  Any
other 'copies' are actually references.

To store data in om-next you follow this structure:

    (def app-state { :keyword { id real-information }})

or a real example:

    (def app-state {:curr-user { "ftravers" {:age 21 :height 183}}})

We can now add a second location that 'refers' to the first like so:

    (def app-state {:app-owner [:curr-user "ftravers"]
                    :curr-user { "ftravers" {:age 21 :height 183}}})

The format: `[:keyword id]` is called an `ident`.  Its a reference to
some shared data.
