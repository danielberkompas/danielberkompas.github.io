---
layout: post
title: "Telephonist: State Machines for Twilio"
categories:
  - elixir
---

After a couple months of work, I've finally got the library I've been working
toward for Twilio, and I'm calling it "[Telephonist][telephonist]". You can read
all about it [over on Github][telephonist], but here's a taste:

<!-- more -->

```elixir
defmodule CustomCallFlow do
  use Telephonist.StateMachine, initial_state: :choose_language

  state :choose_language, twilio, options do
    say "#{options[:error]}" # say any error, if present
    gather timeout: 10 do
      say "For English, press 1"
      say "Para español, presione 2"
    end
  end

  state :english, twilio, options do
    say "Proceeding in English..."
  end

  state :spanish, twilio, options do
    say "Procediendo en español..."
  end

  # If the user pressed "1" on their keypad, transition to English state
  def transition(:choose_language, %{Digits: "1"} = twilio, options) do
    state :english, twilio, options
  end

  # If the user pressed "2" on their keypad, transition to Spanish state
  def transition(:choose_language, %{Digits: "2"} = twilio, options) do
    state :spanish, twilio, options
  end

  # If neither of the above are true, append an error to the options and
  # remain on the current state
  def transition(:choose_language, twilio, options) do
    options = Map.put(options, :error, "You pressed an invalid digit. Please try again.")
    state :choose_language, twilio, options
  end
end
```

Use the state machine from your web framework like this:

```elixir
# The web framework shown here is pseudo-code
def index(conn, twilio) do
  options = %{} # Whatever I want to be able to use in my states and transitions
  state = Telephonist.CallProcessor.process(CustomCallFlow, twilio, options)
  render conn, xml: state.twiml
end
```

Which will render some nice TwiML:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<Response>
  <Gather timeout="10">
    <Say>For English, press 1</Say>
    <Say>Para español, presione 2</Say>
  </Gather>
</Response>
```

Unlike my [previous attempt at this][twiliomenu], Telephonist has _no 
dependency_ on any ORM layer, is very concurrent, and produces code that
is easy to maintain.

Telephonist will be released on [Hex][hex] soon, (once I get the documentation
finished) and in the future, I may create a Ruby version.

[telephonist]: https://github.com/danielberkompas/telephonist
[twiliomenu]: https://github.com/danielberkompas/twiliomenu
[hex]: http://hex.pm
