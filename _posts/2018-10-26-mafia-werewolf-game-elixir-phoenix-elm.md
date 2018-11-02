---
title: ! "Mafia game using Elixir + Elm: Backend"
excerpt: How cool is it to have a side project and implement interesting ideas from time to time? Decided to check it out...🤫
category: Elixir
---

<a href="https://en.wikipedia.org/wiki/Mafia_(party_game)"> Mafia (a.k.a Werewolf)</a> is a party game modeling a <b>conflict</b> between two groups: an informed minority (<b>the mafia</b>), and an uninformed majority (<b>the innocents</b>), while I am trying to model a <b>harmony</b> between Backend (<b>Elixir</b>) and Frontend (<b>Elm</b>):

{% highlight haskell %}
import Elm
import Elixir

type alias Mafia =
   { backend : Elixir
   , frontend : Elm
   }

type alias Wefewolf = Mafia
{% endhighlight %}

In these <b>blog post series</b> I will cover the main idea, the intuition behind the backend architecture and the moments which I found interesting on the frontend. Therefore, I am going to divide the series into <b>two parts</b>: the first one about `Elixir` and the second one about `Elm` (mostly because currently only one category can be set to a blog post and I was perplexed which one to choose 😅)

The first working version is ready and recently upgraded to `Elm 0.19` and `Elixir 1.4.0-rc.2` and can be found on <a href="https://github.com/igor-drozdov/mafia" target="_blank"> <b> GitHub </b> </a>.

<h4> Synopsis </h4>
Even though this is <b>a web application</b>, the <b>game is actually offline</b>. The main idea is to <b>replace the narrator</b> which asks a city to sleep/wake up, remembers the choices of the mafia, laughs at the ignorance of the players.

A screen visible to everybody (for example a TV screen) plays the role of a narrator and displays which state of the game it is (Leader window). Players, being in the same room, connect to the game via a mobile device and therefore has a separate screen (Follower window) for viewing the personal data (which role in the game they were assigned, UI for nominating a particular user). Depending on the current state, `Leader window` either displays call-to-action on the screen and/or plays a voice message.

The following video demonstrates the current game flow (but actually without audio):

{% include video.html id="mtTelzvYjUg" %}

Another way to see it in action is to <a href="https://github.com/igor-drozdov/mafia#tests" target="_blank"> run the acceptance test </a> within `demo` environment.

Or just play with it: <a href="https://mafia-game.gq" target="_blank">https://mafia-game.gq</a>

<h4> Architecture </h4>

My personal requirements to the backend architecture were:
<ul>
  <li> Backend is the main logic container </li>
  <li> It is flexible enough for adding new behavior (new roles, new changes in the game flow) </li>
</ul>

<blockquote>
<p>
  From the high-level perspective playing real-world Mafia game reminds transition between states:
   <code class="highlighter-rouge">Day Time</code> ⇒
   <code class="highlighter-rouge">Night Time</code> ⇒
   <code class="highlighter-rouge">Day Time</code> ⇒
   <code class="highlighter-rouge">...</code>.
  From the lower level, it reminds transition between more specific states:
  <code class="highlighter-rouge">City Sleeps</code> ⇒
  <code class="highlighter-rouge">Mafia Wakes Up</code> ⇒
  <code class="highlighter-rouge">...</code>
</p>
</blockquote>

Based on the intuition from the real-world game I decided to reflect each state in the code-level module and transfer from one state to another.
If a state contains too much responsibility, it can be split on multiple more specific ones and therefore the logic will be extracted into separate modules at the code-level.
For example, `Night Time` holds too much responsibility for a state, what about creating more specific ones instead: `City Sleeps`, `Mafia Wakes Up`, `Mafia Sleeps`...
I've been probably too lyrical and named such states as `chapters`🙃

<blockquote>
<p>
As a result, it turned out into a linear structure where every chapter executes and at the end calls the next one.
Like classic job workers, but with an essential difference: some of the chapters have indefinite time of execution.
For example, when mafia makes its choice, the worker must wait for a signal to proceed: this is where Elixir's (Erlang's) processes with messages really shines💡
</p>
</blockquote>

Below is a diagram with all the chapters that exist at the moment. Arrows define the transitions, which are in some cases conditional (for example, when the winner is chosen).

<img src="/assets/images/posts/state-diagram.svg"/>

Code-wise mostly each of these chapters has a separate module created (`PlayerN` ones are obviously created dynamically). The module represents a `GenServer` process.  During its lifetime the process performs a particular action and/or sends a command to the frontend via websockets in order to control the game flow (display a message, play a sound and etc...). After its execution the process stops `normally` using `{:stop, :shutdown, state}` as a return value from a callback.

As a result, it has come down to three types of processes:
<ul>
<li>A process which executes and stops: The simplest case. No special solution was required here.</li>
<li>A process which executes and stops after some time: In this case, the process must wait (sleep 😉) for some time. I struggled the temptation to use <code class="highlighter-rouge">:timer.sleep</code> and took advantage of <code class="highlighter-rouge">Process.send_after</code> to perform some task and termination after some specified amount of time.</li>
<li>A process which waits for a user input: Actually, there is no something special here. The process just waits for a message from a user in order to process it in a callback. After the processing, it performs the termination.</li>
</ul>

Has the architecture satisfied my personal requirements?

<ul style="list-style: none;">
  <li> ✅ The application behavior is driven by the backend, which sends ws messages to the frontend. </li>
  <li> ✅ In order to extend the functionality (for example, adding some behavior of a new role in the game flow) it is enough to create a chapter. </li>
</ul>

<hr />
<h4> Conclusion </h4>

I liked the experience of working on this game. Process-oriented paradigm seemed like a natural choice for my type of problem. `Elixir` has great features as a language itself and growing ecosystem which provides solid tools, such as `Phoenix`, `Hound` and etc, which helped and will help me to enhance the application.
<br />
Next time I'm going to describe the frontend part of my application written in `Elm`. Stay tuned 🤙
