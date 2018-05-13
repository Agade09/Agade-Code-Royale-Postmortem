# Agade Code 4 Life Postmortem

I will describe my AI which finished 3rd on the Code Royale competition on Codingame.

## Introduction

My AI decides heuristically on a higher level which site to go to and on a more micro-management level decides how to get there. I pretty much hardcoded the opening building order because I saw in many games that the opening was critical (and I didn't find a better way than hardcode). If your opponent gets the upper hand in the opening he will grab all the territory and probably win. My goal is to build 2 mines and a barracks, send a wave of knights to the enemy thanks to the starting 100 gold and then start building towers to defend against his incoming knights. Afterwards I pretty much only build towers and try to grab as much territory as possible.

I believed a balanced AI, which does not only build towers, but uses some mines and barracks to place additional pressure on the opponent was optimal but I did not manage to achieve it. Building towers is simple and safe, when you build a mine, you have to consider if it won't be destroyed by enemy creeps and when you don't build towers you have to worry about the opponent possibly gaining territory.

In effect I won my games in two ways: getting in some initial damage to the enemy queen in the opening and then "drawing out" the game with a lot of towers (although I didn't implement any extra hiding if I had an HP advantage), or sometimes grabbing so much territory that the enemy had nothing left (i.e: building towers in the enemy base if any opportunity arose).

## Site selection

Sites were sorted by the following criteria, in order of decreasing importance:
* `Dist(Tower,site)<Tower_Range-Site_Radius-Queen_Radius` Must be able to touch the site without being in enemy tower range. I should have subtracted the `Touching_Delta` as well.
* Is the site claimable by my queen. Either a neutral site, or an enemy mine or barracks.
* `-Dist(Queen,site)-Dist_To_Me(Site)-Dist_To_Enemy(Site)` a scoring of claimable sites. Distance to a player was defined as distance to the queen + distance to each building divided by the number of buildings + 1, so the average distance to buildings and the queen. This guided my AI towards the frontline whilst taking sites along the way (low distance to my buildings). I wanted my AI to be "territorially aggressive". One flip side I noticed was my AI was regularly hanging around the frontline in the mid/late game, achieving nothing apart from being closer to the enemy barracks, when it would have made more sense to go back and swap some towers for mines. Or at the very least not hang around barracks producing knights and end up losing HP which happened too often. There was also an issue of targeting "offside" sites that weren't covered by towers but that couldn't be reached without going through towers.

On Monday morning I added a hack to change `-Dist_To_Enemy(Site)` to `+Dist_To_Enemy(Site)` in the early game for the first wave of knights because I was sick of losing to my AI simply being too aggressive and losing HP in the early game. This gained a lot of trueskill points.

According to this sorting the top 2 sites were fed as targets to my search algorithm which handled the pathfinding. 

If I was touching a site I had some conditions on what to build, therefore not calling the search algorithm. But very often this was overridden by a "Knight danger" criterion, and since my search algorithm only builds towers that is mostly what my AI did. For reference I had these formulas for site building scores:
* `11*(Max_Dist-Dist_To_Enemy(Site))/Max_Dist` as a tower score. The closer a site is to the enemy to more you may want to build a tower there. Max_Dist is the diagonal span of the map.
* `(My_Barracks==0?12:0)*(Max_Dist-Dist_To_Enemy(Site))/Max_Dist` as a barracks score. I try to have 1 barrack, with a score slightly higher than a tower to give it priority.
* `maxMineSize+1e-2*gold+2*(S.Dist_To_Enemy(Site)-Max_Dist)/Max_Dist` as a mine score.
These formulas were overridden to infinity with some ifs for the opening of the game. I try to build 2 mines, but will switch to a barrack earlier if the enemy has already started producing his knights and then I prepare my defenses with towers.

## Dancing queen

Being able to kite effectively is essential in this game. I really underestimated it in the beginning of the contest but there is a lot one can do to safely and expediently get rid of enemy knights. One can delay enemy knights by making them hug sites, and once they get to your base you can have them hug your towers to take maximum damage whilst avoiding a lot (or all) of it yourself.
At the higher levels a lot of games were won simply by having 1 more HP than the opponent and beyond the HP advantage I also noticed games could be won/lost simply by poor kiting. Indeed, if you spend 20 turns running around the map to get rid of the opponent's first wave of knights while he only spends 10, he can grab a lot of territory and you are in a losing position.

I had an approximate simulation with a depth 7 simulated annealing non-exhaustive search.

### Simulation

I initially and naively thought the simulation didn't need to be accurate at all and/or that accurate simulation would be too slow anyway. So I was first in the beginning of the contest with a very approximate simulation (creeps going through towers kind of approximate). Throughout the contest I gradually improved my simulation, leading to large elo gains just from better kiting. You need to model knight-site collisions quite well to nicely slow them down against sites, you need knight-knight collisions so that your AI isn't deluded in thinking that it can block 20 knights against 1 tower (they collide with each other and spread out)... In retrospect it would have been a lot more efficient to make a decent simulation from the start instead of patching it for a week.

I could run about 5000 simulations in the most computationally intensive situations and about twice as much in more standard situations with lower amounts of knights.

Because my simulation was so inaccurate I added "pessimism" epsilons, e.g: Knights could hit me from 5 units further than their true range so that I wouldn't lose HP to 1 pixel errors. I had a similar epsilon for the damage caused by my towers (knights were 3 units further for the damage calculation) and for the distance of my queen to enemy towers (so I wouldn't step in by 1 pixel by mistake and lose 1HP).

To my surprise, despite all the simulation errors I had, due to simplifying the game engine, my biggest issue was what Robostac discussed in his postmortem: when knights hug a tower, it comes down to rounding errors to find out which knight will get hit by the tower. This caused me to lose a lot of games by getting hit by a knight I expected would be dead. I think this was a serious oversight from the CC3 team: rounding errors should not be part of the rules of the game. This could have been avoided for example by comparing distance to the tower with an epsilon and tie breaking on unit ID.

### Move generation

I guided the search by producing moves in the following way:
* 35% chance to do a "hug move" (20%) or a "tangent move" (15%)
* Tangent move is going around the nearest site by targeting one of two intersection points of a circle of radius `60` (queen speed) around your queen and the site. Tangent move is only tried if the queen is within `Site_Radius+Queen_Speed` of the site (intersection points exist). If tangent move is not possible then hug move is performed instead. I used [this](https://stackoverflow.com/questions/3349125/circle-circle-intersection-points) stackoverflow topic for the intersection of two circles.
* Hug move is moving towards the nearest site in a straight line if not touching it. If touching it, it is building/upgrading a tower. I don't allow my AI to build towers on my mines, my AI would therefore sometimes let enemy creeps destroy a mine to replace it with a tower.
* 65% chance to make a move in a random direction, with a speed of `60` 85% of the time and a random speed 15% of the time.

### Eval
* `-1000` if queen is dead.
* `+Queen.HP`
* `-5e-3*Dist(Queen,target)`
* `-5e-4*Dist(Queen,next_target)`
* `-10*max(0,Knight.HP/Dist(Queen,Knight)-0.02)` for each enemy knight. I'm not sure that this form is optimal because I hardly tested anything. But my intuition was that enemy knight HP is bad in a linear way because a knight with twice as much HP will probably be able to hit my queen twice as much if in range. In a similar way knight distance is good in a linear way because a knight twice as far will probably have half the HP by the time he gets to my queen. The "relu (rectified linear unit) nonlinearity" (the max) is to take into account that a knight sufficiently far away, with low enough HP, poses no threat. Here the `0.02` coefficient should be `>0.01` because a knight loses at least 1 HP per 100 units traveled (knight's speed is 100 and HP decay is 1 per turn).
* `-1e-2*HP` for enemy archers and giants.
* `0.25+5e-3*HP` per tower. That is 4.25 queen HP for a fully upgraded tower. This feels way too high.
* `Fully_Upgraded_Tower_Value+0.01` per barrack.
* `0.1*Mine_Level` per mine. This should probably have been higher compared to that high tower value.

I evaluated as `Eval(Turn_0)+0.95*Eval(Turn_1)+0.95^2*Eval(Turn_2)+...` until Monday morning. I believe this was causing some of the issues I noticed when trying to make a balanced AI: my AI seemed not to want to replace buildings. I think this was due to the fact that in this scoring system the last state is valued the least, but since playing this game is almost like PVE what we care most about in the search is the state on the last turn. So I arbitrarily evaluated a solution as `Eval(Turn_0)+0.95*Eval(Turn_1)+...+7*Eval(Turn_6)`. This values the final state the most but preserves the desire of the AI to make things happen sooner than later. I quickly batched this in with other changes on Monday morning so I don't know how much it helped.

## Unit Training

I train 1 wave of knights if I have the gold. I experimented briefly with bunching waves of knights but didn't pursue.

## Conclusion

I finished with ~800 lines. I wasn't highly motivated for this contest for some reason, so I didn't make a local referee and I hardly did any tests to compare my engine to the referee... Looking back I regret being so sloppy. I didn't spend as much time on this contest as I did on some others in the past, but I feel like all that time I did spend was used quite inefficiently, which is just a waste. I'm not very proud of this AI (tower spamming, losing alot of HP by playing daredevil in front of the enemy barracks while they were producing,...). I'm also not particularly delighted about finishing 3rd by 0.02 trueskill points but I'm grateful to make another podium. Congratulations to Robostac for making a nice AI.
