2023-04-02

# wordle with friends
tldr: i built [this](https://wordle-with-friends-26qcw2gcuq-uc.a.run.app) - realtime synced wordle

https://user-images.githubusercontent.com/10104080/229333733-13a788d4-6571-41d5-b2fe-03324bc5925c.mov


### prologue
so apparnetly [wordle is no longer popular](https://slate.com/human-interest/2022/09/wordle-nytimes-game-users-interview.html) but i only found out about it a month ago bahaha. just like other games, it's a lot more fun to play it with someone else and i can't seem to find an implementation of this online. to be precise, what i want here is multiple people sharing the same game session where the characters typed by one person become immediately visible to other players in the same session.

### prior work
i did some research first:
- [wordle for friends](https://www.wordleforfriends.com/) lets the user to provide the word to be guessed, but no game state sharing amongst the players. really fun idea, though.
- [wordle with friends](https://wordlewithfriends.app/) the state syncs, but the sync happens at the guess-level, not character-level (seems to be using firebase lol).
- [reactle](https://github.com/cwackerfuss/react-wordle) open source implementation of wordle.


### implementation
since i'm not exactly proficient with react or any frontend framework really so i've decided to take the easiest way: fork `reactle` and implement the syncing on top of it via `websocket`.

here is what i've built: [wordle with friends](https://wordle-with-friends-26qcw2gcuq-uc.a.run.app) (i realize the name is the same as another existing app alas). hitting the root path at that host should generate a new game along with the link that can be shared with those who you want to play with.

i tried to keep the setup as dead simple as possible so it's just a docker container running in gcp cloud run. if any of y'all want to help me offset the cost, venmo me at @Channyclaus :.)

### related
1. i am a HUGE fan of [entr](https://github.com/eradman/entr) since it eliminates the need for the programmer to manually hit a bunch of keys when iterating. i've only recently discovered this option `-r`, which autoreloads the server like
```
$ ls * | entr -rz ./httpd
```
i was like omg i'm going to use this all the freakin time!!!!

i started doing `git ls-files | entr -s -r "npx react-scripts build && npx ts-node app.ts"` for iterating, but then my macbook (m2 air 2022) would start crashing and rebooting. i'm relatively confident that this issue has something to do with `entr -r` since if i don't use that, my laptop never crashes. i haven't been able to pin down the cause enough however to make sure that this is either `entr` issue or `macos` issue. [issue](https://github.com/eradman/entr/issues/116) on the repo so hopefullly it'll help with visibility and in the meantime i'm trying to poke at it whenever i get a chance.

2. this work was partly inspired by the takehome assignment i had to do for an interview, which was qutie brutal bahaha. they asked me to add a feature to the existing scaffold to sync play, pause and seek on youtube videos and also implement replay feature. pretty cool stuff ngl but definitely got a little traumatized by watching the same video over and over again while testing.
