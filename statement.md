# Goal

The goal of this playground is show how the Monte Carlo Tree Search (MCTS) algorithm works, applied to a simple two player game. 
It is oriented towards people who are acquainted with the theory of MCTS but experience difficulties implementing it.
Below is a basic and (hopefully) easy to follow implementation demonstrating the game play of Tic Tac Toe.
For ease of reading the four main steps of the algorithm are indicated as comments in the code.

# Prerequisits to use the code

- basic understanding of MCTS
- basic understanding of Java

# Things to try

- Try different number of iterations that the algorithm goes through. 
For instance with lower numbers it does not manage to collect enough information through random rollouts and the game finishes as win for one of the players.
- Try different value of the exploration paramenter (currently set to 2).
 For example with higher values the algorithm favours exploration over exploitation which results in relatively higher visit count for suboptimal moves.

# Things to try to implement 

- Try to implement different expansion policy and expand single child at a time - in the current version all children are expanded at once and one of them is picked for rollout.
Single node expansion is a bit more complicated to implement but saves memory.
- Try to implement cleverer rollout method - currently the rollout is completely random and the players will take suboptimal moves (for instance miss a victory or miss to block the other player who is about to win). 
Better rollouts lead to better results.

# Notice
I do not hold a CS degree and am not a professional developer.
The code below does not pretend to use best practices in programming or MCTS implementation and can be optimized. This playground is to be perceived as a proof of concept writeup that demonstrates the working of the algorithm.

# Code

