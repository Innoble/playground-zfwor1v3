# X-mas Rush Post Mortem

This contest was a lot of fun for me. Also very intense because of the close fight between number 1 and number 2. I ended at rank 2, which is a personal record for me. Very happy with the result. Congratulations to Jolindien for a decisive win against me (and everyone else). Thanks to the creators of the game, who did a very good job.

In this post mortem I will try to explain the main things I think I did right and also the things that didn't work for me.

## Contents

+ #### Simulation
+ #### Pathfinding
+ #### Search
+ #### Eval
+ #### Optimization and other stuff

## Simulation

This contest was hard for beginners to get into. This is mainly because of the necessity of a push simulation. Push is complex in that you need to take into account pushes by both players on 7 possible different rows and columns. The difficulty depends greatly on your choice of board representation. I decided to go with a "bitboard". This is a difficult to handle board representation that is very fast.

Bitboards can be efficient because they can be quickly modified using bit operators. My board was an int[9]. The first 7 elements were 7 map rows, the 8th integer was my hand tile and the 9th integer was my opponents hand tile. One of those integer map rows could look like this in terms of individual bits:  1001 0111 1110 1111 0011 1100 1010 
.Basically I used the map tiles we are given as input and only convert them to bits. You can do this as follows:

```C++

int StringToBits(string s)
{
	int binary = 0;

	for (int i = 0; i < 4; i++)
	{
		if (s[i] == '1')
			binary += 1 << (3 - i);
	}
	return binary;
}
```

For quests and players I used a similar representation. For players the first 4 bits were the x -coordinate (could have used 3, reasons below). The next 3 bits were the y-coordinate. I use a player[2] array. I don't track items, only quests. In hindsight I missed an opportunity due to this lack of item tracking, more on that later. For quests I used the same coordinate representation, only I put the item id on bits 7 to 10 and the tile code (1100 etc.) on bits 11 to 14. In this way, everything is represented in the form of integers.


My Push code you can find below. I show about half of it because it is a big method, but it should be easy to extend to include all possibilities. One important note: I don't use the -1 and -2 given for hand items. My x becomes 7 for my hand and 8 for opponent hand and 9 for disabled (empty slots or completed) quests. This is also why x needs 4 bits. 

    

::: Push Method (click to uncollapse)

