# X-mas Rush Post Mortem

This contest was a lot of fun for me. Also very intense because of the close fight between number 1 and number 2. I ended at rank 2, which is a personal record for me. Very happy with the result. In this post mortem I will try to explain the main things I think I did right and also the things that didn't work for me.

## Contents

+ ### Simulation
+ ### Pathfinding
+ ### Search
+ ### Optimization and other stuff

## Simulation

This contest was hard for beginners to get into. This is mainly because of the necessity of a push simulation. Push is complex in that you need to take into account pushes by both players on 7 possible different rows and columns. The difficulty depends greatly on your choice of board representation. I decided to go with a "bitboard". This is a difficult to handle board representation that is very fast.

Bitboards can be efficiënt because they use less memory and be quickly modified using bit operator. My board was an int[9]. The first 7 elements were 7 map rows, the 8th integer was my hand tile and the 9th integer was my opponents hand tile. One of those integer map rows could look like this in terms of individual bits:  1001 0111 1110 1111 0011 1100 1010 
Basically I used the map tiles we are given as input and only convert them to bits. You can do this as follows:

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

For quests and players I used a similar representation. For players the first 4 bits where the x -coordinate (could have used 3, reasons below). The next 3 bits where the y-coordinate. I use a player[2] array. I don't track items, only quests. In hindsight I missed an opportunity due to this lack of item tracking, more on that later. For quests I used the same coordinate representation, only I put the item id on bits 7 to 10 and the tile code (1100 etc.) on bits 11 to 14. In this way, everything is represented in integers, which makes sure I can do a fast push method.


My Push code you can find below. I show about half of it because it is a big method, but it should be easy to extend to include all possibilities.


```C++

void PushMap() //true = column, true = up or left
{
	if (nodeId[0] == nodeId[1] && rowCol[0] == rowCol[1])
		return; // This is to stop the method when deadlocked

	for (int player = 0; player < 2; player++) // two player pushes
	{
		if (rowCol[player]) // is it a row push?
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

					if (x == 7 + player) // if the quest is in my hand, place it on the board.
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
						if (y == id) // is the y coordinate equal to the row being pushed?
						{
							x--; // reduce the x coordinate because of the push to the left
							quest &= (~127); // clear coordinates;
							if (x < 0) // pushed off the board? Then take in hand
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

					if (y == id) // is the player on the row being pushed?
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
				} // make the tile in the correct column equal to tile in the row below
                
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
						playerPosCopy[i] = id | (y << 4); // set the new player position
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

 Because it is such a big piece of code, it may seem slow, but most of it is ignored during a run because of the branching. I could have thought more about condensing it to remove similar code, but it works and it would not become faster by condensing it. I unit tested the method by printing the map before and after push. To do this I used:

```C++

string printCodes[]
{
	"", "", "", "\u2557", "", "\u2550", "\u2554", "\u2566", "", "\u255D", "\u2551", "\u2563", "\u255A", "\u2569", "\u2560", "\u256C"
};

```

This outputs a pretty map picture when using lookups with those strings converted to bits I showed above.

## Pathfinding

I have 3 pathfinding algorithms, two of which are very fast. They all assume you try to get the maximum amount of items. But the greedy one also makes assumptions about the order in which you do it. 

I use them as follows:

+ Slow BFS with classes (pathnodes) to run only once, on move turn. I create the first layer of nodes in my search tree this way. I kept this with classes because it was easier and because changing it wouldnt help me. It needed to be different, because I needed to backtrack through the nodes to generate the movement list (strings to output). 
+ Fast (non-greedy) BFS for the first move layer of my push turn. I cache all possible outcomes with the maximum amount of items gathered. Basically you get one move per reachable tile this way, usually many. This BFS is classless, using only with integer arrays. 
+ Even faster greedy BFS. This assumes a player will get the closest items first and then the rest. In 99% of cases this will net the same result as the above version. I used this for the deeper search layers. 

I will explain the second version in a separate article because it is much more general, being usable for different multi's and contests.


## Search

As people assumed, I used my own Smitsimax algorithm. I didn't want to let people know I used this. It is one thing to give away a tool, but another to tell people when to use it. If people knew what I used to get to rank 1 during contest, they would switch to it much more quickly and catch up more easily. If I had to risk failure by choosing this algorithm, then so should everyone else. Choosing your strategy is part of the challenge. If people already know what is good, then there is no risk to it. For a long time I was not sure I went the right way using this algorithm. I was "stuck" at around rank 10 for a day or two and then I fixed a bug that shot me up to rank 1 by a wide margin. That's when I was sure. 

### Problems with Smitsimax

As others mentioned, the moves in a search are tricky. Pushes are always free choices, not constrained by the opponents' actions. Moves depend on the state of the board. At first I solved this by making only push-nodes and doing my greedy bfs to determine the move options. I picked the best move option by eval. The eval was a very fast lookup created by a key consisting of quest item position, player position and tile orientations. It worked well. Much later I found (to my regret about time lost coding) a random move selection worked even better. So in the end, my moves (excepting the output move in the move turn) were all random. 

I noticed however, that moves inside the search are somehow very unimportant compared to pushes, assuming you always collect the maximum amount of items available. When I first got to rank 1 halfway through the contest, I didn't even have a search on my move turn. I just did a single pathfinding run. My push search was pretty deep and this mattered more. 

Later on I found a way to generate ALL moves for ALL pushes. During the search for the pushturn I could select any combination of depth 1 pushes and use lookups to get all pathfinding endresults. Generating this move cache took less than a millisecond. You would think it would be best to use this to get a Move-node layer after the first push layer and use full smitsimax during the first 3 phases (push-move-push). However, when I did this, my bot performed worse. I unit tested a lot, because I could not believe it, but it really was true. Even with all moves cached it was better to do Push -> Random Cached Move -> Push -> Random greedy move -> Push -> Random greedy move. So no statistics were gathered on move nodes. 

#### Summarized

##### Move turn




