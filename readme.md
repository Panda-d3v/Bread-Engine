# Overview
Bread engine is a chess engine written in C++. There is still a lot of room for improvement, but Bread is relatively strong (and probably unbeatable by humans). Currently, it is rated 2700 elo on the computer chess rating lists. Bread uses minimax search with NNUE (efficiently updatable neural network) as an evaluation function. The neural network is homemade and trained using lichess's open database with evaluated games.

Bread engine does not have a graphical interface built in. However it supports the uci protocol, you can therefore run it on any chess graphical interface, such as [cute chess](https://github.com/cutechess/cutechess) or [arena](http://www.playwitharena.de/).

# Installation

*Please note that the engine requires a cpu with AVX2 support.*

### Windows

***You can download precompiled binaries for Windows in the release section.***

### Linux

To use the engine on Linux, you need to build the project yourself. You can find more details on how to do this below.

---

If you want to check whether everything is working properly, you can run the executable and write the following commands one after the other:

```console
uci
position startpos
go movetime 5000
```

and the engine will return the best move. You can find more details on the [uci protocol](https://www.wbec-ridderkerk.nl/html/UCIProtocol.html) webpage.

## Build the project yourself

**You need Cmake and a C++ compiler installed.**

The build has been tested using compilers MSVC 19.38, Clang 17.0 and GCC 13.1.

### Windows

Clone the repository, open a command line, navigate to the project's root directory and type:
```powershell
mkdir build
cd build
cmake ..
cmake --build . --target release --config release
```
This will generate a build folder with the executable.

### Linux

Run the script `compile_script_for_linux.sh`. The executable will be generated in the `build` folder.

## Run the engine on a chess graphical interface:
### Cute chess
You can download cute chess [here](https://github.com/cutechess/cutechess/releases).
To add the engine select: tools -> settings -> engines -> + -> command, and choose the engine's executable.

To play a game against the engine, select: game -> new and choose the settings you want.

# Custom commands

Bread engine supports the following commands:
- `bench`: gives the total number of nodes and nodes per seconds on a series of positions.
- `bench nn`: gives the speed of the neural network inference

Bread engine also has a uci option called `nonsense`, which makes it bongcloud, underpromote to bishops and knights, as well as print lyrics from "never gonna give you up" during search.

# Technical details
in this section we provide an overview of the search algorithm, and the main engine features. The explanations are very high level, so feel free to search more details on other resources.

- [Search Algorithm](#search-algorithm)
  - [Minimax](#minimax)
  - [Alpha Beta pruning](#alpha-beta-pruning)
  - [Transposition Table](#transposition-table)
  - [Move ordering](#move-ordering)
  - [Iterative deepening](#iterative-deepening)
  - [Quiescence search](#quiescence-search)
  - [pondering](#pondering)
  - [aspiration windows](#aspiration-windows)
  - [other optimizations](#other-optimizations)
  - [threefold repetition](#threefold-repetition)
- [Neural Network](#neural-network)
  - [Neural network types](#neural-network-types)
  - [Description of NNUE](#description-of-nnue)
  - [the halfKP feature set](#the-halfkp-feature-set)
  - [quantization](#quantization)
  - [optimized matrix multiplication](#optimized-matrix-multiplication)

## Search Algorithm
A chess engine is just a program that takes a chess position and returns the corresponding best move to play. To play a full game of chess, you can give the engine the positions that arise on the board one after the other.

### Minimax
A "good move" in chess can be defined as a move which "improves your position".

So let's say you wrote a function that takes a chess position and returns a score that is positive if white has the advantage, and negative if black has the advantage. A very basic implementation would be to count the white pieces, and subtract the number of black pieces.
The associated score is usually in centipawns but the measure is arbitrary, the important is that the evaluation is coherent: if the position A is better than B for white, then f(A) > f(B).

Now that we have an "evaluation function", we can use it to find moves: For a given chess position, we can look at every legal move, and choose the one that increases the evaluation function the most if it is white's turn to play, and decreases the evaluation the most if it is black's turn to play.
However, this is not enough to make a strong chess engine. The issue is that the evaluation function is never perfect, so the engine will make mistakes. If we use the "piece count" evaluation function, our engine may take a defended pawn with a queen, and loose it on the next move. 
Therefore, what we need to do is consider the opponent's responses to our moves. So when the engine thinks, after queen takes pawn, it will look at all the opponent's legal moves and evaluate these using the evaluation function. The opponent wants to minimize the score so it will pick the lowest evaluation. Now we can update the queen takes pawn position to be bad for us.

We saw how the engine can consider its move, and the opponent's response. We say that the engine searches at depth 2. But there is no reason to stop there, a depth of 3 means the engine considers: its move, the opponent's response, and its move again.
Usually, the deeper the engine can search, the better the output move will be. The main bottleneck is complexity: there is an average of 30 moves per chess position. So at depth 1, the engine needs to search 30 positions. At depth 2, it needs to search 30^2 = 900 positions, and in general 30^depth positions, which gets huge very quickly. The goal of the following sections is to optimize this algorithm.


### Alpha Beta pruning
In the minimax algorithm described earlier, there are some positions that we don't need to search for: if the engine has already found a move that guarantees +0.2 for white and while searching for the next move it finds a guaranteed -1.1 position for black, then there is no need to continue searching for black's other options, as white will never go down that path anyway.
Here is an explanation [video](https://www.youtube.com/watch?v=l-hh51ncgDI&t=562s).

### Transposition Table
In chess, it is possible to reach the same position with different moves. Therefore, to avoid searching for the same positions multiple times, we store hashes of the chess positions along with the score, the depth and best move in a transposition table, and use them when we encounter the position again. However, care must be taken when storing position evaluations. Indeed, with alpha beta pruning, some evaluations are only upper or lower bounds of the real evaluation.

### Move ordering
To leverage the power of alpha beta pruning, it is best to "guess" what moves are best according to some heuristics, and sort the moves from best to worst. This way, more pruning will occur thanks to alpha beta pruning.

### Iterative deepening
Iterative deepening consists of searching for positions at depth 1 first, then depth 2, depth 3 etc.
One might wonder why we need to search at depth 1 if we are going to search at depth 2 anyway. Isn't this a waste of time?
Actually, the results from the depth 1 search are stored in the transposition table, so even though they can't be reused because the depth is shallower, they can be used for enhanced move ordering. Counterintuitively, the benefit from better move ordering counteracts the time loss from searching at shallower depths. Therefore, to search at a given depth, it is better to search through each depth until the target depth is reached.
Moreover, iterative deepening takes care of the following problem: how to know ahead of time at which depth to search, for a limited time chess game? With iterative deepening, one can start a search at infinite depth, and interrupt it when the allocated time for the move has elapsed.

### Quiescence search
An issue with minimax is the horizon effect. We saw it in action when looking at the depth 1 example: the engine can see queen takes pawn, but not pawn takes queen on the next move. This issue was never resolved on leaf nodes. To mitigate positional misunderstanding by the AI, we can run a "quiescence search" instead of directly evaluating leaf nodes. The quiescence search is similar to minimax, but it only searches through captures (and checks) until the position becomes "quiet". At this point the evaluation function is more likely to give an accurate result.

### Pondering
When the opponent is thinking, the engine can run a search on the move that it thinks is the best for the opponent. If the opponent plays the move, we call it a ponderhit, and the engine can play faster as it has already spent time searching for the position. If the opponent plays another move, the engine just searches for the position normally.

### Aspiration windows
After a deep enough search, we do not expect the evaluation to fluctuate a lot. At this point, we can use a "search window" on the root, and prune more branches that are outside the window. However, if the search falls low/high, we need to perform the search again.
This optimisation turned out to decrease the engine strength due to search instabilities and was therefore removed.

### Other optimizations
the following optimizations are also implemented in bread engine:
- [late move reductions](https://www.chessprogramming.org/Late_Move_Reductions)
- [killer moves](https://www.chessprogramming.org/Killer_Move)
- [piece square tables for move ordering](https://www.chessprogramming.org/Piece-Square_Tables)
- [endgame tablebase](https://en.wikipedia.org/wiki/Endgame_tablebase)
- [futility pruning](https://www.chessprogramming.org/Futility_Pruning)


### Threefold repetition
The first idea that comes to mind to implement threefold repetition is to have something like 
```c++
if (pos.is_threefold_repetition()){
    return 0;
}
```
in the minimax function. However, care must be taken for the transposition table, as the evaluation of 0 is "history dependent". In another variation of the search, this position might be winning or losing, and reusing an evaluation of 0 would lead to engine blunders. Therefore, Bread Engine doesn't store positions that started repeating and were evaluated as 0. This way, the problematic positions aren't reused, but higher in the tree, the possibility of forcing a draw is taken into account.

## Neural Network
### Neural network types
In its first prototype, Bread Engine relied on a convolutional neural network as its evaluation function. You can find the network specs [here](./images/CNN%20specs.png?raw=true). Even though convolutional neural networks are suited for pattern recognition, they are too slow for a strong chess engine. A classic multilayer perceptron, on the other hand, may have weaker accuracy, but its speed more than makes up for it. Moreover, recent techniques make use of clever aspects of chess evaluation to speed up these networks even more.

### Description of NNUE
Bread Engine's NNUE is heavily inspired by the approach described by [stockfish](https://github.com/official-stockfish/nnue-pytorch/blob/master/docs/nnue.md).

When evaluating leaf nodes during minimax search, one can note that the positions encountered are very similar. Therefore, instead of running the whole neural network from scratch, the first layer output is cached and reused.

Let's consider a simple deep neural network, taking 768 inputs corresponding to each piece type on each possible square (so 12 * 64 in total). Suppose we have already run the neural network on the starting position, and that we cached the first layer output before applying the activation function (this is called the "accumulator"). If we move a pawn from e2 to e4, we turn off the input neuron number n corresponding to "white pawn on e2" and turn on the input neuron m corresponding to "white pawn on e4". Recall that computing a dense layer output before applying bias and activation is just matrix multiplication. Therefore to update the accumulator, we need to subtract the n-th row of weights from the accumulator, and add the m-th row of weights. The effect of changing the two input neurons on further layers is non trivial, and these need to be recomputed. Therefore, it is advantageous to have a large input layer, and small hidden layers.

### The halfKP feature set
There are many different choices for the input feature representation, but Bread uses halfKP. Note that modern engines rather rely on halfKA, which is similar to halfKP, but adds features for the opponent's king position. 

This board representation works with two components: the perspective of the side to move, and the other perspective.
Each perspective consists of a board representation similar to before, but with separate features for each friendly king square.
So there are 64 king squares * 64 piece squares * 10 piece types (not including the kings) = 40960 input features for one perspective.
This overparametrisation from the intuitive 768 to 40960 input neurons helps to leverage the efficiency of NNUE.
In the case of Bread Engine, the two perspectives are each run through the same set of weights 40960->256 and concatenated into a vector of size 512, with the side to move perspective first, and the other second.
Here are the full [network specs](./images/NNUE%20specs.png?raw=true) (Relu is actually clipped relu between 0 and 1).

### Quantization
An important way to speed up the neural network is to use quantization with SIMD vectorization on the cpu (unfortunately, gpu's aren't usually suited for chess engines as minimax is difficult to paralellize because of alpha beta pruning (see [ybwf](https://www.chessprogramming.org/Young_Brothers_Wait_Concept) or [lazy SMP](https://www.chessprogramming.org/Lazy_SMP)). Also, the data transfer latency between cpu and gpu is too high).

The first layer weights and biases are multiplied by a scale of 127 and stored in int16. Accumulation also happens in int16.  With a maximum of 30 active input features (all pieces on the board), there won't be any integer overflow unless (sum of 30 weights) + bias > 32767, which is most definitely not the case. We then apply clipped relu while converting to int8. Now `layer_1_quant_output = input @ (127*weights) + 127*bias = 127*(input @ weights + bias) = 127*true_output`.
For the second layer, weights are multiplied by 64 and stored in int8, and bias is multiplied by 127 * 64 and stored in int32.
To be able to store weights in int8, we need to make sure that weights * 64 < 127, so a weight can't exceed 127/64 = 1.98. This is the price to pay for full integer quantization. After this layer, the output is
`layer_2_quant_output = quant_input @ (64*weights) + 127*64*bias = 127*64*(input @ weights + bias) = 127*64*true_output`. Therefore, we need to divide the quantized output by 64 before applying the clipped relu (this is done using bitshifts as 64 is a power of 2. We lose at most 1/127 = 0.0078 of precision by doing this).
Layers 3 and 4 are similar. Layer 4 is small enough that we can accumulate outputs in int16.

Here is some code to run the hidden layers (this was used until Bread Engine 0.0.6):
```c++
inline __m256i _mm256_add8x256_epi32(__m256i* inputs){ // horizontal add 8 int32 avx registers.
    inputs[0] = _mm256_hadd_epi32(inputs[0], inputs[1]);
    inputs[2] = _mm256_hadd_epi32(inputs[2], inputs[3]);
    inputs[4] = _mm256_hadd_epi32(inputs[4], inputs[5]);
    inputs[6] = _mm256_hadd_epi32(inputs[6], inputs[7]);

    inputs[0] = _mm256_hadd_epi32(inputs[0], inputs[2]);
    inputs[4] = _mm256_hadd_epi32(inputs[4], inputs[6]);

    return _mm256_add_epi32(
        // swap lanes of the two regs
        _mm256_permute2x128_si256(inputs[0], inputs[4], 0b00110001),
        _mm256_permute2x128_si256(inputs[0], inputs[4], 0b00100000)
    );
};

template<int in_size, int out_size>
void HiddenLayer<in_size, out_size>::run(int8_t* input, int32_t* output){

    __m256i output_chunks[int32_per_reg];
    const __m256i one = _mm256_set1_epi16(1);

    for (int j = 0; j < num_output_chunks; j++){
        __m256i result = _mm256_set1_epi32(0);
        for (int i = 0; i < num_input_chunks; i++){
            __m256i input_chunk = _mm256_loadu_epi8(&input[i*int8_per_reg]);
            for (int k = 0; k < int32_per_reg; k++){
                output_chunks[k] = _mm256_maddubs_epi16(
                    input_chunk,
                    _mm256_loadu_epi8(&this->weights[(j*int32_per_reg+k) * this->input_size + i*int8_per_reg])
                );
                output_chunks[k] = _mm256_madd_epi16(output_chunks[k], one); // hadd pairs to int32
            }
            result = _mm256_add_epi32(result, _mm256_add8x256_epi32(output_chunks));
        }
        result = _mm256_add_epi32(result, _mm256_loadu_epi32(&this->bias[j*int32_per_reg]));
        result = _mm256_srai_epi32(result, 6); // this integer divides the result by 64 to scale back the output.
        _mm256_storeu_epi32(&output[j*int32_per_reg], result);
    }
};
```

### Optimized matrix multiplication
*The following method is original, it may or may not be a good approach.*

To optimize the matrix multiplication code above, one can note that in the `_mm256_add8x256_epi32` function, we use `_mm256_hadd_epi32` for horizontal accumulation. However, we would like to use `_mm256_add_epi32` which is faster, but does vertical accumulation. To achieve this we can shuffle vertically the weights in the neural network, to have the weights that need to be added together on different rows. After the multiplication, we can shuffle the weights horizontally so they align, and add them vertically.

```c++
inline __m256i _mm256_add8x256_epi32(__m256i* inputs){

    inputs[1] = _mm256_castps_si256(_mm256_permute_ps(_mm256_castsi256_ps(inputs[1]), 0b00111001));
    inputs[2] = _mm256_castps_si256(_mm256_permute_ps(_mm256_castsi256_ps(inputs[2]), 0b01001110));
    inputs[3] = _mm256_castps_si256(_mm256_permute_ps(_mm256_castsi256_ps(inputs[3]), 0b10010011));

    inputs[5] = _mm256_castps_si256(_mm256_permute_ps(_mm256_castsi256_ps(inputs[5]), 0b00111001));
    inputs[6] = _mm256_castps_si256(_mm256_permute_ps(_mm256_castsi256_ps(inputs[6]), 0b01001110));
    inputs[7] = _mm256_castps_si256(_mm256_permute_ps(_mm256_castsi256_ps(inputs[7]), 0b10010011));

    inputs[0] = _mm256_add_epi32(inputs[0], inputs[1]);
    inputs[2] = _mm256_add_epi32(inputs[2], inputs[3]);
    inputs[0] = _mm256_add_epi32(inputs[0], inputs[2]);

    
    inputs[4] = _mm256_add_epi32(inputs[4], inputs[5]);
    inputs[6] = _mm256_add_epi32(inputs[6], inputs[7]);
    inputs[4] = _mm256_add_epi32(inputs[4], inputs[6]);
    
    return _mm256_add_epi32(
        inputs[0],
        // swap lanes of the second register
        _mm256_castps_si256(_mm256_permute2f128_ps(_mm256_castsi256_ps(inputs[4]), _mm256_castsi256_ps(inputs[4]), 1))
    );
};
```
In the following representation, same digits need to be added together.

- Previous `_mm256_add8x256_epi32` function:

| input 1: | 0 | 0 | 0 | 0 | 0 | 0 | 0 | 0 |
|----------|---|---|---|---|---|---|---|---|
| input 2: | 1 | 1 | 1 | 1 | 1 | 1 | 1 | 1 |
| input 3: | 2 | 2 | 2 | 2 | 2 | 2 | 2 | 2 | 
| input 4: | 3 | 3 | 3 | 3 | 3 | 3 | 3 | 3 |
| input 5: | 4 | 4 | 4 | 4 | 4 | 4 | 4 | 4 |
| input 6: | 5 | 5 | 5 | 5 | 5 | 5 | 5 | 5 |
| input 7: | 6 | 6 | 6 | 6 | 6 | 6 | 6 | 6 |
| input 8: | 7 | 7 | 7 | 7 | 7 | 7 | 7 | 7 |

| output register: | 0 | 1 | 2 | 3 | 4 | 5 | 6 | 7 |
|------------------|---|---|---|---|---|---|---|---|

- New `_mm256_add8x256_epi32` function (takes vertically shuffled input):

| input 0: | 0 | 1 | 2 | 3 | 4 | 5 | 6 | 7 |
|----------|---|---|---|---|---|---|---|---|
| input 1: | 3 | 0 | 1 | 2 | 7 | 4 | 5 | 6 |
| input 2: | 2 | 3 | 0 | 1 | 6 | 7 | 4 | 5 |
| input 3: | 1 | 2 | 3 | 0 | 5 | 6 | 7 | 4 |
| input 4: | 4 | 5 | 6 | 7 | 0 | 1 | 2 | 3 |
| input 5: | 7 | 4 | 5 | 6 | 3 | 0 | 1 | 2 |
| input 6: | 6 | 7 | 4 | 5 | 2 | 3 | 0 | 1 |
| input 7: | 5 | 6 | 7 | 4 | 1 | 2 | 3 | 0 |

intermediate result using `_mm256_permute_ps`:

| register 0: | 0 | 1 | 2 | 3 | 4 | 5 | 6 | 7 |
|-------------|---|---|---|---|---|---|---|---|
| register 1: | 0 | 1 | 2 | 3 | 4 | 5 | 6 | 7 |
| register 2: | 0 | 1 | 2 | 3 | 4 | 5 | 6 | 7 |
| register 3: | 0 | 1 | 2 | 3 | 4 | 5 | 6 | 7 |
| register 4: | 4 | 5 | 6 | 7 | 0 | 1 | 2 | 3 |
| register 5: | 4 | 5 | 6 | 7 | 0 | 1 | 2 | 3 |
| register 6: | 4 | 5 | 6 | 7 | 0 | 1 | 2 | 3 |
| register 7: | 4 | 5 | 6 | 7 | 0 | 1 | 2 | 3 |

vertical sum of the 4 first and last registers:

| register 0: | 0 | 1 | 2 | 3 | 4 | 5 | 6 | 7 |
|-------------|---|---|---|---|---|---|---|---|
| register 1: | 4 | 5 | 6 | 7 | 0 | 1 | 2 | 3 |

swap lanes of the second register and vertical sum:

| output register: | 0 | 1 | 2 | 3 | 4 | 5 | 6 | 7 |
|------------------|---|---|---|---|---|---|---|---|


# Learning resources
[chess programming wiki](https://www.chessprogramming.org/Main_Page)

[avx tutorial](https://www.codeproject.com/Articles/874396/Crunching-Numbers-with-AVX-and-AVX)

[intel intrinsics guide](https://www.intel.com/content/www/us/en/docs/intrinsics-guide/index.html#ig_expand=140,92,83)

# Notable games
- Bread Engine 0.0.9 vs chess.com's Magnus Carlsen bot 1-0:

1\. d4 Nf6 2. Nf3 d5 3. c4 e6 4. Nc3 Be7 5. cxd5 exd5 6. Bf4 O-O 7. e3 Nh5 8. Be5 Nc6 9. h3 Be6 10. Bh2 Nf6 11. Bd3 Nb4 12. Bb1 c5 13. dxc5 Bxc5 14. O-O Nc6 15. Bc2 Rc8 16. Rc1 a6 17. Bg3 Ba7 18. Qe2 d4 19. Rfd1 Re8 20. Bh4 b5 21. Ne4 Bc4 22. Qe1 Bxa2 23. b3 Bb6 24. Qe2 Rxe4 25. Bxe4 Bxb3 26. Bxh7+ Kh8 27. Bc2 Bc4 28. Bd3 Bb3 29. Bc2 Bc4 30. Bd3 Bb3 31. Bf5 Bxd1 32. Qxd1 Rc7 33. Bg3 Ne7 34. Bxc7 Bxc7 35. Qxd4 Kg8 36. Qa7 Ne8 37. Bc2 a5 38. Qc5 Nd6 39. Rd1 Qd7 40. Nd4 Nec8 41. Qh5 g6 42. Bxg6 fxg6 43. Qxg6+ Kf8 44. e4 b4 45. Rd3 Qe8 46. Qh6+ Ke7 47. e5 Bb6 48. Nf5+ Kd8 49. Nxd6 Bxf2+ 50. Kxf2 Nxd6 51. Qxd6+ Kc8 52. Qa6+ Kb8 53. Rd6 Qf7+ 54. Kg1 Qc7 55. Rb6+ Qxb6+ 56. Qxb6+ Kc8 57. Qxa5 b3 58. Qc3+ Kd7 59. Qxb3 Ke8 60. Qe6+ Kf8 61. Qf6+ Ke8 62. h4 Kd7 63. e6+ Kd6 64. e7+ Kd7 65. Qf8 Kc6 66. e8=Q+ Kd5 67. Qf5+ Kd4 68. Qee4+ Kc3 69. Qc5+ Kb2 70. Qec2+ Ka1 71. Qa3#

- Bread Engine 0.0.4 vs chess.com's Hikaru Nakamura bot 1-0:

1\. d4 Nf6 2. Nf3 c5 3. g3 cxd4 4. Nxd4 e5 5. Nf3 Nc6 6. Bg2 d5 7. O-O Be7 8. c4 d4 9. b4 e4 10. Ng5 Bxb4 11. a3 Ba5 12. Nd2 Bc3 13. Rb1 e3 14. fxe3 O-O 15. Nde4 Ng4 16. h3 Nge5 17. Qc2 f5 18. Nxc3 dxc3 19. Nf3 Qf6 20. Rb3 Qg6 21. Rxc3 Qxg3 22. Nxe5 Qxe5 23. Bb2 Be6 24. Rf4 Qf6 25. Rd3 Ne5 26. Rb3 Rfe8 27. Rb5 Bd7 28. Bxb7 Rad8 29. Rc5 Qb6 30. Bd5+ Kh8 31. Bd4 Qh6 32. Qc3 Ng6 33. Rf3 Re7 34. Ra5 Rb8 35. Ra6 Qg5+ 36. Kh2 f4 37. Rxa7 Nh4 38. Rxf4 Nf5 39. Rf3 Qh5 40. Bf6 Nh4 41. Rf2 Ng6 42. Bxg7+ Rxg7 43. Rxd7 Qe5+ 44. Qxe5 Nxe5 45. Rxg7 Kxg7 46. c5 Rc8 47. Rf5 Nc6 48. Rg5+ Kf6 49. Rh5 Kg6 50. Rh4 Ne5 51. Re4 Kf5 52. c6 Nxc6 53. Re6 Na5 54. Kg3 Rc3 55. a4 Ra3 56. Rb6 Rxa4 57. e4+ Ke5 58. e3 Rxe4 59. Bxe4 Kxe4 60. Kf2 h5 61. Rd6 Nb3 62. Rh6 Nc5 63. Rxh5 Nd3+ 64. Ke2 Nc5 65. h4 Nd7 66. Rg5 Nf6 67. Rg6 Nh5 68. Kf2 Kd3 69. Kf3 Kd2 70. Kg4 Kxe3 71. Kxh5 Ke4 72. Kg4 Kd4 73. h5 Kd3 74. h6 Ke4 75. h7 Kd3 76. h8=Q Kc4 77. Qc3+ Kxc3 78. Kf3 Kd4 79. Rg5 Kd3 80. Rg4 Kc2 81. Ke2 Kc3 82. Re4 Kb2 83. Kd2 Kb3 84. Rd4 Kb2 85. Rd3 Kb1 86. Kc3 Ka1 87. Kb3 Kb1 88. Rd1# 1-0

# Acknowledgements
[lichess's open database](https://database.lichess.org/) for training data for the neural network.

[stockfish](https://stockfishchess.org/) for their documentation and training evaluations.

[tensorflow](https://www.tensorflow.org/)

[onnx runtime](https://onnxruntime.ai/)

[sebastian lague](https://www.youtube.com/watch?v=U4ogK0MIzqk&t=1174s) for inspiring me to do this project

[disservin's chess library](https://github.com/Disservin/chess-library)

[fathom tablebase probe](https://github.com/jdart1/Fathom)
