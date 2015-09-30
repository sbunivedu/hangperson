Hangperson: a SaaS getting-started assignment
=============================================================

(Adapted from an [assignment](https://github.com/saasbook/hw-sinatra-saas-hangperson)
by Armando Fox and Nick Herson)

In this assignment you'll be introduced to part of the basic cycle of creating SaaS in
a disciplined way.

The full Agile/XP cycle we follow in ESaaS (the online course) includes talking to the
customer, using BDD to develop scenarios, turning those scenarios into
runnable integration/acceptance tests with Cucumber, using those
scenarios plus TDD to drive the creation of actual code, and deploying
the result of each iteration's work to the cloud.

In this introductory assignment, we've provided RSpec unit tests to let 
you use TDD to develop game logic
for the popular word-guessing game Hangman.  In the full Agile/XP cycle,
you'd develop these tests yourself as you code.

You'll then use the Rails framework to make the Hangman game available
as SaaS.  Adapting the game logic for SaaS will introduce you to
thinking about RESTful routes and service-oriented architecture.
As you develop the "SaaS-ified" Hangman game, you'll use Cucumber to
describe how gameplay will work from the player's point of view and as
"full stack" integration tests that will drive SaaS development.  In the
full Agile/XP cycle, you'd develop Cucumber scenarios yourself based on
consultation with the customer, and create the necessary *step
definitions* (Cucumber code
that turns plain-English scenarios into runnable tests).  In this
assignment we provide both the scenarios and step definitions for you.

You'll deploy your game to the cloud using Heroku, giving you experience
in automating SaaS deployment.

Prerequisites
-------------

You will need to be comfortable with the following concepts and skills:

* Basic Git usage and how to push your code to GitHub, as described in
Appendix A of *Engineering Software as a Service*.
 
* "Survival level" Unix command-line skills and facility with an editor
to edit code files.

* Basic concepts behind SaaS application architecture, as described in
Chapter 2 of *Engineering Software as a Service*.

Part I. Developing Hangperson Using TDD and Autotest
============================================

**Goals:** Use test-driven development (TDD) based on the tests we've
provided to develop the game logic for Hangman, which forces you to
think about what data is necessary to capture the game's state. This
will be important when you SaaS-ify the game in the next part.

**What you will do:**  Use `autotest`, our provided test cases will be
re-run each time you make a change to the app code.  One by one, the
tests will go from red (failing) to green (passing) as you create the
app code.  By the time you're done, you'll have a working Hangperson
game class, ready to be "wrapped" in SaaS using Rails.

Overview
--------

Our Web-based version of the popular game "hangman" works as follows:

* The computer picks a random word

* The player guesses letters in order to guess the word

* If the player guesses the word before making seven wrong guesses of
letters, they win; otherwise they lose.  (Guessing the same letter
repeatedly is simply ignored.)

To make the game fun to play, each time you start a new game the app
will actually retrieve a random English word from a remote server, so
every game will be different.  This feature will introduce you not only
to using an external service (the random-word generator) as a "building
block" in a **Service-Oriented Architecture**, but also how a Cucumber
scenario can test such an app deterministically with tests that
**break the dependency** on the external service at testing time.

* Clone this repo in your working environment.
* Change into the directory of the repo, say `autotest`.  

This will fire up the Autotest framework, which looks for various files
to figure out what kind of app you're testing and what test framework
you're using.  In our case, it will discover the file called `.rspec`,
which contains RSpec options and indicates we're using the RSpec testing
framework.  Autotest will therefore look for test files under `spec/` and the
corresponding class files in `lib/`.

We've provided a set of 19 test cases to help you develop the game class.
Take a look at `spec/hangperson_game_spec.rb`.  It specifies behaviors
that it expects from the class `lib/hangperson_game.rb`.  
initially, we have added `:pending => true` to every spec, so when
Autotest first runs these, you should see the test case names printed
in yellow, and the report "19 examples, 19 pending."

Now, with Autotest still running, delete `:pending => true` from line 11, and
save the file.  You should immediately see Autotest wake up and re-run
the tests.  You should now have 3 failures and 16 pending examples.

The `describe 'new'` block means "the following block of tests describe
the behavior of a 'new' HangpersonGame instance."  A new new instance of 
the game class is created, and the `its` blocks verify the
presence and values of instance variables.

* Self-check: According to our test cases, how many arguments does the
game class constructor expect, and therefore what will the first line of
the method definition look like that you must add to
`hangperson_game.rb`?

> One argument (in this example, "glorp"), and since constructors in
> Ruby are always named `initialize`, the first line will be
> `def initialize(new_word)` or something similar.

* Self-check: According to the tests in this `describe` block, what
instance variables is a HangpersonGame expected to have?

> @word, @guesses, and @wrong_guesses.

You'll need to create getters and setters for these.  Hint: use `attr_accessor`.
When you've done this successfully and saved `hangperson_game.rb`,
Autotest should wake up again and the three examples that were
previously failing should now be passing (green).

Continue in this manner, removing `:pending => true` from one example at
a time working your way down the specs, until you've implemented all the
instance methods of the game class: `guess`, which processes a guess and
modifies the instance variables `wrong_guesses` and `guesses`
accordingly; `check_win_or_lose`, which returns one of the symbols
`:win`, `:lose`, or `:play` depending on the current game state; and
`word_with_guesses`, which substitutes the correct guesses made so far
into the word.

### Debugging Tip

When running tests, you can insert the Ruby command
`debugger` into your app code to drop into the command-line debugger and
inspect variables and so on.  Type `h` for help at the debug prompt.
Type `c` to leave the debugger and continue running your code.

* Take a look at the code in the class method
`get_random_word`, which retrieves a random word from a Web service we found
that does just that.  Use the following command to verify that the Web
service actually works this way. Run it several times to verify that you
get different words.

`curl --data '' http://watchout4snakes.com/wo4snakes/Random/RandomWord`

(`--data` is necessary to force `curl`
to do a POST rather than a GET.  Normally the
argument to `--data` would be the encoded form fields, but in this case
no form fields are needed.)
Using `curl` is a great way to debug interactions with
external services.  `man curl` for (much) more detail on this powerful
command-line tool.


Part II. RESTful thinking for HangPerson
=================================

**Goals:**  Understand how to expose your app's behaviors as RESTful
actions in a SaaS environment, and how to preserve game state across
(stateless) HTTP requests to your app using the appserver's provided
abstraction for cookies.

**What you will do:** Create a Ralls app that makes use of the
HangpersonGame logic developed in the previous part, allowing you to
play Hangperson via a browser.

Game State
----------

Unlike a shrinkwrapped app, SaaS runs over the stateless HTTP protocol,
with each HTTP request causing something to happen in the app.  And
because HTTP is stateless, we must think carefully about the following
2 questions:  

0. What is the total state needed in order for the next HTTP
request to pick up the game where the previous request left off?

0. What are the different game actions, and how should HTTP requests map
to those actions?


The widely-used mechanism for maintaining state between a
browser and a SaaS server is a **cookie**.  The server can put whatever
it wants in the cookie (up to a length limit of 4K bytes); the browser
promises to send the cookie back to the server on each subsequent
requests. Since each user's browser gets its own cookie, the cookie can
effectively be used to maintain per-user state.

In most SaaS apps, the amount of information associated with a user's
session is too large to fit into the 4KB allowed in a cookie, so as
we'll see in Rails, the cookie information is more often used to hold a
pointer to state that lives in a database. But for this simple example,
the game state is small enough that we can keep it directly in the
session cookie.

* Enumerate the minimal game state that must be maintained
during a game of Hangperson.

> The secret word; the list of letters that have been guessed correctly;
> the list of letters that have been guessed incorrectly.  Conveniently,
> the well-factored HangpersonGame class encapsulates this state using
> its instance variables, as proper object-oriented design recommends.

The game as a RESTful resource
------------------------------

* Enumerate the player actions that could cause changes
in game state.

> Guess a letter: possibly modifies the lists of correct or incorrect
> guesses; possibly results in winning or losing the game.
>
> Start new game: chooses a new word and sets the incorrect and correct
> guess lists to empty.

In a service-oriented architecture, we do not expose
internal state directly; instead we expose a set of HTTP requests that
either display or perform some operation on a hypothetical
underlying **resource**.  **The trickiest and most important part of
RESTful design is modeling what your resources are and what
operations are possible on them.**

In our case, we can think of the game itself as the underlying
resource. Doing so results in some important design decisions about how
routes (URLs) will map to actions and about the game code itself.  
Since we've already identified the game state and player actions that
could change it, it makes sense to define the game itself as a class.
An instance of that class is a game, and represents the resource being
manipulated by our SaaS app.

Mapping resource routes to HTTP requests
----------------------------------------

Our initial list of operations on the resource might look like this,
where we've also given a suggestive name to each action:

0. `create`: Create a new game
0. `show`: Show the status of the current game
0. `guess`: Guess a letter

* Self-check: for a good RESTful design, which of the resource operations
should be handled by HTTP GET and which ones should be handled by HTTP
POST?

> Operations handled with `GET` should not have side effects on the
> resource, so `show` can be handled by a `GET`, but `create` and `guess`
> (which modify game state) should use `POST`.  (In fact,
> in a true service-oriented architecture we can also choose to use other
> HTTP verbs like `PUT` and `DELETE`, but we won't cover that in this
> assignment.)

HTTP is a request-reply protocol and the Web browser is
fundamentally a request-reply user interface, so each action by the user
must result in something being displayed by the browser.  
For the "show status of current game" action, it's pretty clear that
what we should show is the HTML representation of the current game, as
the `word_with_guesses` method of our game class does.
(In a fancier implementation, we would
arrange to draw an image of part of the hanging person.)

But when the player guesses a letter--whether the guess is correct or
not--what should be the "HTML representation" of the result of that
action?

Answering this question is where the design of many Web apps falters.

In terms of game play, what probably makes most sense is after the
player submits a guess, display the new game state resulting from the
guess.  **But we already have a RESTful action for displaying the game
state.**  So we can plan to use an **HTTP redirect** to make use of that action.

This is an important distinction, because an HTTP redirect triggers an
entirely new HTTP request.  Since that new request does not "know" what
letter was guessed, all of the responsibility for **changing** the game
state is associated with the guess-a-letter RESTful action, and all of
the responsibility for **displaying** the current state of the game
without changing it is associated with the display-status action.  This
is quite different from a scenario in which the guess-a-letter action
**also** displays the game state, because in that case, the game-display
action would have access to what letter was guessed.  Good RESTful
design will keep these responsibilities separate, so that each RESTful
action does exactly one thing.

A similar argument applies to the create-new-game action.  The
responsibility of creating a new game object rests with that action (no
pun intended); but once the new game object is created, we already have
an action for displaying the current game state.

So we can start mapping our RESTful actions in terms of HTTP requests as
follows, using some simple URIs for the routes:

<table>
<thead>
<tr>
<th> Route and action </th><th>  Resource operation</th> <th>  Web result</th>
</tr>
</thead>
<tbody>

<tr><td> `GET /show`      </td><td>   show game state      </td><td>  display correct & wrong guesses so far</td></tr>
<tr><td> `POST /guess`    </td><td>   update game state with new guessed letter </td><td> redirect to `show`</td></tr>
<tr><td> `POST /create`   </td><td>   create new game      </td><td>  redirect to `show`</td></tr>
</tbody>
</table>

In a true service-oriented architecture, we'd be nearly done.  But a
site experienced through a Web browser is not quite a true
service-oriented architecture.

Why?  Because a human Web user needs a way to `POST` a form.  A `GET`
can be accomplished by just typing a URL into the browser's address bar,
but a `POST` can only happen when the user submits an HTML form (or, as
we'll see later, when AJAX code in JavaScript triggers an HTTP action).

So to start a new game, we actually need to provide the user a way to
post the form that will trigger the `POST /create` action.  You can
think of the resource in question as "the opportunity to create a new
game."  So we can add another row to our table of routes:

<table>
<tbody>
<tr>
<td>`GET /new`   </td><td>  give human user a chance to start new game </td><td> display a form that includes a "start new game"  button  </td>
</tr>
</tbody>
</table>

Similarly, how does the human user generate the `POST` for guessing a
new letter?  Since we already have an action for displaying the current
game state (`show`), it would be easy to include on that same HTML page
a "guess a letter" form that, when submitted, generates the `POST
/guess` action.

We will see this pattern mirrored later in Rails: a typical resource
(such as the information about a player) will have `create` and `update`
operations, but to allow a human being to provide the data used to
create or update a player record, we will have to provide `new` and
`edit` actions respectively that allow the user to enter the information
on an HTML form.

* Self-check: why is it appropriate for the `new` action to use
`GET` rather than `POST`?  

> The `new` action doesn't by itself cause any state change: it just
> returns a form that the player can submit.

* Self-check: explain why the `GET /new` action wouldn't be needed if
your Hangperson game was called as a service in a true service-oriented
architecture. 

> In a true SOA, the service that calls Hangperson can generate an HTTP
> `POST` request directly.  The only reason for the `new` action is to
> provide the human Web user a way to generate that request.

Lastly, when the game is over (whether win or lose), we shouldn't be
accepting any more guesses.  Since we're planning for our `show` page to
include a letter-guess form, perhaps we should have a different type of
`show` action when the game has ended---one that does **not** include a
way for the player to guess a letter, but (perhaps) does include a
button to start a new game.  We can even have separate pages for winning
and losing, both of which give the player the chance to start a new
game.  Since the `show` action can certainly tell if the game is over,
it can conditionally redirect to the `win` or `lose` action when called.

* Self-check(*): show the route for each of the RESTful actions in the
game, based on the description of what the route should do; we've
provided the first one.

<table>
<tr>
<td>
Show game state, allow player to enter guess; may redirect to Win or
Lose  </td>
<td>GET /show</td>
</tr>
<tr>
<td>Display form that can generate `POST /create`   </td><td>  GET /new</td>
</tr>
<tr><td>Start new game; redirects to Show Game after changing
state</td><td>  POST /create</td></tr>
<tr><td>
Process guess; redirects to Show Game after changing state </td><td>POST
/guess</td></tr>
<tr><td>
Show "you win" page with button to start new game </td><td>GET
/win</td></tr>
<tr><td>
Show "you lose" page with button to start new game </td><td>GET
/lose</td></tr>
</table>


## Summary of the design

You may be itchy about not writing any code yet, but you have finished
the most difficult and important task: defining the application's basic
resources and how the RESTful routes will map them to actions in a SaaS
app.  To summarize:

* We already have a class to encapsulate the game itself, with instance
variables that capture the game's essential state and instance methods
that operate on it when the player makes guesses.  In the
model-view-controller (MVC) paradigm, this is our model.

* Using Rails, we will expose operations on the model via
RESTful HTTP requests.  In MVC, this is our controller.

* We will create HTML views and forms to represent the game state, to
allow submitting a guess, to allow starting a new game, and to display a
message when the player wins or loses.  In MVC, these are our views.

Connecting HangpersonGame to Rails
------------------------------------
In Rails HTTP requests (URLs+methods) are routed to controller actions. 
An *action* causes Rails to look for the file `views/`*action*`.erb` and 
run them through the Embedded Ruby processor,
which looks for constructions `<%= like this %>`, executes the Ruby code
inside, and substitutes the result.  The code is executed in the same
context as the call to `erb`, so the code can "see" any instance
variables set up in the controller action methods.

The Session
-----------

We've already identified the items necessary to maintain game state, and
encapsulated them in the game class.
Since HTTP is stateless, when a new HTTP request comes in, there is no
notion of the "current game".  What we need to do, therefore, is save the game
object in some way between requests.

If the game object were large, we'd probably store it in a database on
the server, and place an identifier to the correct database record into
the cookie.  (In fact, as we'll see, this is exactly what Rails apps
do.)
But since our game state is small, we can just put the whole
thing in the cookie.  Rails's `session` library lets us do this: in
the context of the Rails app, anything we place into the special
"magic" hash `session[]` is preserved across requests.  In fact,
objects placed there are *serialized* into a text-friendly form that is
preserved for us.

There is one other session-like object we will use.  In some cases
above, one action will perform some state change and then redirect to
another action, such as when the Guess action (triggered by `POST /guess`) 
redirects to the Show action (`GET /show`) to redisplay the
game state after each guess.  But what if the Guess action wants to
display a message to the player, such as to inform them that they have
erroneously repeated a guess?  The problem is that since every request
is stateless, we need to get that message "across" the redirect, just as
we need to preserve game state "across" HTTP requests.

To do this, we use the `flash[]`, which is a hash for remembering short messages that
persist until the *very next* request (usually a redirect), and are then
erased. 

* Self-check: Why does this save work compared to just storing those
messages in the `session[]` hash?

> When we put something in `session[]` it stays there until we delete
> it.  The common case for a message that must survive a redirect is
> that it should only be shown once; `flash[]` includes the extra
> functionality of erasing the messages after the next request.
