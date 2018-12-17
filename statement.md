# X-mas Rush Post Mortem

This contest was a lot of fun for me. Also very intense because of the close fight between number 1 and number 2. I ended at rank 2, which is a personal record for me. Very happy with the result. In this post mortem I will try to explain the main things I think I did right and also the things that didn't work for me.

## Contents

-Simulation
-Search
-Optimization

## Simulation

This contest was hard for beginners to get into. This is mainly because of the necessity of a push simulation. Push is complex in that you need to take into account pushes by both players on 7 possible different rows and columns. The difficulty depends greatly on your choice of board representation. I decided to go with a "bitboard". This is a difficult to handle board representation that is very fast.

Bitboards can be efficiënt because they use less memory and be quickly modified using bit operator. My board was an int[9]. The first 7 elements were 7 map rows, the 8th integer was my hand tile and the 9th integer was my opponents hand tile. One of those integer map rows could look like this in terms of individual bits:  1001 0111 1110 1111 0011 1100 1010 
Basically I used the map tiles we are given as input and only convert them to bits. You can do this as follows:

'''C++
int StringToBinary(string s)
{
	int binary = 0;

	for (int i = 0; i < 4; i++)
	{
		if (s[i] == '1')
			binary += 1 << (3 - i);
	}
	return binary;
}
'''


My Push code you can find below. I show about half of it because it is a big method, but it should be easy to 