```C++

void PushMap() 
{
	if (nodeId[0] == nodeId[1] && rowCol[0] == rowCol[1])
		return; // This is to stop the method when deadlocked

	for (int player = 0; player < 2; player++) // two player pushes
	{
		if (rowCol[player]) // is it a row push? Or a column push
		{
			int id = nodeId[player]; // which row is being pushed?
			int mapRow = mapCopy[id]; // the map row that is being pushed
			int handTile = mapCopy[7 + player]; // the hand tile of this player
			if (leftUp[player])  // is it pushed left?
			{
				mapCopy[7 + player] = (mapRow >> 24) & 15;  
// the left most tile is pushed off and placed in hand.
				mapCopy[id] = (mapRow << 4) | handTile; 
// the map row is shifted 4 bits (1 tile) to the left and the handtile is added

				for (int i = 0; i < 12; i++) 
				{
// loop over all tracked quests (I track 6 per player, 3 current and 3 next if known
				
					int quest = questsCopy[i]; 

					if (quest == 9) 
						continue; 
// the quest is 9 if no quest exists in this slot. 	
// I use this 9 because 0 to 6 are x coordinates in the row, 7 is my handtile and 8 is the opponents.
					int x = quest & 15; // take the right 4 bits for x

					if (x == 7 + player) 
// if the quest is in my hand, place it on the board.
					{
						quest &= ~127; 
// clear the x and y coordinates. ~127 means NOT 127 setting the first 7 bits to 0.
						quest |= 6 | (id << 4); 
// OR with 6 (the right most x coordinate and with y shifted 4 to the left.
						questsCopy[i] = quest; 
// set to array
					}
					else if (x < 7) // the quest is on the board
					{
						int y = (quest >> 4) & 7; // get the y coordinate
						if (y == id) 
// is the y coordinate equal to the row being pushed?
						{
							x--; 
// reduce the x coordinate because of the push to the left
							quest &= (~127); // clear coordinates;
							if (x < 0) 
// pushed off the board? Then take in hand
								quest |= (7 + player);
							else
								quest |= x | (y << 4); 
// otherwise set the new coordinates
							questsCopy[i] = quest;
						}
					}
				}

				for (int i = 0; i < 2; i++)
				{
					int y = playerPosCopy[i] >> 4; 
// get player y by shifting 4 to the right

					if (y == id) 
// is the player on the row being pushed?
					{
						int x = playerPosCopy[i] & 7; // get the x 
						x--;  

						if (x < 0) 
// x smaller than 0? Then set to the other side of the board.
							x = 6;
						playerPosCopy[i] = x | (id << 4);
					}
				}
			}
			else
			{
			// similar for the other direction.
			}
		}
	}
	
	  // At this point all row pushes are done. Columns are next

	for (int player = 0; player < 2; player++)
	{
		if (!rowCol[player]) // this is a column being pushed
		{
			int id = nodeId[player];
			int handTile = mapCopy[7 + player];
			int shift = (6 - id) * 4; 
// The amount a row has to be shifted to expose the pushed column tiles
			int clear = ~(15 << shift); 
// This is used to clear a map row only in the column in the tile that is being pushed.
			if (leftUp[player]) // push the column up?
			{
				mapCopy[7 + player] = (mapCopy[0] >> shift) & 15; 
// take the tile at the top and put it in the player hand

				for (int i = 1; i < 7; i++) 
				{
					int tile = (mapCopy[i] >> shift) & 15;
					mapCopy[i - 1] &= clear;
					mapCopy[i - 1] |= (tile << shift);
				} 
// make the tile in the correct column equal to tile in the row below
                
				mapCopy[6] &= clear;
				mapCopy[6] |= (handTile << shift);
// the lowest mapTile is set equal to the handTile that is pushed in
				for (int i = 0; i < 12; i++)
				{
					int quest = questsCopy[i];
        
					if (quest == 9)
						continue;
					int x = quest & 15;
            
					if (x == 7 + player)
					{
						quest &= (~127); // clear coordinates;
						quest |= id | (6 << 4);
						questsCopy[i] = quest;
					}
					else if (x < 7)
					{
						if (x == id)
						{
							int y = (quest >> 4) & 7;
							y--;
							quest &= (~127); // clear coordinates;
							if (y < 0)
								quest |= (7 + player);
							else
								quest |= x | (y << 4);
							questsCopy[i] = quest;
						}
					}
				}

				for (int i = 0; i < 2; i++)
				{
					int x = playerPosCopy[i] & 7;

					if (x == id)
					{
						int y = playerPosCopy[i] >> 4;
						y--;

						if (y < 0)
							y = 6;
						playerPosCopy[i] = id | (y << 4); 
// set the new player position
					}
				}
			}
			else
			{
// similar for the other direction
			}
		}
	}
}

```

:::




 Because it is such a big piece of code, it may seem slow, but most of it is ignored during a run because of the branching. I could have thought more about condensing it to remove (nearly) duplicate code, but it works and it would not become faster by condensing it. I unit tested the method by printing the map before and after push. To do this I used:

```C++

string printCodes[]
{
	"", "", "", "\u2557", "", "\u2550", "\u2554", "\u2566", "", "\u255D", "\u2551", "\u2563", "\u255A", "\u2569", "\u2560", "\u256C"
};

```

This outputs a pretty map picture when using  array lookups with those strings converted to bits the way I showed above.

## Pathfinding

I have 3 pathfinding algorithms, two of which are very fast. They all assume you try to get the maximum amount of items. But the greedy one also makes assumptions about the order in which you do it (closest first). 

I use them as follows:

+ Slow BFS with classes (pathnodes) to run only once, on move turn. I create the first layer of nodes in my search tree this way. I kept this with classes because it was easier and because changing it wouldnt help me. It needed to be different, because I needed to backtrack through the nodes to generate the movement list (strings to output). Therefor I need to store a parent reference.  
+ Fast (non-greedy) BFS for the first move layer of my push turn. I cache all possible outcomes with the maximum amount of items gathered. Basically you get one move per reachable tile this way. This BFS is classless, using only integer arrays. 
+ Even faster greedy BFS. This assumes a player will get the closest items first and then the rest. In 99% of cases this will net the same result as the above version. I used this for the deeper search layers (move 2 and 3). 

More information on optimizing BFS can be found here: https://tech.io/playgrounds/38626/optimizing-breadth-first-search


## Search

