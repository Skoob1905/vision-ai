# PatternPlay

_IMPROVE YOUR PATTERN PLAY, IMPROVE YOUR GAME_

Your AI Companion app designed to help you improve your pool game!

## Preface (2 min read)

### Problem

As a pool player for 20+ years, I've seen how long it takes in order to be 'good' at the game. It takes time and skill, you start thinking its all about potting, then you get to a level where you learn spin to manipulate the cue ball, but finally for me (the holy grail) its pattern play.

### The Idea

Finding the best pattern is the number 1 most important thing to do on a pool table. It makes every shot, the potting and the positioning part, a lot lot easier. Planning 3 maybe 4 or even more shots ahead will make, each shot easier on average. Always better to play 2 or 3 50/50 shots than taking one over the pocket and leaving a nasty finish.

### Artificial Intelligence

Without a doubt AI will rule the world and I'm on a mission to bring this into the world of Pool, english pool, American 9-Ball/10-Ball, Chinese pool, Billiards the list goes on .... We can produce models, study previous games, train an absolute demon to play pool and learn fast, all so you dont have to. Any decent pool player will know the importance of finding the patterns and only the masters will know how to do it the best way ... so lets make one!

### The Prospect

How many times have you broke off, knowing you need to find the pattern for the finish, and can't find it. What if .... you took a picture of the layout (on any shot mind you, not just the break off) and find out:

1. What is the best shot to take? For the best % finish?
2. What are the viable options? Maybe there's a couple routes?
3. What would the professional pool players do in this situation?

I believe these are all the things are all the things that can be answered

## Proposed Solution

Firstly we need some data.

### MVP (Minimum Viable Product)

Lets just start with something small to see what we can actually achieve, small training data; simple solution just to see if this can work.

### Training The Object Detection Model

I would like to start with some sort of Object Detection AI which will look at any given table layout and give us the following information?

1. Where are the pockets/cushions (pockets could be labelled Top-Left (TL), Bottom-Right (BR), Middle-Left (ML) etc. etc.)
2. Where are the balls on the table between shots

### Ball Locations

I want to store the (x,y) location of every ball on the table. All of the training data will be a picture taken at some elevated angle meaning the position on the screen is not the true position of the ball ... so all coordinates will need to be projected (a linear transformation should do the trick) and then we can track a real life representation of wtf is going on ... unfortunately we dont have overhead shot for training data, so we have to make do.

### Snapshots

A snapshot will be taken per shot i.e. shot 1 will have balls x, y, z at positions (x<sub>x</sub>,y<sub>x</sub>),
(x<sub>y</sub>,y<sub>y</sub>),
(x<sub>z</sub>,y<sub>z</sub>) .... then shot 2 with balls x, z (since ball y was potted) at (x<sub>x</sub>,y<sub>x</sub>),
(x<sub>z</sub>,y<sub>z</sub>). We are assuming straight run outs here so every shot will result in ball potted. Then after the run out, the AI will assess which order the balls were taken and store this for a future reference. Some maths magic has to be implemented here but I will document this later.

### Scenario / Data collection

Lets do a walkthrough of some basic scenario where we have 6 balls on the table

a = cue ball <br>
b = black ball<br>
c = red ball 1<br>
d = red ball 2<br>
e = yellow ball 1<br>
f = yellow ball 2

| shot # | x<sub>a</sub> | y<sub>a</sub> | x<sub>b</sub> | y<sub>b</sub> | x<sub>c</sub> | y<sub>c</sub> | x<sub>d</sub> | y<sub>d</sub> | x<sub>e</sub> | y<sub>e</sub> | x<sub>f</sub> | y<sub>f</sub> | pocket # |
| ------ | ------------- | ------------- | ------------- | ------------- | ------------- | ------------- | ------------- | ------------- | ------------- | ------------- | ------------- | ------------- | -------- |
| 1      | 10            | 4             | 34            | 1             | 94            | 96            | 47            | 34            | 92            | 38            | 51            | 28            | null     |
| 2      | 45            | 67            | 34            | 1             | 94            | 96            | 47            | 47            | null          | null          | 51            | 28            | TR       |
| 3      | 20            | 90            | 34            | 1             | 94            | 96            | 47            | 34            | null          | null          | null          | null          | BR       |
| 4      | 62            | 36            | null          | null          | 94            | 96            | 47            | 34            | null          | null          | null          | null          | MR       |

In this example above we have taken a snapshot (after each shot) wher each positional value will come from an object detection AI, and the player has potted there two yellows, followed by the black. Obvs this data is crude and as you can imagine, for 15+ balls on the table, this kind of chart will get very big very quickly ..... which makes me think we're going to need alot of training data!

so each row will contain 32 row items (2 row items per ball plus shot id + pocket ID used) and in a perfect run out of between 4-8 shots, depending on what balls were potted on the break, we will have a data collection size 32 x (between 4-8) so roughly 160 data points.

### What does this mean??

It's all well and good collecting data and making this chart but wtf comes next. Well ...

If you break off and take a picture of the table, the object detection AI will locate the balls and create something like this ...

| shot # | x<sub>a</sub> | y<sub>a</sub> | x<sub>b</sub> | y<sub>b</sub> | x<sub>c</sub> | y<sub>c</sub> | x<sub>d</sub> | y<sub>d</sub> | x<sub>e</sub> | y<sub>e</sub> | x<sub>f</sub> | y<sub>f</sub> | pocket # |
| ------ | ------------- | ------------- | ------------- | ------------- | ------------- | ------------- | ------------- | ------------- | ------------- | ------------- | ------------- | ------------- | -------- |
| 1      | 10            | 4             | 34            | 1             | 94            | 96            | 47            | 34            | 92            | 38            | 51            | 28            | null     |

We need to make some assumptions here:

1. **Did the user go in off?:** Lets assume no just so it easier
2. **Is it an open table?:** Lets assume, like in the scenario, the player is on yellows.

Now over to the AI..

It's job is to prefill the next row (i.e. shot), so that the best possible chance of a finish is possible. Namely:

1. Which is the next ball to go for (yellow 1 or 2)
2. Where is the Ideal place of the cue ball to

to summarise...
|shot #| x<sub>a</sub> | y<sub>a</sub> | x<sub>b</sub> | y<sub>b</sub> | x<sub>c</sub> | y<sub>c</sub> | x<sub>d</sub> | y<sub>d</sub> | x<sub>e</sub> | y<sub>e</sub> | x<sub>f</sub> | y<sub>f</sub> | pocket #
|-|-|-|-|-|-|-|-|-|-|-|-|-|-|
|1|10|4|34|1|94|96|47|34|92|38|51|28|null
|2|??|??|34|1|94|96|47|34|??|??|??|??|??
|3|??|??|34|1|94|96|47|34|??|??|??|??|??
|4|??|??|34|1|94|96|47|34|??|??|??|??|??

where each **??** is a value to filled in by AI. This also assuming no cannons were made, it may be the case the AI will override the values in that middle block to account for any cannons that might need to be played.

## Golden Ticket

If the AI can fill this grid in as perfectly as possible we have created out first AI Pattern for a finish!!

## Disclaimer

This is only the beginning, We have made ALOT of assumptions here which can be given by the user but some made for simplicity of this problem. This may be simple for a small sample size but with more balls, this can be very complicated and may take time a training to actually be able to deliver something which can be accepted as 'decent' by those who know their patterns like the back of there hand.
