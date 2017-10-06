---
title: Building Eloquence game in Elm
category: Elm
---

<p>
I've decided to try out Elm language by implementing a game. At first glance, writing an article about it may appear to be as original as writing “Yet another tutorial about creating a Ruby on Rails blog”. Actually, it is 😆
</p>

<p>
I, strange though it may sound, consider Elm as a smooth introduction to Haskell and functional world. It combines such features as immutability and strong static typing (strong enough to ditch writing tests 😅) and “beginner-friendliness” by virtue of avoiding “ominous” abstractions for managing side effects. Still, it stays simple and easy to get the hang of.
</p>

<p>
As an exercise, what about building one of the games of the amazing <a href="https://www.elevateapp.com" target="_blank">Elevate App</a>? The rules are simple: just provide as many as possible synonyms to the highlighted word in the sentence.
<br/>
The source code is located <a
href="https://github.com/igor-drozdov/eloquence-game/blob/master/src/Main.elm"
target="_blank">
here </a>. I'm just going to highlight basic moments in the article.
</p>

<p>
Here’s how the complete app works:
</p>

<iframe
  src="https://runelm.io/c/g2l?pane=preview"
  width="100%"
  height="320"
  frameBorder="0"
  sandbox="allow-forms allow-popups allow-scripts allow-same-origin allow-modals">
</iframe>


<hr />

<h2> Initialize the app's backbone </h2>

<p>
  <a href="https://github.com/halfzebra/create-elm-app" target="_blank">Create Elm App</a> tool was used to initialize an application with no build configuration.
  Standard functions are generated (<code>init</code>, <code>update</code>, <code>view</code>, etc...) to be filled in the future.

{% highlight haskell %}
module Main exposing (..)

import Html exposing (Html, text, div, img)
import Html.Attributes exposing (src)

---- MODEL ----

type alias Model =
    {}

init : ( Model, Cmd Msg )
init =
    ( {}, Cmd.none )

---- UPDATE ----

type Msg
    = NoOp

update : Msg -> Model -> ( Model, Cmd Msg )
update msg model =
    ( model, Cmd.none )

---- VIEW ----

view : Model -> Html Msg
view model =
    div []
        [ img [ src "/logo.svg" ] []
        , div [] [ text "Your Elm App is working!" ]
        ]

---- SUBSCRIPTIONS ----

subscriptions : Model -> Sub Msg
subscriptions model =
    always Sub.none

---- PROGRAM ----

main : Program Never Model Msg
main =
    Html.program
        { view = view
        , init = init
        , update = update
        , subscriptions = subscriptions
        }
{% endhighlight %}
</p>

<hr />

<h2> Model </h2>

<strong>Model</strong> is the state of the application. First and foremost, let's specify its type.
Since the whole state consists of three stages (before a round is played, playing the round, after the round has been played), it should have corresponding types:

{% highlight haskell %}
type Model
    = StartRound
    | PlayingRound PlayingRoundState
    | GameOver GameOverState
{% endhighlight %}

<h5> StartRound </h5>

It's just a constant which identifies that the app is in the state of awaiting a round to be played.

<h5> PlayingRound </h5>

The stage provides a structure for providing an experience of viewing a sentence and enter
the synonyms for a specific word in the sentence.

{% highlight haskell %}
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
{% endhighlight %}

<ul>
  <li> <strong> words </strong> - the correct synonyms entered by a user </li>
  <li> <strong> sentence </strong> - a record which contains a text to display, a
  target word and the allowed synonyms for the word</li>
  <li> <strong> elapsed </strong> - the number of seconds passed from the beginning of a round </li>
  <li> <strong> word </strong> - the current word being entered by a user </li>
  <li> <strong> wrongWord </strong> - a flag which identifies whether the current word is correct </li>
</ul>

<h5> GameOver </h5>

The stage displays the number of correctly entered synonyms and a random synonym
which could have been entered (if there is one).

{% highlight haskell %}
type alias GameOverState =
    { hint : Maybe String
    , score : Int
    }
{% endhighlight %}

If a user enters all the possible synonyms for a word, there won't be a synonym,
that could be shown as a <strong> hint </strong> in the <strong> GameOver
</strong> state. All of a sudden <strong> hint </strong> won't equal <strong>
null </strong> or <strong> undefined </strong> in that case. <strong> hint
</strong> has <code> Maybe String </code>, which means that it can be either <code> Just
String </code> or <code> Nothing </code>. The type system requires processing every of 
these cases, which is one of the reasons why we are not going to meet cool <code>
Cannot read property 'a' of null </code> in Elm.

<hr />

<h2> Init </h2>

The function which specifies the initial state of the application. In our case,
it is <code> StartRound </code> screen.

{% highlight haskell %}
init : ( Model, Cmd Msg )
init =
    ( StartRound, Cmd.none )
{% endhighlight %}

<hr />

<h2> View </h2>

<strong>View</strong> is a way to view the state as HTML.

{% highlight haskell %}
view : Model -> Html Msg
view model =
    case model of
        StartRound ->
            renderStartScreen

        PlayingRound state ->
            renderPlayingScreen state

        GameOver state ->
            renderGameOverScreen state
{% endhighlight %}

Since the function just renders a particular screen depending on the type of the model,
I won't put the source code in the article. It still can be found <a
href="https://github.com/igor-drozdov/eloquence-game/blob/master/src/Main.elm"
target="_blank">
here </a>.

