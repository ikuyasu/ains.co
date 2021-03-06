Title: Calculating the perfect Tetris play: Part I
Date: 2013-05-23 20:15
Category: Programming
Tags: python, actionscript, reverse engineering, programming
Slug: breaking-tetrisfriends
Author: Ainsley Escorce-Jones
Summary: Looking into the decompiled source code of Tetris Friends, and finding out how to predict an infinite stream of future tetrominoes.

When developing bots for classical games such as Chess and Checkers the depth to which the engine evaluates the game tree is limited by the amount of time  and by computing power. However when we play Tetris, the game itself imposes a hard limit on how far we are able see into the future, the preview provided is typically only in the range of 1 to 5 pieces.

I was curious to see what could be done with a bot if you could predict and generate an infinite stream of the future tetrominoes in a game of Tetris. Here in part one of this series I'll describe the process of predicting all the future pieces in a Tetris Friends game and then go into further developments and how to use our new stream of tetrominoes to make gameplay decisions in my next blog post on this topic.

###Tetris Bags

When playing a game of Tetris, the stream of tetrominoes generated by the game typically follows the [Random Generator](http://tetris.wikia.com/wiki/Random_Generator) algorithm described in the Tetris guideline. It's simple enough to understand, every seven pieces generated during a game are chosen as if they are picked from a bag containing all of the possible pieces. After one of each tetromino has been removed from the bag it is refilled again.   
So, from a given "Tetromino bag" there are only 7! (5040) different ways in which pieces can be removed. 

The order of the pieces in a newly generated Tetromino bag should be generated randomly but true randomness is hard to achieve, especially with computers.  

###Tetris Friends
Figuring that Tetris Friends used some form of Psuedorandomness under the hood to refill the Tetromino bags, if I could reimplement the random number generator used in Tetris Friends It would be easy to predict the incoming stream of Tetrominoes given its initial starting conditions.
Tetris Friends games are all Flash based, and as such were easy to decompile using ShowMyCode.com, some local variable names were lost but the meanings were easy enough to deduce.
#####Psuedorandomness
The Tetris Friends random number generator was helpfully named "TetRandom", at first it seemed interesting that Tetris Friends had chosen to implement their own random number generator instead of using the built-in "Math.Random()". However, a custom random number generator enables use of their own seed, a facility not provided by the Adobe Math class. Once Tetris Friends provide a custom seed they can use predictable stream of Tetrominoes in debugging, testing and in multiplayer games. I directly ported the TetRandom class to [Python](https://github.com/ains/tetrisfriends-reversed/blob/master/TetRandom.py), I won't even pretend to understand exactly what the code is doing, but it seems to be a strange variation of a Mersenne Twister.

#####Tetris Friends Tetris Bags
Below is the constructor of the TetPieceBag class, TetPieceBag class implements the Tetris Bag mechanics as described above, refilling itself every time it is emptied with 7 new pieces generated using the m_RandomGenerator property. 

	public function TetPieceBag(size:int, seed:int){
	    m_BagArray = new TetArrayList(size, true);
	    m_RefillArray = new TetArrayList(size, true);
	    m_RefillSlots = new Array(size);
	    m_RandomGenerator = new TetRandom(seed);
	}  

The Python Implementation of this can be found in the Github repository [here](https://github.com/ains/tetrisfriends-reversed/blob/master/TetPieceBag.py). With the Tetris bag implemented in Python along with the TetRandom class, I would be able to enumerate the an infinite stream of tetrominos identical to those produced by Tetris Friends, the only missing link left to find is the seed used to initialize the Psuedorandom Number Generator.

###The Seed
The key to being able to generate a stream of Tetrominoes would be to work out the seed generated for a given run of Tetris.

Running a grep on the decompiled source code for seed yielded promising results:

	this.mTetrisGameSeed = int((Math.random() * 65536));

Initially I looked into the workings of the Math.random() function, however the underlying implementation of the Math.random() call is not specified in the Adobe documentation.

Perhaps this was for the best, as the alternative solution required a lot less work on my part. The Adobe documentation states that Math.random() generates a floating point value between greater than 0.0 and less 1.0, multiplying this by 65536 and truncating the result to produce and integer leaves only 2<sup>16</sup> potential seeds, and subsequently only 2<sup>16</sup> possible streams of Tetrominoes.

As I explained above, Tetris Bags can contain 5040 different possible combinations of Tetrominoes, after they are depleted the next generated Tetris piece can be any of the 7 possible tetrominoes.

This means that there are 7! * 7 (35280) possible combinations of the first 8 pieces in a game of Tetris, for 9 pieces there are 7! * 7 * 6 possible combinations (211680). This is far greater than the number of possible different Tetromino sequences (65536), so given the first 9 pieces of a game we can work out the unique seed that generated that sequence.

2<sup>16</sup> is a small enough value that we can enumerate the first 9 pieces for every seed and then find a match.

Using the above Python implmentation we can generate a dictionary which maps all possible starting sequences to their respective seeds. For a single run however this leads to incredibly long start up times (unless you have the map serialised)

	bagSize = 7
	seedTetrominoMap = {}

	for seed in range(0, 2**16):
	    bag = TetPieceBag(bagSize, seed) 
	    tetrominoSequence = 
	    	''.join(itertools.islice(bag.pieceGenerator(), 0, 9))

	    seedTetrominoMap[tetrominoSequence] = seed


Alternatively for single shot usage you can lasily enumerate the possible tetromino sequences with a generator. Calling next will halt when the first match is found.

	def headMatches(given, bag):
		generatedPieces = 
			itertools.islice(bag.pieceGenerator(), 0, len(given))
		return (list(generatedPieces) == given)

	searchSpace = (TetPieceBag(bagSize, seed) 
		for seed in range(0, 2 ** 16) 
		if headMatches(given, TetPieceBag(bagSize, seed)))

	firstMatch = next(searchSpace)
Full implementation [here](https://github.com/ains/tetrisfriends-reversed/blob/master/generate_stream.py)

###Test Drive
Enough code, here's a quick run through of how it works. <a href="theme/img/breaking-tetrisfriends/demo1.png" class="lightbox" rel="gal">Here</a> I open up Tetris Friends and play 3 pieces (allowing me to see the 9th piece in the sequence).

The initial sequence given is **"L J I Z O T S T J"**, using this as the given variable in generate_stream.py I receive the follwing output

	The seed which was used to generate the given sequence is: 48543
	The first 50 pieces in the tetromino sequence are
	['L', 'J', 'I', 'Z', 'O', 'T', 'S', 'T', 'J', 'L', 'Z', 'O', 'I', 'S', 'L', 'I', 'Z', 'T', 'S', 'J', 'O', 'Z', 'O', 'T', 'J', 'L', 'S', 'I', 'O', 'S', 'I', 'J', 'L', 'Z', 'T', 'L', 'S', 'J', 'Z', 'T', 'I', 'O', 'Z', 'L', 'J', 'I', 'T', 'S', 'O', 'S']

After getting the results I dropped a few more pieces, <a href="theme/img/breaking-tetrisfriends/demo2.png" class="lightbox" rel="gal">here</a> you can see the tetromino preview starting from the 9th piece, they match (forever, trust me)!

###Conclusions
The code above, unfortunately isn't particularly efficient (I wouldn't recommend running it without using Pypy), but what it does show is that after 9 pieces in a Tetris Friends game it is possible to determine all of the future tetrominos.

This code has been sitting around on my hard disk for a few months, but I thought it would be a good start to my blog. I don't expect the Tetris Friends random number generator to stay the same for too long, but luckily that's not too important for next blog post in this series where I'll discuss ways to use an infinite stream of Tetrominoes to make optimal gameplay decisions, and how to get the seed before the game even starts! For a look at what's to come check out [this repository](https://github.com/ains/gotetris) (A pretty fast move evaulator in Golang)

Thanks for reading and all the code above (not the Decompiled source) is available in this [Github Repository](https://github.com/ains/tetrisfriends-reversed).