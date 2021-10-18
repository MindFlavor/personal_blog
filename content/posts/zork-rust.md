---
title: Let's build Zork using Rust! 
date: 2018-01-17
draft: false
description: "A simple state machine game in Rust"
tags: [ "rust", "statemachine", "codeforfun" ]
cover:
  image: /images/zork-rust/zork-rust.png
  alternate: something good
---

Games like Zork are basically big state machines. You advance in the game performing actions that lead your character from situation to situation. Eventually you either die horribly or win the game. The purpose of this post is to build a - simplified - textual game. We use it as a pretext to explore one way of treating state machines using Rust (yes, it's a clickbaity title)...

## State machines in Rust

There are various ways to model a state machine in Rust. Today we build on top of a gorgeous idea by [Florian Gilcher](https://twitter.com/Argorak) (you can see his original tweet here: [https://twitter.com/Argorak/status/940221231709683713](https://twitter.com/Argorak/status/940221231709683713)). Basically he suggests to model state passing around functions pointers. This works beautifully because you end up splitting your states in different functions instead of having a huge `match` statement.

Let's see some code first. We will comment it afterwards.

```rust
#[derive(Debug)]
struct Machine;

struct StateFn(fn(&mut Machine) -> StateFn);

impl Machine {
    fn start(&mut self) -> StateFn {
        println!("start");
        StateFn(Self::state)
    }

    fn state(&mut self) -> StateFn {
        println!("state");
        StateFn(Self::end)
    }

    fn end(&mut self) -> StateFn {
        println!("end");
        StateFn(Self::end)
    }
}
```

Here we have a `struct` called `Machine` which will hold some information that will be hard to model as a state machine (empty in our case). We also define another struct, `StateFn`, which holds the current state (expressed as a function). The convention, here, is that each *state function* will accept a mutable reference of `Machine` and will spit out the next state.

The syntax might be baffling at first so let's take a look at it. This line:

```rust
struct StateFn(fn(&mut Machine) -> StateFn);
```

Reads: create a `struct` called `StateFn`. This struct will have one implicit field. This field will accept only function pointers. The function pointed must have a single parameter - mutable reference of `Machine` - and will return a owned `StateFn`.

The state machine depicted above is this one:

![](/images/zork-rust/00.png#center)

To "run" it we can use this simple code:

```rust
fn main() {
    let mut m = Machine;
    let mut c = StateFn(Machine::start);
    println!("m == {:?}", m);

    c = c.0(&mut m);
    println!("m == {:?}", m);

    c = c.0(&mut m);
    println!("m == {:?}", m);

    c = c.0(&mut m);
    println!("m == {:?}", m);

    c = c.0(&mut m);
    println!("m == {:?}", m);
}
```

The `c.0()` syntax allows us to extract the first implicit field of `StateFn` and call it as a function. We pass the function our *world state* which is a `Machine` instance. Reassigning our `StateFn` binding simulates the evolution of the state machine. We can remove that `.0` function call implementing `Deref`.

### Deref

[Florian Gilcher](https://twitter.com/Argorak) gives us an elegant solution to get rid of the `c.0()` dereference. Rust allows us to implement custom deref behavior using the `Deref` trait. Let's do this:

```rust
impl Deref for StateFn {
    type Target = fn(&mut Machine) -> StateFn;

    fn deref(&self) -> &Self::Target {
        &self.0
    }
}
```

With this code we can simplify this call:

```rust
c = c.0(&mut m);
```

with this one:

```rust
c = c(&mut m);
```

So, to recap, we pass around functions that represent our state in the state machine. The functions will manipulate our *world*. With this information we can implement our Zork clone!

### The game

The game will be very simple: there will be just three rooms. This is the state machine of our game:

![](/images/zork-rust/01.png#center)

Here we have two "game-related" variables:

* Player owns the key or not
* Player has opened the door or not

Also, to give interactivity, we store in our "world" the command input by the player. Each state can inspect the command issued and act accordingly. For example we have a magic fountain in a room. The player may issue: "drink from the fountain". If we are in the right room we can let the avatar drink from the fountain. The same command can be invalid in another room though. Lastly we store the player name.

We can model it just like this:

```rust
#[derive(Debug)]
struct Player {
    name: String,
    has_key: bool,
}

#[derive(Debug)]
struct Game {
    player: Player,
    last_command: String,
    door_locked: bool,
}
```

Notice we have both "game-related" variables and "technical" variables jumbled together. This might not be desiderable: we could rid of the "game-related" variable by replacing them via specialized states. For example, instead of having a single *room* state we can have *room_door_locked* and *room_door_unlocked*. Something like this:

![](/images/zork-rust/02.png#center)

### The text processor

In order to build a text-based game you have to handle free form text. Given this is a sample of a Rust state machine I will cheat and just match predefined strings. For example the *room with the key* can be written like this:

```rust
fn key_room_with_key(&mut self) -> StateStruct<Game> {
    match &self.last_command as &str {
        "" => {
            println!("You are in a dark room.");
            StateStruct::input_required(Self::key_room_with_key)
        }
        "inspect" => {
            println!("You are in a dark room. You see a key on the floor.");
            StateStruct::input_required(Self::key_room_with_key)
        }
        "pick up the key" => {
            println!("You gingerly pick up the key and store it for later use.");
            self.player.has_key = true;
            StateStruct::input_required(Self::key_room_empty)
        }
        "go back" => {
            println!("You go back in the hallway.");
            StateStruct::no_input_required(Self::hallway)
        }
        _ => {
            println!("I don't know how to do that! What do you want to do?");
            StateStruct::input_required(Self::key_room_with_key)
        }
    }
}
```

As you can see we just match for specific input strings. Remember, the last received command will be in the `last_command` field. We than do three things:

1. Print something to give feedback to the user.
2. Change the world (optional) modifying our mutable reference.
3. Move to the new state. Here we use two helper functions, `input_required` and `no_input_required` to signal if we have to wait for player input before *activating* the new state.  

### Main 

The main method is just a loop. We start the state machine in the *start* state and *play* the state machine until we reach the *end* state. The main loop is oblivious of what's happening in the state machine, the state transition happen as result of state execution.

```rust
fn main() {
    use std::io::Write;
    let mut game = Game::default();
    
    // start the state machine.
    let mut sf = StateStruct::no_input_required(Game::start);

    // process the start state and progress to the next state.
    sf = sf(&mut game);

    // we play the machine until its end.
    while !sf.completed {

	// if the state requires input we ask the player to supply it.
        if sf.requires_input {
            let mut buffer = String::new();
            print!("> ");
            ::std::io::stdout().flush().unwrap();
            ::std::io::stdin().read_line(&mut buffer).unwrap();
            game.last_command = buffer[0..buffer.len() - 1].to_owned();
        } else {
            game.last_command = "".to_owned();
        }

	// now we play the next state and advance the machine.
        sf = sf(&mut game);
    }
}
```

As you can see the `main` code is straightforward. 

### Wrapping up

Now all we have to do is to implement the states our game will handle. The following complete code will implement the diagram above. Can you complete the dungeon without dying? Also, can you devise a more challenging dungeon to play? Let me know!

```rust
use std::ops::Deref;

type StateFn<T> = fn(&mut T) -> StateStruct<T>;
struct StateStruct<T> {
    function: StateFn<T>,
    requires_input: bool,
    completed: bool,
}

impl<T> StateStruct<T> {
    fn new(function: StateFn<T>, requires_input: bool, completed: bool) -> StateStruct<T> {
        StateStruct {
            function: function,
            requires_input: requires_input,
            completed: completed,
        }
    }

    fn input_required(function: StateFn<T>) -> StateStruct<T> {
        StateStruct::new(function, true, false)
    }

    fn no_input_required(function: StateFn<T>) -> StateStruct<T> {
        StateStruct::new(function, false, false)
    }

    fn completed(function: StateFn<T>) -> StateStruct<T> {
        StateStruct::new(function, false, true)
    }
}

impl<T> Deref for StateStruct<T> {
    type Target = StateFn<T>;

    fn deref(&self) -> &Self::Target {
        &self.function
    }
}

#[derive(Debug)]
struct Player {
    name: String,
    has_key: bool,
}

#[derive(Debug)]
struct Game {
    player: Player,
    last_command: String,
    door_locked: bool,
}

impl ::std::default::Default for Game {
    fn default() -> Self {
        Game {
            player: Player {
                name: "".to_owned(),
                has_key: false,
            },
            door_locked: true,
            last_command: "".to_owned(),
        }
    }
}

impl Game {
    fn reset(&mut self) {
        self.player.has_key = false;
        self.door_locked = true;
    }

    fn start(&mut self) -> StateStruct<Game> {
        println!("You wake up in hallway. Your memory is fuzzy... What's your name?");
        StateStruct::input_required(Self::save_name)
    }

    fn end(&mut self) -> StateStruct<Game> {
        println!("You eneded the game! {} wins! Congrats!", self.player.name);
        StateStruct::completed(Self::end)
    }

    fn save_name(&mut self) -> StateStruct<Game> {
        ::std::mem::swap(&mut self.player.name, &mut self.last_command);
        println!("Yes, that's right! You are {}!", self.player.name);
        StateStruct::no_input_required(Self::hallway)
    }

    fn hallway(&mut self) -> StateStruct<Game> {
        match &self.last_command as &str {
            "" => {
                println!("You are in a hallway. You can inspect it, go left or right.");
                StateStruct::input_required(Self::hallway)
            }
            "inspect" => {
                println!(
                    "You are in a hallway. It's unremarkable. You can go either right or left."
                );
                StateStruct::input_required(Self::hallway)
            }
            "go left" => {
                println!("You run left until you reach a dead end.");
                if !self.player.has_key {
                    StateStruct::no_input_required(Self::key_room_with_key)
                } else {
                    StateStruct::no_input_required(Self::key_room_empty)
                }
            }
            "go right" => {
                println!("You run left until you reach a dead end.");
                if self.door_locked {
                    StateStruct::no_input_required(Self::door_room_locked)
                } else {
                    StateStruct::no_input_required(Self::door_room_unlocked)
                }
            }
            _ => {
                println!("I don't know how to do that! What do you want to do?");
                StateStruct::input_required(Self::hallway)
            }
        }
    }

    fn key_room_with_key(&mut self) -> StateStruct<Game> {
        match &self.last_command as &str {
            "" => {
                println!("You are in a dark room.");
                StateStruct::input_required(Self::key_room_with_key)
            }
            "inspect" => {
                println!("You are in a dark room. You see a key on the floor.");
                StateStruct::input_required(Self::key_room_with_key)
            }
            "pick up the key" => {
                println!("You gingerly pick up the key and store it for later use.");
                self.player.has_key = true;
                StateStruct::input_required(Self::key_room_empty)
            }
            "go back" => {
                println!("You go back in the hallway.");
                StateStruct::no_input_required(Self::hallway)
            }
            _ => {
                println!("I don't know how to do that! What do you want to do?");
                StateStruct::input_required(Self::key_room_with_key)
            }
        }
    }

    fn key_room_empty(&mut self) -> StateStruct<Game> {
        match &self.last_command as &str {
            "" => {
                println!("You are in a dark room.");
                StateStruct::input_required(Self::key_room_empty)
            }
            "inspect" => {
                println!("You look around but there is nothing worth mentioning.");
                StateStruct::input_required(Self::key_room_empty)
            }
            "pick up the key" => {
                println!("There is no key to pick up!");
                StateStruct::input_required(Self::key_room_empty)
            }
            "go back" => {
                println!("You go back in the hallway.");
                StateStruct::no_input_required(Self::hallway)
            }
            _ => {
                println!("I don't know how to do that! What do you want to do?");
                StateStruct::input_required(Self::key_room_empty)
            }
        }
    }

    fn door_room_locked(&mut self) -> StateStruct<Game> {
        match &self.last_command as &str {
            "" => {
                println!("You are in a dimly lit room.");
                StateStruct::input_required(Self::door_room_locked)
            }
            "inspect" => {
                println!(
                    "You are in a dimly lit room. You notice a sickly looking fountain and a door."
                );
                StateStruct::input_required(Self::door_room_locked)
            }
            "drink from the fountain" => {
                println!("You drink the water and drop dead immediately. Tough luck!");
                self.reset();
                StateStruct::no_input_required(Self::start)
            }
            "unlock the door" => {
                if self.player.has_key {
                    println!("You use your key to unlock the door.");
                    self.door_locked = false;
                    StateStruct::input_required(Self::door_room_unlocked)
                } else {
                    println!("You do not have a key to use!");
                    StateStruct::input_required(Self::door_room_locked)
                }
            }
            "open the door" => {
                println!("The door is locked! You must find a key first!");
                StateStruct::input_required(Self::door_room_locked)
            }
            "go back" => {
                println!("You go back in the hallway.");
                StateStruct::no_input_required(Self::hallway)
            }
            _ => {
                println!("I don't know how to do that! What do you want to do?");
                StateStruct::input_required(Self::door_room_locked)
            }
        }
    }

    fn door_room_unlocked(&mut self) -> StateStruct<Game> {
        match &self.last_command as &str {
            "" => {
                println!("You are in a dimly lit room.");
                StateStruct::input_required(Self::door_room_unlocked)
            }
            "inspect" => {
                println!(
                    "You are in a dimly lit room. You notice a sickly looking fountain and an already unlocked door."
                );
                StateStruct::input_required(Self::door_room_unlocked)
            }
            "drink from the fountain" => {
                println!("You drink the water and drop dead immediately. Tough luck!");
                self.reset();
                StateStruct::no_input_required(Self::start)
            }
            "unlock the door" => {
                println!("The door is already unlocked!");
                StateStruct::input_required(Self::door_room_unlocked)
            }
            "open the door" => {
                println!("You open the door and escape the dungeon!",);
                StateStruct::no_input_required(Self::end)
            }
            "go back" => {
                println!("You go back in the hallway.");
                StateStruct::no_input_required(Self::hallway)
            }
            _ => {
                println!("I don't know how to do that! What do you want to do?");
                StateStruct::input_required(Self::door_room_unlocked)
            }
        }
    }
}

fn main() {
    use std::io::Write;
    let mut game = Game::default();
    let mut sf = StateStruct::no_input_required(Game::start);

    sf = sf(&mut game);

    while !sf.completed {
        // println!("game == {:?}", game);
        if sf.requires_input {
            let mut buffer = String::new();
            print!("> ");
            ::std::io::stdout().flush().unwrap();
            ::std::io::stdin().read_line(&mut buffer).unwrap();
            game.last_command = buffer[0..buffer.len() - 1].to_owned();
        } else {
            game.last_command = "".to_owned();
        }
        sf = sf(&mut game);
    }
}
```

---

Happy Coding,

**Francesco Cogno**

