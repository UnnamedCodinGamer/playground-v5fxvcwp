# Goal

The goal of this playground is show how the Monte Carlo Tree Search (MCTS) algorithm works, applied to a simple two player game.
It is oriented towards people who are acquainted with the theory of MCTS but experience difficulties implementing it.
Below is a basic and (hopefully) easy to follow implementation demonstrating the game play of Tic Tac Toe.
For ease of reading the four main steps of the algorithm are indicated as comments in the code.

# Notice

The code below does not use best practices in programming or MCTS implementation. This playground is to be perceived as a proof of concept example of how the algorithm works.
If you want you can contribute by making it more accessible by restructuring it or by translating the code to other languages.

# Code

```java runnable
// { autofold
import java.util.ArrayList;
import java.util.Arrays;
import java.util.Collections;
import java.util.Random;

class Node implements Comparable <Node> { // representing the state of the game
	
    int player; // 0 if o's turn has been played, 1 otherwise;
	Node parent;
    int [] gameState; // representing the board
    int move; //0-8
	ArrayList<Node> children;
	double numVisits, UCTValue, victories, draws, losses = 0;
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
		else  UCTValue = ((victories+draws/2) / numVisits) + Math.sqrt(2) * Math.sqrt(Math.log(parent.numVisits) / numVisits);
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
		rand.setSeed(1);
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
		

		while (true) { // simulate a random game

			
			ArrayList<Integer> moves = getAllpossibleMoves(currentGameState);
			int randomMoveIndex = rand.nextInt(moves.size());
			int moveToMake = moves.get(randomMoveIndex);
			currentGameState[moveToMake] = player;
            int won = checkWinOrDraw(currentGameState, player);
            if (won != GAME_CONTINUES) return won;
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
		
		forLoop1: for (ArrayList<Integer> w: winningMoves) {
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
		
		simulator = new TicTacToeSimulator();
	}

    Node selectNodeForRollout() { //select
		
		Node currentNode = rootNode;
	
		while (true) {
			
            if (currentNode.winner != TicTacToeSimulator.GAME_CONTINUES) return currentNode; // if terminal node is selected return it for scoring
	        if (currentNode.children.isEmpty()) {
	        	 
	        	generateChildren(currentNode);
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

    void generateChildren(Node n) { // expand
		
		ArrayList<Integer> moves = simulator.getAllpossibleMoves(n.gameState);
		for (Integer i: moves) {
			int [] nextGameState = n.gameState.clone();
			nextGameState[i] = n.player^1;
			Node child = new Node(n.player^1, n, nextGameState, i);
            child.winner = simulator.checkWinOrDraw(child.gameState, child.player); // check if child is end game node
			n.children.add(child);
		}
	}

    void backpropagateRolloutResult(Node n, int won) { // backpropagate

        Node current = n;   
        while (current != null) {
            current.numVisits++;
            if (won == TicTacToeSimulator.DRAW) current.draws+=1;
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
			backpropagateRolloutResult(leafToRollOutFrom, won); // backpropagation phase
			
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
                
                if (f.bestMove.winner == TicTacToeSimulator.O || f.bestMove.winner == TicTacToeSimulator.X) {
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




