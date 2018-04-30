# cg-code-royale-postmortem
#### Overview

Thanks to csj, harshch000 and Netbattler11. I really enjoyed this contest as there was a good mixture of strategies, no obvious way to solve it and optimising strategy seemed more important than optimising performance. Managing to achieve 1st place in a full contest certainly helped as well.

The kotlin referee was interesting to see as I had no previous experience with the language. I had to lookup some of the more unusual features (eg the 'it' variable), but liked the overall structure. 

Overall I think the balance was fairly good (with the exception of archers who never became relevant). The length of the game was a large factor in this - due to the time taken to get giants online, a large number of games finished with me either just winning in the last few turns or being a few turns away from getting through the enemy defenses. I tried different tactics to get giants online earlier, but never successfully managed.

I spent the first weekend playing around with a basic bot to work out what was / wasn't working well and to think about possible implementations - this bot was on the edge of the top 100 when silver opened. 

From this I came to a few conclusions
- Giants / Archers seemed very poor, so my initial plan was to focus on getting waves of knights to overwhelm enemy towers. Giants actually turned out to be fairly strong once people started to dodge knights properly.
- Enemy prediction isn't very important, as the enemy queen cannot do a lot to interfere (which means I probably don't want to minimax). Units take a long time to build, so you have plenty of warning of trouble. The only thing that could really go wrong would be an aggressive tower rush, but luckily this was entirely countered by building knights.
- As enemy knight movement is entirely based on my queen, the simulation of whats happening around my queen should be almost perfect as it doesn't depend on enemy actions.
- Nothing happens particularly fast, making search depth very important (this meant I didn't really want to do a GA / MC solution , as it would need a lot of simulations to converge).

Once I'd got my simulation and initial strategy running I tested a few games against the current #1 (nagatwin) and managed to win after a couple of minor tweaks so I submitted. This version ended up 5th. I fixed a bug that made my avoidance much better (tower damage was very wrong) and after submitting that I was fighting for first. At this point I'd only actually tested against one opponent and investigated who was beating me. I found I had a 0% winrate against eulerscheZahl's early aggression. After a few tweaks to counter that my submit ended first with a large score lead over everyone else. I then held first for the majority of the remaining time - Agade managed to take it on a couple of occasions but I quickly adapted my strategy to deal with his changes. To test new versions I ran a lot of CGBenchmark tests against Agade, Shingy, Risus and Xyze as these were my biggest threats / counters at various points. 


#### Simulation

I decided to do a straightforward port of the simulation code as I still wasn't sure how best to use a simulation for this contest  and wanted to get something working quickly to test with. I tried to keep it as close as possible but making allowances for performance where possible (always using squared distances until required, removing site->site collisions as I wasn't generating maps). 

A big performance increase came from storing the closest site to every possible integer x / y co-ordinate. This was then used in collisions to only check one site per unit, in my scoring to work out how close to a site I was, at the start of each turn to calculate if I was touching sites and probably in other places I've forgotton. It actually turned out to be incorrect for some maps where you could collide with multiple sites, but this was so rare I decided not to worry about it. 

As the game is very horizontal (wider and players always start at opposite ends) in the collision checks I check the distance on the x axis before checking the true distance which gave a decent performance increase.

The biggest issue I had with the simulation was choosing the correct knight for a tower. Towers target the closest unit, but tower collision is the last thing done before target selection and pushes units out to the exact same distance. This meant that the order was actually chosen based on errors in the floating point calculations, so in order to correctly simulate this the order of operations had to be the same as the referee. I spend quite a while re-ordering these calculations to match a couple of games where I found my simulation saying one creep would die but in fact it lived to attack my queen. As many games come down to single points of damage difference this was quite important.

I did think about trying to use AVX to speed up some of the simulation, but didn't want to commit that much time to performance as it didn't seem to be holding me back. 

I didn't simulate archers or giants until Friday as I didn't think they would be used. Magus was posting replays of giants rampaging over my base causing me to add giants to my simulation. While I didn't often see them used against me, I used them to counter heavy tower defenders. At the same time I also added archer simulation but never really tested or used it as I couldn't see them working.

#### Algorithm

I had two entirely separate algorithms, one for each of the output lines. There wasn't any link between the two. If my queen was going to build a barracks the training code wouldn't know about it. This also meant my queen could try to destroy a barracks just as it started training, but this happened rarely enough to be a non-issue.

#### Training

My code for training was a simple set of conditions
- For the first 50 turns just send knights from the closest barracks to the enemy queen whenever possible. This would often get an initial health advantage or at least cause the enemy queen to go into a defensive mode and not spread quickly across the map.
- For the last 40 turns spend gold as fast as possible - for every idle barracks if I can afford to train then train.
- If I don't have a giant but have a giant barracks then save up 220 gold. Start training a giant and then 8 turns later start training knights.
- If I have a giant them send knights as fast as possible from the closest barracks to the enemy queen. Train extra giants if I have the gold.
- If I don't have a giant and don't have a giant barracks then save up 200 gold and train as many knights as possible until I have less than 80 gold at which point I'd start saving again.


#### Queen 

