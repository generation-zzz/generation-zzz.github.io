# Generation ZZZ - episode $EPISODE

A week of the show, from the running sheet to this site, in the order it has
to happen. Run it through `bin/episode` rather than on its own: that works out
which episode it is and hands this document `$EPISODE`, `$AIRED_ON`, `$ZZZ`
(the zzz checkout), `$SHOW_DB` and `$EPISODE_DIR`.

```
bin/episode           # this week's episode - last Friday's on a weekend
bin/episode 227       # a particular one
bin/episode --list    # the board, without doing anything
```

Every step is checked against the episode itself - the database, the episode
folder, the mail, the site - so quitting is free and re-running picks up where
things actually are. Steps with no way to check (the upload, Amrap, looking at
the page) are remembered instead, for this episode only.

It stops at the broadcast. Everything from "After it goes to air" on waits for
Friday 8pm, because until then there is no playlist to archive - building the
page earlier is how episode 226 went up with no tracks.

## Before you start

### The zzz gems

The zzz tools run from their checkout with bundler, and a new Ruby (mise
moves it along) arrives with none of their gems installed. This is what the
first command of the week falls over on if nothing else.

```bash {id=gems check="cd $ZZZ && bundle check"}
cd "$ZZZ" && bundle install
```

## Build the show

### Fill in the running sheet

Add a tab called `$EPISODE` to the running sheet and fill it in: 4ZZZ release
urls, a track number where it isn't track one, and durations for the talk
breaks. `zzz_sheet` polls that tab, fills in lengths, quota flags and running
times, and saves the running order to the show database each time round.

It loops forever, so run it in another terminal and leave it going while you
edit. Ctrl-c once the total looks right.

```manual {id=sheet needs=gems check="bin/episode done planned"}
cd $ZZZ && bundle exec exe/zzz_sheet $EPISODE
```

### Assemble the episode folder

Downloads every track and builds `$EPISODE_DIR`: the intro, the tracks in
order, a station id before each talk break, a language warning where one is
needed, and `$EPISODE.RPP` for Reaper. It is safe to re-run after changing the
sheet - it leaves an existing `.RPP` alone, so delete that to regenerate it.

```bash {id=prepare needs=sheet check="bin/episode done prepared"}
cd "$ZZZ" && bundle exec exe/zzz_prepare "$EPISODE"
```

### Record and render

Open `$EPISODE_DIR/$EPISODE.RPP` in Reaper, record the talk breaks into their
slots, and render the whole show to `$EPISODE_DIR/$EPISODE.mp3`.

```manual {id=render needs=prepare check="bin/episode done rendered"}
open "$EPISODE_DIR/$EPISODE.RPP"
...record, then render to $EPISODE_DIR/$EPISODE.mp3
```

## Send it to the station

### Upload the mp3

Put `$EPISODE.mp3` in the show's shared Google Drive folder. The link is in
last week's mail to Zed Digital.

```manual {id=upload needs=render}
Upload $EPISODE_DIR/$EPISODE.mp3 to the shared Drive folder.
```

### Mail Zed Digital

From the show's gmail, send the weekly mail with the subject
`generation zzz episode $EPISODE`: the folder link and a sentence about what's
in the show. Write that sentence for the site too - `zzz_describe` reads it
back out as the episode's description, so it is only ever written once.

```manual {id=mail needs=upload}
Send "generation zzz episode $EPISODE" to Zed Digital with the folder link and the blurb.
```

### Sync the sent mail

`zzz_describe` finds the mail with `mu`, which only sees what `mbsync` has
brought down and `mu index` has read.

```bash {id=sync needs=mail check="bin/episode done mailed"}
mbsync genzed && mu index --quiet
```

### Record the description

Reads the blurb out of that mail into the show database, where `bin/export`
picks it up for the site. `zzz_describe $EPISODE "..."` sets it by hand if the
mail says something different to what the site should.

```bash {id=describe needs=sync check="bin/episode done described"}
cd "$ZZZ" && bundle exec exe/zzz_describe "$EPISODE"
```

### Enter the Amrap playlist

Opens Chrome on the station's playlist form and logs in. Create the new show
there, press enter here, and it types the running order in - then check the
entries over in the browser and press enter again to close it.

This is also what `zzz_archive` reads back after the broadcast, so the site's
tracklist can only be as right as this is.

```bash {id=amrap needs=prepare}
cd "$ZZZ" && bundle exec exe/zzz_amrap "$EPISODE"
```

## After it goes to air

### Wait for the broadcast

The show goes out on Zed Digital at 7pm on Friday $AIRED_ON. Come back after
8pm; this ticks itself once it's over.

```manual {id=aired check="bin/episode done aired"}
Generation ZZZ is on air 7-8pm, Friday $AIRED_ON.
```

### Archive what went to air

Walks the show's Amrap playlists and records any episode that has aired since
the last run. The 2 is seconds between requests - it's someone else's server.

```bash {id=archive needs=aired,amrap check="bin/episode done archived"}
cd "$ZZZ" && bundle exec exe/zzz_archive "$SHOW_DB" 2
```

### Look the tracks up in the 4ZZZ library

Finds each new track's release, which is where the local, female, gender
diverse, First Nations and Australian flags on the site come from. Only new
tracks are asked about, so a week's worth takes a minute.

```bash {id=enrich needs=archive check="bin/episode done enriched"}
cd "$ZZZ" && bundle exec exe/zzz_enrich "$SHOW_DB" 3
```

### Classify what the library doesn't know

Artists the 4ZZZ library has never heard of get no flags, so the site shows
nothing for them. Tick the columns that apply (`a l f g i`, an `x` like the
running sheet), and remember a blank means no - delete an artist's row to leave
them for later. The export covers every unclassified artist, busiest first,
not just this week's.

Press `s` to leave it for another time; the site builds without it.

```manual {id=classify needs=enrich check="bin/episode done classified"}
cd $ZZZ
bundle exec exe/zzz_classify export tmp/unclassified.csv
...tick the columns...
bundle exec exe/zzz_classify import tmp/unclassified.csv
```

## Update the site

### Export from the show database

Writes `episodes.yml` and `tracks.yml`, which are all the site is built from.

```bash {id=export needs=describe,enrich check="bin/episode done exported"}
bin/export
```

### Check what 4ZZZ can still stream

4ZZZ only streams an episode for a couple of months, so each week one drops off
and the new one comes on. Only these get a player.

```bash {id=playable needs=export check="bin/episode done playable"}
bin/playable
```

### Build the pages

```bash {id=generate needs=playable check="bin/episode done built"}
bin/generate
```

### Look at it

Serve the site in another terminal and look at
http://localhost:8000/e$EPISODE - tracklist, tags, player, and the season and
stats pages. Ctrl-c when you're happy.

```manual {id=look needs=generate}
bin/serve
```

### Commit and push

`push` sends `@`, so check it only holds the site's changes first - anything
else in there goes too.

```bash {id=push needs=look check="bin/episode done pushed"}
jj st && jj describe -m "episode $EPISODE" && fish -c push
```

### Check it's live

GitHub Pages takes a minute or two after the push. This waits up to five.

```bash {id=live needs=push check="bin/episode done live"}
for i in $(seq 30); do bin/episode done live && echo live && exit 0; sleep 10; done; exit 1
```
