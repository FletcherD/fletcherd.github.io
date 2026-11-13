+++
title = "Name 100: checking names against Wikidata"
description = "Notes on building a 'name 100 women' game where the name lookup is done against an index of 4.4 million people built from a Wikidata dump."
date = 2026-11-13

[taxonomies]
tags = ["Rust", "Wikidata", "game dev", "web", "Claude Code"]

[extra]
toc = true
+++

**[Play it](https://fletcher.cool/name100/)** • **[Source on GitHub](https://github.com/FletcherD/name100)**

There's a browser game going around where you name 100 women as fast as you can. The versions I found online all have problems with the part that matters, which is deciding whether what you typed is a real woman. The one I was shown rejected Billie Eilish and Kamala Harris. I started thinking about how I'd do that lookup properly, and ended up building it over three days, November 10 to 12, with Claude Code writing the code.

## The first design, and what I dropped

My first idea was a database of women seeded from the top Wikipedia articles, with a small language model as a fallback: when a name isn't in the database, the model searches and decides whether it's a valid answer, and the result is cached.

Talking it through, the model went away. Wikidata already has what the model would have been guessing at. Every person has a label and a list of aliases, and the aliases include things like "RBG", "AOC", stage names and maiden names. So the plan became: build an index from Wikidata, do exact and typo-tolerant matching against it, log every miss, and only add something smarter if the miss log shows real women being rejected. So far nothing smarter has been needed, though the game hasn't had many players yet.

## The index

The builder reads the Wikidata JSON dump, which is about 100 GB compressed and has 122 million entities. It keeps every human and fictional human with at least one Wikipedia article in any language, and stores their English and multilingual (`mul`) labels and aliases. That comes to:

- 4,431,846 people: 852,869 women, 3,548,219 people who aren't women, and 30,758 fictional characters
- 5,826,959 names
- a 726 MB index

Men and fictional characters are in the index on purpose. They let the game say "Elon Musk is male" or "Hermione Granger is fictional" instead of "not found", and they stop a name shared with a much more famous man from quietly resolving to an obscure woman.

Reading the compressed dump takes 68 minutes. Since I was rebuilding often, I added a `slim-dump` step that cuts the dump down to the 37 million entities a build can use (12.6 GB). A build from that takes about 100 seconds.

Names go into an [fst](https://github.com/BurntSushi/fst) map. Person records are sorted by popularity and split into a small "hot" file with what matching needs and a "cold" file with what's only needed to display a result. The server is a single Rust binary that loads in under a second and uses about 285 MB of memory. An exact lookup takes about 5 µs in-process, and a typo correction about 310 µs.

## Typos

The first version allowed up to two edits against every name in the index. I typed "some woman" as a junk test and it was accepted as Casey Johnston, who has the alias "Swole Woman" in Wikidata, two edits away. It was also slow, at 2–3 ms on a partial index.

Typo correction is now one edit (a swapped pair of letters counts as one), and only towards people with at least 20,000 page views a year. That's about 585,000 correctable names in a separate small index. "Taylor Swfit" works. "some woman" is rejected.

## Measuring fame

The lookup needs a fame measure in several places: choosing between people who share a name, deciding who typos can correct to, and deciding when one word is enough.

I started with the number of Wikipedia language editions a person has an article in. That doesn't work well. Some people have had their article translated into a hundred languages by a few dedicated editors: Hebe Camargo has 148 editions and 134,000 views a year, which put her above Taylor Swift. It also couldn't separate namesakes.

I switched to [QRank](https://qrank.toolforge.org/), which is the sum of page views over the last 52 weeks across all Wikimedia projects and languages for each Wikidata item. It separates namesakes much better (Steffi Graf has about 94 times the views of the runner Stephanie Graf), and it favours people who've been in the news recently, which is good, because those are people players guess.

## One-word names

If you type "Jennifer", the game shouldn't pick the most popular Jennifer for you. But some people are known by one name, and I wanted to err on the side of allowing those, because otherwise there's no reasonable way to enter them.

The rule I ended up with has two thresholds. If the word is the person's own name, like Sophie, Mitski or Laufey, they need 300,000 views a year. If the word is only an alias, they need 1,000,000. The second threshold exists because a flat 300,000 let "Elizabeth" resolve to a wrestling manager who has it as an alias. The people who are really meant by one word of their name (Diana, Oprah, Kamala) are above a million.

## Wikipedia redirects

Wikidata's aliases still miss things. "Britney" isn't listed as an alias of Britney Spears, for example. English Wikipedia's redirects cover a lot of these: "Britney" redirects to her article, and so do common misspellings like "Ruth Bader Ginsberg" and "Hilary Clinton". Redirects also catch misspellings that are more than one edit away.

So the builder also reads English Wikipedia's redirect table, and the lookup order is name, then redirect, then typo correction. I only use redirects to a whole article. Redirects to a section of an article are mostly not names for the person: they're songs, lists, and relatives who don't have their own article. In a sample of 70 that would have become new names, none was a name for the woman herself. There are 1.27 million section redirects and dropping them costs almost nothing.

Whole-article redirects still let through a few things that aren't names, like a song title redirecting to the singer. I decided that was fine, since players are trying to type names.

## Gender

The game uses whatever Wikidata records as the person's gender, which means it's sometimes wrong or arguable. Miley Cyrus is rejected, because Wikidata's only current value for her is genderfluid. Olivia Colman is rejected too, because Wikidata records only non-binary for her. That one is one of six known failures in my test file.

I didn't want to hand-patch individual people. Rejections are short ("Miley Cyrus is genderfluid") and have a "report as wrong" link, and the server keeps a deduplicated count of reports. The page says up front that the data comes from Wikidata and is sometimes wrong.

## Other categories

Early on I made the builder store raw attributes for each person (gender, occupation, citizenship, awards and so on, eleven in all) instead of a single "is a woman" flag, so the same index could run other games. That turned into a search box: you can type "actors" or "Nobel Prize" and play that, or combine a category with women.

A category is any attribute value held by at least 100 people, widened through Wikidata's class tree so that "actor" includes film actors and stage actors. The class tree goes up into abstractions quickly, so the first catalogue included categories like "emotion", "text" and "employee". The current rule keeps a widened category only if enough of its people hold it directly, with a hand-maintained list of exceptions. There are 7,798 categories.

## The game and the server

The game is a static page (TypeScript and Vite, no framework) on GitHub Pages. Game state lives in the browser. There's no leaderboard and no anti-cheat, since anyone could write a bot anyway and it's just for fun. When a name is ambiguous the game doesn't show you the candidates, because that would make it too easy. At the end you get a breakdown of who you named by occupation, country, birth year and fame, and a second tab with what everyone else named in the same game.

The lookup server runs on a Google Cloud e2-micro, which is in the free tier and has 1 GB of memory, behind Caddy. Lookups are rate limited per address.

Before deploying I ran a fuzz test that sent 800,000 malformed requests at every endpoint. Nothing crashed, but its list of slowest requests showed a real problem: a category lookup of a common first name ("John" among actors) checks up to 2,000 people with that name, and each check read a record from the 540 MB cold file. On the VM's slow disk that took 0.4 to 2.1 seconds. The fix was a separate memory-mapped file of just the attributes, plus a 24-bit hash of each person's own name in the hot record so the lookup can tell a name from an alias without touching the cold file. Those lookups now take 30 to 130 ms uncached.

## Testing the matching

I have a file of 869 names with expected results that runs against the full index. Six fail for known reasons. Of the 5,000 most-viewed women, 4,958 are accepted by their own name. The rest are ambiguous with someone of similar fame, like the two Jane Seymours.

## Not done yet

- Updating the index. Wikidata doesn't publish incremental JSON dumps, so for now the plan is to rebuild from a fresh full dump every month or few.
- Going through the miss log and the reports once real people have played it, which is the actual test of whether the lookup is good.
