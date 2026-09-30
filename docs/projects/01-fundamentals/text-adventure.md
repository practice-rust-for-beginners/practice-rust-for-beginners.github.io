# Project: Text Adventure Game

**Track:** 🦀 Fundamentals &nbsp;|&nbsp; **Difficulty:** 🟢 Beginner &nbsp;|&nbsp; **Crates:** standard library only

---

## What You'll Build

A terminal text adventure where the player explores rooms, picks up items, talks to NPCs, and solves a simple puzzle to win. Every mechanic is built with Rust enums, structs, and pattern matching.

```text
You are in the ENTRANCE HALL.
A dusty corridor stretches before you.
Exits: north, east
Items here: rusty key

> take rusty key
You pick up the rusty key.

> go north
You are in the LIBRARY.
Tall shelves line every wall.
Exits: south
Items here: ancient tome

> examine ancient tome
The tome reads: "The vault opens when the key meets the lock."

> inventory
- rusty key

> go south
...
```

---

## Tech Stack

| Component | Choice |
|-----------|--------|
| Game state | Structs + enums |
| Room graph | `HashMap<RoomId, Room>` |
| Command parsing | Pattern matching on `&str` tokens |
| I/O | `std::io` REPL |

---

## Step 1 — Project Setup

```bash
cargo new text-adventure
cd text-adventure
```

---

## Step 2 — Define the World Types

```rust
// src/world.rs
use std::collections::HashMap;

#[derive(Debug, Clone, PartialEq, Eq, Hash)]
pub enum RoomId {
    EntranceHall,
    Library,
    Vault,
    Garden,
}

#[derive(Debug, Clone)]
pub struct Room {
    pub name:        &'static str,
    pub description: &'static str,
    pub exits:       HashMap<Direction, RoomId>,
    pub items:       Vec<Item>,
}

#[derive(Debug, Clone, PartialEq, Eq, Hash)]
pub enum Direction { North, South, East, West }

impl Direction {
    pub fn from_str(s: &str) -> Option<Self> {
        match s {
            "north" | "n" => Some(Direction::North),
            "south" | "s" => Some(Direction::South),
            "east"  | "e" => Some(Direction::East),
            "west"  | "w" => Some(Direction::West),
            _              => None,
        }
    }
}

#[derive(Debug, Clone, PartialEq)]
pub struct Item {
    pub name:        &'static str,
    pub description: &'static str,
}
```

---

## Step 3 — Build the World

```rust
// src/world.rs (continued)
pub fn build_world() -> HashMap<RoomId, Room> {
    let mut rooms = HashMap::new();

    rooms.insert(RoomId::EntranceHall, Room {
        name: "Entrance Hall",
        description: "A dusty corridor stretches before you.",
        exits: HashMap::from([
            (Direction::North, RoomId::Library),
            (Direction::East,  RoomId::Garden),
        ]),
        items: vec![Item { name: "rusty key", description: "An old iron key, slightly corroded." }],
    });

    rooms.insert(RoomId::Library, Room {
        name: "Library",
        description: "Tall shelves line every wall.",
        exits: HashMap::from([
            (Direction::South, RoomId::EntranceHall),
            (Direction::West,  RoomId::Vault),
        ]),
        items: vec![Item { name: "ancient tome", description: "\"The vault opens when the key meets the lock.\"" }],
    });

    rooms.insert(RoomId::Vault, Room {
        name: "Vault",
        description: "A heavy iron door stands before you. It has a keyhole.",
        exits: HashMap::from([(Direction::East, RoomId::Library)]),
        items: vec![],
    });

    rooms.insert(RoomId::Garden, Room {
        name: "Garden",
        description: "Sunlight filters through overgrown vines.",
        exits: HashMap::from([(Direction::West, RoomId::EntranceHall)]),
        items: vec![Item { name: "flower", description: "A bright red poppy." }],
    });

    rooms
}
```

---

## Step 4 — Game State

