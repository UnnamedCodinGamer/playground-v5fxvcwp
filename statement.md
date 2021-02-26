# Goal

The goal of this playground is show how the monte carlo tree search (MCTS) algorithm works, applied to a simple two player game. 
It is oriented towards people who are acquainted with the theory of MCTS but experience difficulties implementing it.
Below is a basic and (hopefully) easy to follow implementation demonstrating the game play of Tic Tac Toe.
To ease the reading the four main steps of the algorithm are indicated as comments in the code.

# Prerequisits to use the code

- basic understanding of MCTS
- basic understanding of Java

# Things to try

- try different number of iterations that the algorithm goes through. For instance with lower numbers it does not manage to collect enough information through random rollouts and the game finishes as win for one of the players.
- try different value of the exploration paramenter (currently set to 2). For example if you set higher values the algorithm favours exploration over exploitation which results in higher visit count for suboptimal moves.

```java runnable
// { autofold
import java.util.ArrayList;
import java.util.Arrays;
import java.util.Collections;
import java.util.Random;

class Node implements Comparable <Node> { // representing the state of the game
	
	Node parent;
	ArrayList<Node> children;
	int [] gameState;
	double numVisits, UCTValue, victories = 0;
	int player; // 0 if o's turn has been played, 1 otherwise;
	int move; //0-8
    int winner = TicTacToeSimulator.GAME_CONTINUES;
	
	Node(int pl, Node p, int[] s, int m) {
		player = pl;
		parent = p;
		gameState = s;
		move = m;
		children = new ArrayList<Node>();
	}
	
	@Override
	public int compareTo(Node other) {
		
		return Double.compare(other.UCTValue, UCTValue);
	}
	
	void setUCTValue() {
		
		if (numVisits == 0) UCTValue = Double.MAX_VALUE; // make sure every child is visited at least once
		else UCTValue = (victories / numVisits) + 2 * Math.sqrt(Math.log(parent.numVisits) / numVisits);
	}
	
	Node getBestMove() {
		
		double numVisits = 0;
		Node bestMove = null;
		for (Node child: children) {
			if (child.numVisits > numVisits) {
				bestMove = child;
				numVisits = child.numVisits;
			}
		}
		if (bestMove == null) System.out.println("game won");
		return bestMove;
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
		
		int won = checkWinOrDraw(n.gameState, n.player); 
		if (won != GAME_CONTINUES) return won; // check if game is won and node is terminal
		int player = n.player^1; // whose player's turn it is to make a move
		int [] currentGameState = n.gameState.clone();
		ArrayList<Integer> moves = new ArrayList<Integer>();
	
		while (true) {
			moves.clear();
			moves = getAllpossibleMoves(currentGameState);
			if (moves.isEmpty()) {
				return DRAW; // draw
			}
			
			for (Integer m: moves) {
				int [] newGameState = currentGameState.clone();
				newGameState[m] = player;
				won = checkWinOrDraw(newGameState, player);
				if (won == player) {
					return player;
				}
				
			}
			
			int randomMoveIndex = rand.nextInt(moves.size());
			int moveToMake = moves.get(randomMoveIndex);
			currentGameState[moveToMake] = player;
			
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
	
	Node rootNode;
	TicTacToeSimulator simulator;
	Node bestMove;
	
	MCTSBestMoveFinder() {
		
		this.simulator = new TicTacToeSimulator();
	}

    Node selectNodeForRollout() { //select
		
		Node currentNode = rootNode;
	
		while (true) {
			
	        if (currentNode.children.isEmpty()) {
	        	int won = simulator.checkWinOrDraw(currentNode.gameState, currentNode.player); // check if game is won and node is terminal - no need to expand terminal node
        		if (won != simulator.GAME_CONTINUES) return currentNode;
	        	simulator.generateChildren(currentNode);
	        	currentNode = currentNode.children.get(0);
	        	return currentNode;
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

	void findBestMove(int numIterations) {
		
		for (int i = 0; i < numIterations; i++) {
			Node leafToRollOutFrom = this.selectNodeForRollout();
			int won = simulator.simulateGameFromLeafNode(leafToRollOutFrom);
			
			Node current = leafToRollOutFrom;   // backpropagate
			while(current != null) {
				current.numVisits++;
				if (won == simulator.DRAW) current.victories+=0.5;
				else if (current.player == won) {
					current.victories+=1;
				} 
				current = current.parent;
			}
		}
		for (Node child: rootNode.children) System.out.println(Arrays.toString(child.gameState) + " " + child.numVisits + " " +child.victories +" "+ child.UCTValue + " " + child.move);
		bestMove = rootNode.getBestMove();
		if (bestMove != null) simulator.printGameState2D(bestMove.gameState);
		System.out.println();
	}
}

class Main {

	public static void main(String[] args) {

		MCTSBestMoveFinder f = new MCTSBestMoveFinder();
		int numberOfIterations = 200;

		int [] initGameState = new int [9];
		Arrays.fill(initGameState, TicTacToeSimulator.EMPTY);
		
		int iter = 0;
		while (true) { // game loop

			if (iter == 0) {
				f.rootNode = new Node(TicTacToeSimulator.O, null, initGameState, -1);
			} else {
				if (f.bestMove == null) break;
				f.rootNode = new Node(f.bestMove.player, null, f.bestMove.gameState, f.bestMove.move);
			}
			if (!f.simulator.getAllpossibleMoves(f.rootNode.gameState).isEmpty()) f.findBestMove(numberOfIterations);
			else break;
			iter++;
		}
	}
}

//}
```