<hr />

<h2> Subscriptions </h2>

The application needs to update the timer every second and change the state to
<strong> GameOver </strong> after the 20 seconds from the start of a round is
passed. That's why a subscription needs to be defined.

<p>
  <strong> Subscriptions </strong> is how your application can listen for external input.
</p>

{% highlight haskell %}
subscriptions : Model -> Sub Msg
subscriptions model =
    Time.every second Tick
{% endhighlight %}

That will call <code> update </code> function every second with a <strong> Tick
</strong> message

<hr />

<h2> Update </h2>

<code> update</code> function is the most complicated part of the application,
which contains the decent part of the logic (at least in my case).
It is a way to update the state depending on the type of the message emitted as a
result of users' interaction with the application.

{% highlight haskell %}
update : Msg -> Model -> ( Model, Cmd Msg )
update msg model =
    case model of
        PlayingRound state ->
            updatePlayingRound msg state

        StartRound ->
            updateIdleState msg model

        GameOver state ->
            updateIdleState msg model
{% endhighlight %}

<h5> updateIdleState </h5>

<strong> StartRound </strong> and <strong> GameOver </strong> identically handle
the same types of messages, hence use the same function. A user should be able
to start (or restart) a round in these particular stages. That's why if the
message passed to <code> update</code> is of type <strong> StartGame </strong>
a hard-coded initial playing state is returned.

{% highlight haskell %}
updateIdleState : Msg -> Model -> ( Model, Cmd Msg )
updateIdleState msg model =
    case msg of
        StartGame ->
            let
                focus =
                    Dom.focus fieldId |> Task.attempt (\_ -> NoOp)

                playingModel =
                    let
                        sentence =
                            Sentence "The application is"
                                "great"
                                "Be more expressive"
                                (Set.fromList
                                    [ "awesome"
                                    , "amazing"
                                    , "breathtaking"
                                    , "exceptional"
                                    , "fabulous"
                                    , "glorious"
                                    , "impressive"
                                    , "incredible"
                                    , "majestic"
                                    , "magnificent"
                                    , "marvelous"
                                    , "superb"
                                    , "supercalifragilisticexpialidocious"
                                    , "unbelievable"
                                    , "unthinkable"
                                    ]
                                )
                    in
                        PlayingRound (PlayingRoundState [] sentence 0 "" False)
            in
                ( playingModel, focus )

        _ ->
            model ! []
{% endhighlight %}

There are two things, that I would like to point out here:

<ul>
  <li>
    A trick for setting a focus on a field is used here. As you see, it must be done
    by sending a specific task/command to the update function (<code>( playingModel, focus )</code>), not by <code> $(fieldId).focus() </code>
    from an arbitrary place, which gives you control over side effects in the
    application.
  </li>

  <li>
    <code> let in </code>, which I might overindulge in, lets you provide helper
    functions scoped to the given function. Right, the whole function became
    large, but it still easy to be read, because the logic separated into
    additional functions.
  </li>
</ul>

<h5> updatePlayingRound </h5>

The function handles users' interaction during a playing of a round.

{% highlight haskell %}
updatePlayingRound : Msg -> PlayingRoundState -> ( Model, Cmd Msg )
updatePlayingRound msg state =
    case msg of
        UpdateWord value ->
            PlayingRound { state | word = value }
                ! []

        AddWord ->
            if Set.member state.word state.sentence.synonyms then
                let
                    oldSentence =
                        state.sentence

                    newSentence =
                        { oldSentence | synonyms = Set.remove state.word oldSentence.synonyms }
                in
                    PlayingRound { state | words = state.word :: state.words, sentence = newSentence, word = "" }
                        ! []
            else
                ( PlayingRound { state | wrongWord = True }
                , Process.sleep 200 |> Task.perform (\_ -> ClearField)
                )

        ClearField ->
            PlayingRound { state | word = "", wrongWord = False } ! []

        EndRound words randomNumber ->
            let
                word =
                    Array.get randomNumber (Array.fromList words)
            in
                GameOver (GameOverState word (List.length state.words)) ! []

        Tick _ ->
            if state.elapsed == period then
                let
                    words =
                        Set.toList state.sentence.synonyms

                    command =
                        Random.generate (EndRound words) (Random.int 0 (List.length words - 1))
                in
                    ( PlayingRound state, command )
            else
                PlayingRound { state | elapsed = state.elapsed + 1 } ! []

        _ ->
            PlayingRound state ! []
{% endhighlight %}

There are two things, that I would like to explain here:

<ul>
  <li> <strong> AddWord </strong> either stores the correctly entered word or
  marks that the word is invalid (to paint the input field in red) and after 200
  milliseconds clears the field (needs to be done via commands) </li>

  <li> <strong> Tick </strong> either updates <code> elapsed </code> or ends a
  round by sending a command with a randomly generated number. Note, that
  generating a random number is also a side effect, which is, as expected, addressed by
  sending command. </li>
</ul>

<hr />

<h2> Conclusion </h2>

What I really liked during my experience of building this game is that the
development was led by the language itself. The Elm Architecture, type system,
and even immutability was defining the way the application must have been built.
It was also interesting to try these and other concepts (such as sum types) in action and see how they do something tangible👆