```rust
// src/game.rs
use crate::world::{RoomId, Item, Direction, build_world};
use std::collections::HashMap;
use crate::world::Room;

pub struct GameState {
    pub current_room: RoomId,
    pub inventory:    Vec<Item>,
    pub rooms:        HashMap<RoomId, Room>,
    pub vault_open:   bool,
    pub won:          bool,
}

impl GameState {
    pub fn new() -> Self {
        Self {
            current_room: RoomId::EntranceHall,
            inventory:    Vec::new(),
            rooms:        build_world(),
            vault_open:   false,
            won:          false,
        }
    }

    pub fn current(&self) -> &Room {
        self.rooms.get(&self.current_room).unwrap()
    }

    pub fn describe_room(&self) {
        let room = self.current();
        println!("\nYou are in the {}.", room.name.to_uppercase());
        println!("{}", room.description);
        let exits: Vec<String> = room.exits.keys()
            .map(|d| format!("{d:?}").to_lowercase()).collect();
        println!("Exits: {}", exits.join(", "));
        if !room.items.is_empty() {
            let names: Vec<&str> = room.items.iter().map(|i| i.name).collect();
            println!("Items here: {}", names.join(", "));
        }
    }
}
```

---

## Step 5 — Command Processing

```rust
// src/commands.rs
use crate::game::GameState;
use crate::world::Direction;

pub fn process(input: &str, state: &mut GameState) {
    let parts: Vec<&str> = input.trim().splitn(2, ' ').collect();
    let verb = parts[0];
    let noun = parts.get(1).copied().unwrap_or("");

    match verb {
        "go" | "move" => {
            if let Some(dir) = Direction::from_str(noun) {
                let next = state.current().exits.get(&dir).cloned();
                match next {
                    Some(RoomId::Vault) if !state.vault_open => {
                        println!("The vault door is locked.");
                    }
                    Some(room_id) => {
                        state.current_room = room_id;
                        state.describe_room();
                        if state.current_room == crate::world::RoomId::Vault {
                            println!("\n✨ You found the treasure! You win!");
                            state.won = true;
                        }
                    }
                    None => println!("You can't go that way."),
                }
            } else {
                println!("Go where? (north, south, east, west)");
            }
        }
        "take" | "pick" => {
            let room = state.rooms.get_mut(&state.current_room).unwrap();
            if let Some(pos) = room.items.iter().position(|i| i.name == noun) {
                let item = room.items.remove(pos);
                println!("You pick up the {}.", item.name);
                state.inventory.push(item);
            } else {
                println!("There's no '{}' here.", noun);
            }
        }
        "use" => {
            if noun == "rusty key" && state.current_room == crate::world::RoomId::Vault {
                if state.inventory.iter().any(|i| i.name == "rusty key") {
                    println!("The key turns. The vault door swings open!");
                    state.vault_open = true;
                } else {
                    println!("You don't have the rusty key.");
                }
            } else {
                println!("You can't use that here.");
            }
        }
        "examine" | "look" => {
            let found = state.current().items.iter()
                .find(|i| i.name == noun)
                .or_else(|| state.inventory.iter().find(|i| i.name == noun));
            match found {
                Some(item) => println!("{}", item.description),
                None if noun.is_empty() => state.describe_room(),
                None => println!("You don't see a '{}' here.", noun),
            }
        }
        "inventory" | "i" | "inv" => {
            if state.inventory.is_empty() {
                println!("Your inventory is empty.");
            } else {
                for item in &state.inventory { println!("- {}", item.name); }
            }
        }
        "help" => println!("Commands: go <dir>, take <item>, use <item>, examine <item>, inventory, quit"),
        _ => println!("I don't understand '{}'. Type 'help' for commands.", input),
    }
}
```

---

## Step 6 — Main Loop

```rust
// src/main.rs
mod world;
mod game;
mod commands;

use std::io::{self, Write};
use game::GameState;

fn main() {
    println!("=== THE VAULT ===");
    println!("A Rust text adventure. Type 'help' for commands.\n");

    let mut state = GameState::new();
    state.describe_room();

    loop {
        if state.won { break; }
        print!("\n> ");
        io::stdout().flush().unwrap();

        let mut input = String::new();
        io::stdin().read_line(&mut input).unwrap();
        let input = input.trim();

        if matches!(input, "quit" | "exit" | "q") { println!("Goodbye!"); break; }
        if !input.is_empty() { commands::process(input, &mut state); }
    }
}
```

---

## Step 7 — Run It

```bash
cargo run
```

---

## Stretch Goals

- [ ] Add more rooms, items, and a longer puzzle chain
- [ ] Add an NPC with dialogue using a `HashMap<&str, &str>` response table
- [ ] Save/load game state to a JSON file using `serde_json`
- [ ] Add a combat system with an `Enemy` struct and `fight` command

---

[Back to All Projects](../index.md){ .md-button }


[Next Project: File Organiser](file-organiser.md){ .md-button .md-button--primary }