My queen actions were decided using a beam search (breadth first search where only a limited number of the best actions are expanded each turn) capped to 15 depth (this was originally uncapped, but led to some very bad decisions when it was only queen vs queen). If there were 10 or less units I would take the top 50 moves at each turn, otherwise I'd take the top 25. This meant I only needed around 8000 sims to get to my maximum depth (my bot would very rarely use the full turn time during silver / gold - my eval caused me to need more sims later).  

At each depth the top nodes from the previous turn would each be expanded into 8 new nodes (with my queen moving 60 in a different compass direction for each) and 3 extra nodes created if the queen was touching a site that could be built on (build barracks, build mine (if gold available), build tower).

If the enemy queen was touching a tower it would be upgraded, otherwise they would stand still. This helped prevent damage against some bots who would aggressivly upgrade towers.

Unknown mines in my sim were assumed to have a maxMineSize of 1. I didn't do any tracking of enemy mine sizes, expected gold or anything with site symmetry.

My default strategy was to try and win with large numbers of knights, building an early barracks and then an equal number of towers and mines. I allowed a second barracks to be built if it was much closer to the enemy (and the first barracks would then be ignored in my scoring, allowing me to build on top of it if I got close). On any turn after 50 I could switch to an alternate strategy involving giants. This originally occured if the enemy had 0 barracks or 0 mines and later it would switch whenever I was losing or drawing. If the enemy had 0 mines I removed any tower requirements from my evaluation in order to maximise mine income. 

I started adding the giants strategy on Friday in response to a lot of losses to tower bots and then as I got it working better and better I used it more and more. I have never built an archer barracks or an archer. 

I didn't specify which type of barracks was to be built, just returned to build a barracks. The choice was made based on the current game state (this led to a really annoying to track down bug where my queen would keep swapping a giant barracks to a tower and back again due to a scoring error that I finally fixed on Sunday evening). This choice was controlled by whether I'd triggered my giant strategy or not - if I had and had no giant barracks it would be a giant barracks, otherwise it was a knight barracks.


#### Eval 

For each node there was an ongoing score built up each turn based on my queens actions, with a final score added based on an evaluation of the game state.

Each Turn:
- Destroying a Building would subtract from my score. This was to stop issues where my bot would change a structure multiple times to end up with the structure required. 

Once I'd simulated 8 turns I added the score at this point to my total, so the final score would be built up score + turn 8 score + final turn score. This was to prioritise good early moves.

Final Evaluation:
- First run two turns of simulation (with both queens doing nothing). This helped avoid moves that were just delaying (eg kiting knights round an empty site). This was added on Saturday and made my games take much longer (as I now hit the timelimit on all but the simplest turns). This is also when I added the half beam width for more complicated games.

- My Queens hitpoints. This was the overall score of the game and weighted very heavily to override almost everything else. 
- Enemy Queens hitpoints / 10. The enemy hp was divided by 10 to encourage aggressive tower placement but discourage trading hp (as I wasn't predicting the enemy I'd usually lose out as they'd move and I'd still take damage). This division was added later after seeing a lot of games lost by 1 hp due to failed trades. As I'm writing this up I've realised this was an integer division and probably explains some behaviour that didn't make sense (my queen would have guarenteed tower damage on the enemy but build a mine instead).

- If I had 0 or more than 2 (3 if I'd triggered my giant strategy) barracks I had a large negative penalty to my score.
- A bonus for how close the closest knight barracks was to the enemy queen.
- A negative value for not having a giant barracks if I'd triggered the giant strategy

- If I had less than 5 towers, more than 2 mines and less towers than mines I added a large negative penalty. This was disabled if the enemy had no mines after turn 50. 
- A bonus for every point of income from mines. 
- Each tower would give a bonus (roughly equivalent to a 2 income mine) that was scaled based on missing HP squared (this priortised partially upgrading towers).

- A negative value for each enemy building (prioritise destruction over neutral)

- Distance to closest buildable site (neutral site / enemy barracks / enemy mine).
- Distance on x axis to center of map 

- A completely random value (equivalent to up to 100 distance) to break ties and make my bot non deterministic (sorry!) as I didn't want to be too easily predictable. 

#### Data Structure / Code implemenation notes

During turn 1:
- Calculate the closest site to every possible integer x/y co-ordinate for use in collisions / scoring.
- For each site calculate the 800 possible ranges for a tower built on that site.


I had 4 main classes for game state, based on the 3 input loops with a global game class to contain a single instance.
- Units were very simple, having a point, type, health and owner. I converted the types to all be positive integers to allow looking up mass / radius / movement speed in global constant arrays based on the type.
- A global array of SiteInfo that each had a point, radius, maxMineSize, and an array of 800 values for all possible values of attackRadius. This information was global to all possible simulations and I didn't want to have multiple copies of this. 
- A instance specific site class that contained structureType, owner, goldRemaining, param1 and param2.
- My game class had an array of 24 sites, an array of 50 units, a unit count and the indexes in the array of my queen and the enemy queen as they were needed so often. Spawned units got added at the end as I didn't want to move queens around in the array, which would make collisions slightly incorrect (but unless I was near the enemy barracks probably not important). 


#### CGBenchmark

https://github.com/s-vivien/CGBenchmark
Massive thanks to Neumann for this tool, it helped a lot in optimising my strategy. 