```java runnable
// { autofold
import java.util.ArrayList;
import java.util.Arrays;
import java.util.Collections;
import java.util.Random;

class Node implements Comparable <Node> { // representing the state of the game
	
	Node parent;
	ArrayList<Node> children;
	int [] gameState; // represent the board
	double numVisits, UCTValue, victories, draws, losses = 0;
	int player; // 0 if o's turn has been played, 1 otherwise;
	int move; //0-8
    int winner = TicTacToeSimulator.GAME_CONTINUES; // indicates if node is end game node (game is won, lost or drawn)
	
	Node (int pl, Node p, int[] s, int m) {
		player = pl;
		parent = p;
		gameState = s;
		move = m;
		children = new ArrayList<Node>();
	}
	
	@Override
	public int compareTo(Node other) { // sort nodes in descending order according to their UCT value
		
		return Double.compare(other.UCTValue, UCTValue);
	}
	
	void setUCTValue() {
		
		if (numVisits == 0) UCTValue = Double.MAX_VALUE; // make sure every child is visited at least once
		else  UCTValue = ((victories+draws) / numVisits) + 2 * Math.sqrt(Math.log(parent.numVisits) / numVisits);
	}
}

class TicTacToeSimulator {

    static final int EMPTY = -1;
    static final int O = 0;
    static final int X = 1;

    static final int DRAW = 2;
    static final int GAME_CONTINUES = -2;
	
	ArrayList<ArrayList<Integer>> winningMoves; // holds all winning moves for look-up;
	Random rand;
	
	TicTacToeSimulator() {
		
		rand = new Random();
		rand.setSeed(1); // set seed to enable reliable replays
		setWinningMoves();
	}
	
	private void setWinningMoves() {
		
		winningMoves = new ArrayList<ArrayList<Integer>>(Arrays.asList(   	
                                                                        new ArrayList<Integer>(Arrays.asList(0, 1, 2)), 
                                                                        new ArrayList<Integer>(Arrays.asList(3, 4, 5)),
                                                                        new ArrayList<Integer>(Arrays.asList(6, 7, 8)),
                                                                        new ArrayList<Integer>(Arrays.asList(0, 3, 6)),
                                                                        new ArrayList<Integer>(Arrays.asList(1, 4, 7)),
                                                                        new ArrayList<Integer>(Arrays.asList(2, 5, 8)),
                                                                        new ArrayList<Integer>(Arrays.asList(0, 4, 8)),
                                                                        new ArrayList<Integer>(Arrays.asList(2, 4, 6))));
	}
	
	int simulateGameFromLeafNode(Node n) { // do rollout
		
		if (n.winner != GAME_CONTINUES) return n.winner; // check if game is won and node is terminal
		int player = n.player^1; // whose player's turn it is to make a move
		int [] currentGameState = n.gameState.clone();
		ArrayList<Integer> moves = new ArrayList<Integer>();
	
		while (true) { // simulate a random game

			moves.clear();
			moves = getAllpossibleMoves(currentGameState);
			if (moves.isEmpty()) {
				return DRAW; 
            }
			int randomMoveIndex = rand.nextInt(moves.size());
			int moveToMake = moves.get(randomMoveIndex);
			currentGameState[moveToMake] = player;
            int won = checkWinOrDraw(currentGameState, player);
            if (won == player) {
                return player;
            }
			
			player^=1;
		}
	}
	
	ArrayList<Integer> getAllpossibleMoves(int [] gameState) { 
		
		ArrayList<Integer> allPossibleMoves = new ArrayList<Integer>(9);
		for (int row = 0; row < gameState.length; row++) {
			if (gameState[row] == EMPTY) {
				allPossibleMoves.add(row);
			}
		}
		return allPossibleMoves;
	}
	
	int checkWinOrDraw(int [] gameState, int player) {
		
		forLoop1: for (ArrayList<Integer> w: this.winningMoves) {
			int n = 0;
			for (Integer index: w) {
				
				if (gameState [index] != player) {
					continue forLoop1;
				} else n++;
			}
			if (n == 3) return player;
		}

		ArrayList<Integer> moves = getAllpossibleMoves(gameState);
		if (moves.isEmpty()) return DRAW;
		
		return GAME_CONTINUES;
	}
	
	void generateChildren(Node n) { // expland
		
		ArrayList<Integer> moves = getAllpossibleMoves(n.gameState);
		for (Integer i: moves) {
			int [] nextGameState = n.gameState.clone();
			nextGameState[i] = n.player^1;
			Node child = new Node(n.player^1, n, nextGameState, i);
            child.winner = checkWinOrDraw(child.gameState, child.player); // check if child is end game node
			n.children.add(child);
		}
	}
	
	void printGameState2D(int[] gameState) {
		
		for (int i = 1; i < gameState.length+1; i++) {
			if (gameState[i-1] == O) System.out.print("O ");
			else if (gameState[i-1] == X) System.out.print("X ");
			else System.out.print("- ");
			if (i % 3 == 0) System.out.println();
		}
	}
}

class MCTSBestMoveFinder {
	
	TicTacToeSimulator simulator;
    Node rootNode;
	Node bestMove;

	MCTSBestMoveFinder() {
		
		this.simulator = new TicTacToeSimulator();
	}

    Node selectNodeForRollout() { //select
		
		Node currentNode = rootNode;
	
		while (true) {
			
	        if (currentNode.children.isEmpty()) {
	        	 
        		if (currentNode.winner != TicTacToeSimulator.GAME_CONTINUES) return currentNode; // check if game is won and node is terminal - no need to expand terminal node
	        	simulator.generateChildren(currentNode);
	        	return currentNode.children.get(0);
	        } else {
	        	for (Node child: currentNode.children) {
	        		child.setUCTValue();
	        	}
	        	Collections.sort(currentNode.children);
	        	currentNode = currentNode.children.get(0);
	        	if (currentNode.numVisits == 0) {
	        		return currentNode;
	        	} 
	        }
		}
	}

    void backpropagateRolloutResult(Node n, int won) { // backpropagate

        Node current = n;   
        while (current != null) {
            current.numVisits++;
            if (won == simulator.DRAW) current.draws+=0.5;
            else if (current.player == won) {
                current.victories+=1;
            } else current.losses+=1;
            current = current.parent;
		}
    }

	void findBestMove(int numIterations) {
		
		for (int i = 0; i < numIterations; i++) {

			Node leafToRollOutFrom = selectNodeForRollout(); // selection / expansion phase
			int won = simulator.simulateGameFromLeafNode(leafToRollOutFrom); // rollout phase
			backpropagateRolloutResult(leafToRollOutFrom, won); // backpropagation
			
		}

        double numVisits = 0; // iterate over the children of the root node and pick as best move the node which had been visited most often
        for (Node child: rootNode.children) {
            if (child.numVisits > numVisits) {
                bestMove = child;
                numVisits = child.numVisits;
            }
        }
        for (Node child: rootNode.children) System.out.println(Arrays.toString(child.gameState) + " " + child.numVisits + " " +child.victories +" "+child.draws+" "+ child.losses +" "+ child.UCTValue + " " + child.move);
		System.out.println();
        simulator.printGameState2D(bestMove.gameState);
		System.out.println();
	}
}

class Main {

	public static void main(String[] args) {

		MCTSBestMoveFinder f = new MCTSBestMoveFinder();
		int numberOfIterations = 1000;

		int [] initGameState = new int [9];
		Arrays.fill(initGameState, TicTacToeSimulator.EMPTY);
		
		int iter = 0;
		while (true) { // game loop

			if (iter == 0) {
				f.rootNode = new Node(TicTacToeSimulator.O, null, initGameState, -1);
			} else {
                int won = f.simulator.checkWinOrDraw(f.bestMove.gameState, f.bestMove.player);
                if (won == TicTacToeSimulator.O || won == TicTacToeSimulator.X) {
                    System.out.println("suboptimal game, increase the number of iterations");
					break;
				}
                f.rootNode = new Node(f.bestMove.player, null, f.bestMove.gameState, f.bestMove.move);
			}
			if (!f.simulator.getAllpossibleMoves(f.rootNode.gameState).isEmpty()) f.findBestMove(numberOfIterations);
			else break;
			iter++;
		}
	}
}

```