As people assumed, I used "my own" [(turns out it already existed)](https://www.researchgate.net/publication/224396568_CadiaPlayer_A_Simulation-Based_General_Game_Player) algorithm [Smitsimax](https://tech.io/playgrounds/36476/smitsimax). I didn't want to let people know I used this. It is one thing to give away a tool, but another to tell people when to use it. If people knew what I used to get to rank 1 during the contest, they would switch to it much more quickly and catch up more easily. If I had to risk failure by choosing this algorithm, then so should everyone else. Choosing your strategy is part of the challenge. If people already know what is good, then there is no risk to it. For a long time I was not sure I went the right way using this algorithm. I was "stuck" at around rank 10 for a day or two and then I fixed a bug that shot me up to rank 1 by a wide margin. That's when I was sure. 

### Problems with Smitsimax

As others mentioned, the moves in a search are tricky. Pushes are always free choices, not constrained by the opponents' actions. Moves depend on the state of the board. At first I solved this by making only push-nodes and doing my greedy bfs to determine the move options. I picked the best move option by eval. The eval was a very fast lookup created by a key consisting of quest item position, player position and tile orientations. It worked well. Much later I found (to my regret about time lost coding) a random move selection worked even better. So in the end, my moves (excepting the output move in the move turn) were all random. 

I noticed however, that moves inside the search are somehow very unimportant compared to pushes, assuming you always collect the maximum amount of items available. When I first got to rank 1 halfway through the contest, I didn't even have a search on my move turn. I just did a single pathfinding run. My push search was pretty deep and this mattered more. 

Later on I found a way to generate ALL moves for ALL pushes. During the search for the pushturn I could select any combination of depth 1 pushes and use lookups to get all pathfinding endresults. Generating this move cache took less than a millisecond. You would think it would be best to use this to get a Move-node layer after the first push layer and use full smitsimax during the first 3 phases (push-move-push). However, when I did this, my bot performed worse. I unit tested a lot, because I could not believe it, but it really was true. Even with all moves cached it was better to do Push -> Random Cached Move -> Push -> Random greedy move -> Push -> Random greedy move. So no statistics were gathered on move nodes. I think the reason that it is not good to use the move nodes, is that there is often not much difference between the quality of tiles you can end on. It is much more important to just get the maximum amount of items. Sure some tiles will be better to land on, but with such a large branching factor, you're not likely to converge on this difference anyhow. Much better to focus on push combinations that lead to more items found. Thats my theory anyway. Hard to prove.

#### Summarized

+ Move turn: BFS generated move nodes first layer -> Push with random greedy move -> Push with random greedy move
+ Push turn: Push and cached moves (no pathfinding during 1st layer thanks to cache) -> Push with random greedy -> Push with random greedy

## Eval

As I wrote above, I tried an evaluation based on quest item location and end-of-move location that also included the orientations of the tiles (entrances pointing at eachother = better). This worked well, but to my regret I found out that random moves are better. I also tried distance to edge (both ways, penalized and as a bonus). This didn't help. I added a bonus for item in hand. Not sure if it helped, it's in the final version though. I tried an idea of eulerscheZahl to try to favor landing on random items that are not quest items after item collection. You could get lucky this way and immediately get another item. This seemed a no brainer. I tried it and it failed for me. Only later on, to my embarassment, did I realize this was because I do not track the location of non-quest items during push. Then it makes no sense to do this. I would get non-updated locations that are meaningless. It was too late to add this properly, but it would not have been an easy change anyway. I would have had to start tracking all items again (doubling the size of my quest array). 

    So in the end I scored 3 things:
    
    + Quests completed: 50 points (could be any value)
    + Quest item in hand: 10 points (not really fitted, just has to be smaller than quest completed)
    + Win bonus: 300 points. I can't just make this infinite. That would screw up the smitsimax. I just made it high enough to favor the move. I didn't fit this, no time.


## Optimization and other stuff

I practiced conversion from C# to C++ a lot on multiplayer arenas and was able to do it in 4 hrs of coding + 3 hrs finding 1 bug. That's a 1900 line bot. Very proud of that. My sim count (defined as 1 push + 1 move) went from 10-30k to 100k as a result of that conversion. Even before conversion, I had quite a lot of sims in C#. I think this is mainly because of the fast BFS, which is the bottleneck. Other than that, I did what I always do. Everything is in global arrays and I don't generate any garbage to clean up. 

I coded a deadlock breaker like most of us did. My way was to check the map every turn and if the map was equal to the map in the last turn, I would increment a deadlock count. If it reached 5, I would disallow my last row/col + id choice and assume the opponent only made that choice. That helped with convergence of the search. In some cases you will actually pick a nice counter move. I would always do this when I was losing, but I also would do it if the pointcount was less than 9 and I was tied for points. That way I got very few draws. This was important, because I needed the points from beating players other than Jolindien. It worked quite well (not well enough obviously!).

As you can see: I share a lot. If you have advice to make my botting better, I would much appreciate. There are always things to improve. Maybe next time I can get one rank higher with your help?




