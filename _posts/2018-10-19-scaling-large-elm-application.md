---
title: Scaling Large Elm Applications
excerpt: Mesmerized by the buzz topics such as "scaling large applications" and "modular development", I dared to refactor one of my one-file apps. You won't beleive what happened next... 🤭
category: Elm
---

In <a href="/elm/2017/10/05/building-eloquence-game-in-elm/" target="_blank">that article</a> I developed an application which works fine, but not very convenient to extend. That is why my goal is to take advantage of a tree structure and optimally break the application down to multiple files with a narrow scope, in order to keep fewer things in mind on changing a particular part of the application.

<blockquote>
<p>
The goal is to take advantage of a tree structure of an application since my brain processes linear structures with O(n) complexity.
</p>
</blockquote>

By the way <a href="https://www.elm-tutorial.org/en-v01/02-elm-arch/07-composing-2.html" target="_blank"> this guide </a> would perfectly solve this kind of a problem for the applications with models of <a href="https://en.wikipedia.org/wiki/Product_type" target="_blank"> product type </a> and it is actually cool how the app starts kind of consisting of smaller subwidgets, which are <a href="https://en.wikipedia.org/wiki/Closure_(mathematics)" target="_blank"> closed under a collection of operations</a>, i.e. a submodel has the same type after performing an operation from the collection.

The model of my application is particularly of <a href="https://en.wikipedia.org/wiki/Tagged_union" target="_blank">sum types</a>, which comprises different states: `Start`, `Playing` and `GameOver` -- the concepts which potentially can be extracted into smaller subwidgets. The problem is that the application should be able to transit from one state to another, i.e. these subwidgets no longer have the closure under all the commands that trigger the update, since one of the commands should update a submodel to a different state.

<blockquote>
<p>
The problem is to scale an app with transitions from one state to another, which is quite natural for the games since games often have different states of the whole application with an independent state.
</p>
</blockquote>

Let's start from creating three separate files for every state and moving the code there: `Start.elm`, `Playing.elm` and `GameOver.elm`.

<h5> Model </h5>

<br/>
The model structure changes from

<em> Main.elm </em>
{% highlight elm %}
type Model
    = StartRound
    | PlayingRound PlayingRoundState
    | GameOver GameOverState

type alias Sentence =
    { prefix : String
    , word : String
    , description : String
    , synonyms : Set.Set String
    }

type alias PlayingRoundState =
    { words : List String
    , sentence : Sentence
    , elapsed : Float
    , word : String
    , wrongWord : Bool
    }

type alias GameOverState =
    { hint : Maybe String
    , score : Int
    }
{% endhighlight %}

To

<em> Main.elm </em>
{% highlight elm %}
import Start as StartWidget
import Playing as PlayingWidget
import GameOver as GameOverWidget

type Model
    = Start
    | Playing PlayingWidget.Model
    | GameOver GameOverWidget.Model
{% endhighlight %}

<em> Playing.elm </em>
{% highlight elm %}
type alias Sentence =
    { prefix : String
    , word : String
    , description : String
    , synonyms : Set.Set String
    }

type alias Model =
    { words : List String
    , sentence : Sentence
    , elapsed : Float
    , word : String
    , wrongWord : Bool
    }
{% endhighlight %}

<em> GameOver.elm </em>
{% highlight elm %}
type alias Model =
    { score : Int
    , hint : Maybe String
    }
{% endhighlight %}

<h5> Messages </h5>

<br/>
Currently, the messages structure is

<em> Main.elm </em>
{% highlight elm %}
type Msg
    = NoOp
    | StartGame
    | UpdateWord String
    | AddWord
    | ClearField
    | EndRound (List String) Int
    | Tick Time
    | RestartGame
{% endhighlight %}

Let us notice that:

<ul>
  <li>
   <code class="highlighter-rouge">StartGame</code>
   message is triggered during <code class="highlighter-rouge">Start</code>
   state and transmits the model into <code class="highlighter-rouge">Playing</code> state
  </li>
  <li>
   <code class="highlighter-rouge">RestartGame</code>
   message is triggered during
   <code class="highlighter-rouge">GameOver</code>
   state and transmits the model into
   <code class="highlighter-rouge">Playing</code>
   state
  </li>
  <li>
   <code class="highlighter-rouge">UpdateWord</code>
   <code class="highlighter-rouge">AddWord</code> 
   <code class="highlighter-rouge">ClearField</code> and
   <code class="highlighter-rouge">Tick</code> 
   messages are triggered during
   <code class="highlighter-rouge">Playing</code>
   state and do not change the type of state, but
   <code class="highlighter-rouge">EndRound</code>
   changes
  </li>
</ul>

<blockquote>
<p>
The idea is to create a
<code class="highlighter-rouge">Transition</code>
message for every state and process it on a higher level in order to transit the application to another state.
</p>
</blockquote>

<em> Main.elm </em>
{% highlight elm %}
type Msg
    = StartMsg StartWidget.Msg
    | PlayingMsg PlayingWidget.Msg
    | GameOverMsg GameOverWidget.Msg
    | NoOp
{% endhighlight %}

<em> Start.elm </em>
{% highlight elm %}
type Msg
    = Transition
{% endhighlight %}

<em> Playing.elm </em>
{% highlight elm %}
type Msg
    = SentenceMsg Sentence.Msg
    | Tick Time
    | EndGame (List String) Int
    | Transition Int (Maybe String)
{% endhighlight %}

<em> GameOver.elm </em>
{% highlight elm %}
type Msg
  = Transition
  | Restart
{% endhighlight %}

<h5> Update </h5>

As a result, we receive quite general `update` function, which either calls `update` functions of a subwidget depending on the message received or transmits the model into another type of state

<em> Main.elm </em>
{% highlight elm %}
update : Msg -> Model -> ( Model, Cmd Msg )
update msg model =
    case ( msg, model ) of
        ( StartMsg StartWidget.Transition, _ ) ->
            let
                ( newModel, subMsg ) =
                    PlayingWidget.init
            in
                ( Playing newModel, Cmd.map PlayingMsg subMsg )

        ( PlayingMsg (PlayingWidget.Transition score hint), _ ) ->
            ( GameOver <| GameOverWidget.init score hint, Cmd.none )

        ( PlayingMsg m, Playing state ) ->
            let
                ( newModel, subMsg ) =
                    PlayingWidget.update m state
            in
                ( Playing newModel, Cmd.map PlayingMsg subMsg )

        ( GameOverMsg GameOverWidget.Transition, _ ) ->
            let
                ( newModel, subMsg ) =
                    PlayingWidget.init
            in
                ( Playing newModel, Cmd.map PlayingMsg subMsg )

        ( GameOverMsg m, GameOver state ) ->
            let
                ( newModel, subMsg ) =
                    GameOverWidget.update m state
            in
                ( GameOver newModel, Cmd.map GameOverMsg subMsg )

        _ ->
            model ! []
{% endhighlight %}

<h5> View </h5>

The app renders view according to the current state and wraps the outgoing messages from subviews using `Html.map`

<em> Main.elm </em>
{% highlight elm %}
view : Model -> Html Msg
view model =
    case model of
        Start ->
            Html.map StartMsg <| StartWidget.view

        Playing state ->
            Html.map PlayingMsg <| PlayingWidget.view state

        GameOver state ->
            Html.map GameOverMsg <| GameOverWidget.view state
{% endhighlight %}

Subscriptions are handled similarly (the whole app can be viewed on <a href="https://github.com/igor-drozdov/eloquence-game/blob/master/src/Main.elm" target="_blank"> GitHub </a>)

<hr />

<h2> Conclusion </h2>

Since we deal with mere functions and groups of functions (modules), we possess enough flexibility to say that the provided example is not the only way to structure your application. For instance, we could pass `PlayingMsg` function into `update` functions of `Start` or `GameOver` modules instead of processing `Transition` messages. And it is great, that we can choose an approach which fits best for solving a particular problem ✊
