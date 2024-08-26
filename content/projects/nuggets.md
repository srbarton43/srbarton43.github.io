---
title: Nuggets
author: Samuel Barton

date: 2023-04-22
summary: A multi-player exploration game inspired by Rogue
description: Nuggets is a multi-player exploration game inspired by Rogue. The object of the game is to collect more gold nuggets than any other player. The game ends when all gold nuggets have been collected by some player. I created this game with 3 other students as our final project for COSC50.
cover:
    image: /projects/nuggets/cover.png
    hiddenInList: true
TocOpen: false
ShowToc: true
---

## Design

The *Nuggets* game requires two standalone programs, the client and the server.
We have further broken down these programs into smaller modules.

### Client

The *client* acts in one of two modes:

1. *spectator*, the passive spectator mode
2. *player*, the interactive game-playing mode

#### Functional decomposition into modules

Simply a client module, handling all client functionality.

#### Major data structures

##### `client`
The client data structure  holds all important values for the client as a global variable:
* nrows - stores number of rows in screen
* ncols - stores number of cols in screen
* justCollected - stores number of gold just picked up for status line
* purse - player purse for status line
* goldRemaining - number of nuggets remaining
* map - the string representation of the map
* playername - the name of player, null if spectator
* playerID - player's id for status line
* hostname - hostname of server
* port - port of server

### Server

#### Functional decomposition into modules

* The *grid* module keeps a 2-dimensional array of gridpoints, where each gridpoint stores information about what character it is (".", "#", etc.), what player is on it (if applicable) and how much gold it contains.
* The *game* module keeps track of the game grid, along with an array of the *players* (and any spectators) that are in the game. As outlined above, each player stores its *(row, column)* position, purse size, playerID, and the portion of the game grid that is currently visible to him/her.

#### Major data structures

##### `game`
The game module will exist within the server to hold a state of Nuggets as it is being played, complete with the current game grid, the players, spectator, and more. 
* Grid - stores data about points across map of the game
* array of players - stores each of the players that are in the game
* lastPlayerNum - Array index of player last added
* spectator - stores unique spectator player data, if one exists

##### `player`
The player module will handle various aspects of players within the game. Its uses will include storing the regions of the grid visible to the player and formatting it for clients, tracking a player's position and how much gold they have collected, storing their name for the leaderboard, and more.
* name - stores the player's name
* playerID - the player's unique character playerID
* 2d character array map - represents the portion of the grid visible to the player
* purse - stores amount of gold the player has
* row - stores the player's row
* col - stores the player's column
* address - player's address in the network, so we can identify players as we receive client messages

##### `grid`
The grid will represent the data stored on the map and at each point on the map. The grid module loads a mapfile into the game's memory, scatters gold across open rooms, and facilitates players' interaction with the map on the server's end as they search for nuggets. 
* 2d array of gridpoints which store data about each point on the grid
* number of rows in the grid
* number of columns in the grid
* number of roomSpots in the grid
* number of gold pieces remaining in the grid

## Implementation

The game was implemented in C, and uses the ncurses library in order to provide the text display (see https://github.com/srb-private-org/nuggets-game). 

{{< figure src="/projects/nuggets/clientserver.png" caption="Diagram of Client/Server Implementation" >}}

### Client

#### Data structures

##### `client`:

A global struct to track the size of client's window, their status line in the game (which will either print the amount of gold they have or any error messages that come up), and the map that's visible to them.

```c
static struct client {
	int nrows; 
	int ncols;
  	int justCollected;
  	int purse;
  	int goldRemaining;
  	char* map;
  	char* playername;
  	char playerID;
	bool connected;
  	char* hostname;
  	char* port;
}
```

#### Definition of function prototypes

A function to parse the command-line args, initialize the network:

```c
static void parseArgs(const int argc, char* argv[], char** hostname, char** port,
                      char** playername);
```

This function initializes the ncurses objects needed by the client, including the display.

```c
static void ncurses_init();
```

This function updates the display to correspond with the current state of the client.

```c
static void updateDisplay(const char* statusLine);
```

This function is used in conjunction with message_loop to handle input.

```c
static bool handleInput(void* arg);
```

This function sends the key input to the server.

```c
static bool sendKey(const addr_t to, int key);
```

This function sends the player (or spectator) message to the server.

```c
static bool sendPlayMessage(const addr_t to, char* playerName);
```

This function handles the message from server.

```c
static bool handleMessage(void* arg, const addr_t from, const char* message);
```
This function handles the `OK` message.

```c
static bool handleOK(const char* message);
```

This function handles the `GRID` message.

```c
static bool handleGrid(char* message);
```

This function handles the `GOLD` message.

```c
static bool handleGold(char* message);
```

This function handles the `DISP` message.

```c
static bool handleDisp(char* message);
```

This function handles the `QUIT` message.

```c
static bool handleQuit(char* message);
```

This function handles the `ERROR` message.

```c
static bool handleError(char* message);
```

This function handles unrecognized messages.

```c
static bool handleUnrec(char* message);
```

This function returns player status line.

```c
static char* getPlayerStatus();
```

This function returns spectator status line.

```c
static char* getSpectatorStatus();
```

#### Detailed pseudo code

##### `main`:

	parse & validate command line args with parseArgs
	initialize log module
	form server address from args
	set ncurses (including the display) up with ncurses_init
	// do not init ncurses yet ; do so once we know we have received grid data from server, so we know we have a connection
	call sendPlayMessage with the playerName (which is null if the client is spectating)
	begin message_loop, passing display and functions for handling input and messages
	exit curses
	shut down communication with server

##### `parseArgs`:
	if command line does not have two or three arguments
		exit nonzero
	if we cannot set an address to the given hostname and portnumber using message_setAddr
		exit nonzero

##### `ncurses_init`:

	initialize the screen
	call cbreak
	call noecho
	create a new global struct client
	store the number of rows and columns on the screen in client's nrows and ncols
	set the color pair to yellow and black

##### `updateDisplay`:

	print the status line at the top of screen
	print the map under the statusline
	call refresh to update the screen

##### `handleInput`:
	
	set an adrr_t* to the void pointer passed in
	if that addr_t* is NULL
		print a message to stderr
		return true
	if that address is not a valid address
		print a message to stderr 
		return true
	if the `connected` bool in client var hasn't been set to true
		log the error
		return true to exit
	
	set int c equal to the next keyboard press
	return sendKey of c

##### `sendKey`:

	if the server passed in is NULL
		print an error message to log
		return true
	send message to server with form "KEY k"
	return false

##### `sendPlayMessage`:

	if server is null
		print error message to stderr
	else if playerName is null
		send "SPECTATE" message to server
	else
		send message to server of form "PLAY (playerName)

##### `handleMessage`:

	if server is null
		print error message to stderr
		return true
	if first word is OK
		call handleOK with messages
	else if first word is GRID
		return handleGrid with message
	else if first word is GOLD
		return handleGold with message
	else if first word is DISPLAY
		return handleDisp  with message
	else if first word is QUIT
		return handleQuit with message
	else if first word is ERROR
		return handleError with message
	else
		return handleUnrec with mesaage

##### `handleOK`:

	parse the player ID from the message and add it to the struct
	return false

##### `handleGrid`:

	call ncurses_init() since we know we are now connected
	set the global client var's connected field to true
	parse nrows and ncols from the message
		while client struct's nrows and ncols are less than nrows + 1 and ncols, respectively
			print a window sizing message to log
			tell the user to enlarge the window and try again
	return false

##### `handleGold`:

	parse number nuggets collected
	parse number nuggets in purse 
	parse number nuggets still to be found
	update the client struct's statusLine with these values
	call updateDisplay on the client struct
	return false
		
##### `handleDisp`:

	starting after the first newline character, parse grid display string from message
	set client struct's map equal to that string
	update the map with ncurses
	return false

##### `handleQuit`:

	end curses
	parse the message text following the word "QUIT" into a string
	print this string to stdout followed by newline
	return true

##### `handleError`:

	parse explanation into string
	write error message to log
	set client's status line equal to the text in the error message
	call updateDisplay
	return false

##### `handleUnrec`:

	log error message with log.h since unhandled message
	return false

##### `getPlayerStatus`:

	return formatted status line for player in memory

##### `getSpectatorStatus`:

	return formatted status line for spectator in memory

### Server

#### Data structures

##### Game Data Structure
This data structure will store info about the current game being played, including the game grid, an array containing slots for `MaxPlayers` (26) players in the game. Within the server module, game data will be stored globally so that it need not be passed to and validated by every function.

```c
struct gameData {
  grid_t* grid;
  player_t* players[MaxPlayers];
  player_t* spectator;
  int nextPlayerIndex;
}
```

#### Definition of function prototypes

A function to parse & validate the command-line arguments and initialize the game struct
```c
static int parseArgs(const int argc, char* argv[]);
```

A function to initialize the game data struct and the grid & player data therein. 
```c
static void initializeGame(char* mapFile);
```

A function to delete the game data and the grid and player data therein.
```c
static void gameData_delete();
```

A function to send the game's grid dimensions to clients
```c
static void sendGridDimensions(const addr_t to);
```

A function to send gold information to clients.
```c
static void sendGoldInfo(const addr_t to, int justCollected, int purse, int remaining);
```

A function to send the current display to clients.
```c
static void sendDisplay(const addr_t to, char* stringifiedMap);
```

A function to send QUIT messages with explanations to clients.
```c
static void sendQuit(const addr_t to, const char* explanation);
```

A function for handling messages from the client
```c
static bool handleMessage(void* arg, const addr_t from, const char* message);
```

A function to handle `SPECTATE` messages.
```c
static void handleSpectateMessage(const addr_t from);
```

A function to handle `PLAY` messages.
```c
static void handlePlayMessage(const addr_t from, const char* message);
```

A function to handle `KEY` keystroke messages.
```c
static bool handleKeyMessage(const addr_t from, const char* message);
```

A function to handle movement key presses
```c
static bool handleMovementCase(const int rowChange, const int columnChange, player_t* player);
```

A function to handle unrecognized messages.
```c
static void handleUnrecognizedMessage(addr_t to, const char* message);
```

A function to get a player pointer from its address.
```c
static player_t* getPlayerFromAddress(addr_t address);
```

A function to send error messages to clients
```c
static void sendError(addr_t to, const char* message);
```

A function to send OK messages to clients
```c
static void sendOK(const addr_t to);
```

A function to check if a line is blank
```c
static bool isBlankLine(const char* string);
```

A function to format a player name.
```c
static void formatName(char* name);
```

A function to convert a player's index in the players array into an ID
```c
static inline char indexToPlayerID(const int index);
```

A function to send a game over quit message to each player.
```c
static void gameOver();
```

A function to update each player's map and send it to them
```c
static void sendUpdatedMaps();
```


#### Detailed pseudo code

##### main

	start logging to stderr
	call parseArgs() with the given cmd line args
	start messages module, validate init
	call message_loop() with handleMessage, store result in bool
	call gameOver()
	call message_done()
	call log_done()
	call gameData_delete()
	return zero if message loop worked, otherwise nonzero


##### `parseArgs`:

	validate commandline args
	if wrong number of args is given
		exit with proper usage message
	if seed provided
		verify it is a valid seed number
		seed the random-number generator with that seed
	else
		seed the random-number generator with getpid()
	verify map file can be opened for reading
	call initializeGame() with the validated map file name

##### `initializeGame`

	allocate memory for global gameData variable
	call grid_loadMap with given mapFile
	call grid_placePiles on the map to scatter gold across the grid
	allocate space for MaxPlayers in game data
	set spectator pointer to null in game data
	set nextPlayerIndex to zero in game data


##### `handleMessage`

	given a message and sender
	if the message matches "SPECTATE"
		call handleSpectateMessage() with sender address
	elif the message begins with "PLAY "
		call handlePlayMessage() with sender address and message
	elif the message begins with "KEY "
		call handleKeyMessage() with sender address and message
		return true to exit loop if key press resulted in game ending
	else
		call handleUnrecognizedMessage, with message
	return false


##### `sendGridDimensions`

	get number of rows, cols stored in game data grid
	build string of form "GRID numberRows numberColumns"
	send the string to the given `to` address


##### `sendGoldInfo`

	given positive ints justCollected, purse, remaining
	build string of form "GOLD justCollected purse remaining"
	send string to given `to` address


##### `sendDisplay`
	
	given a player
	call player_mapToString with the player and game grid dimensions
	form the message "DISPLAY\n" followed by the map string
	send the message to the player's address
	free the map string allocated in memory


##### `sendQuit`

	send "QUIT " followed by given explanation to `to` address


##### `handleSpectateMessage`

	if game's spectator is not NULL
		sendQuit with relevant explanation to old spectator's addr
		call player_delete on old spectator
		set spectator pointer in game data to NULL
	call player_new with null name, given `from` address, grid dimensions, (-1,-1) as row,col, true for isSpectator
	if we created the player without any issues
		store it in gameData spectator pointer
	call grid_updateSpectatorMap on the spectator with the game grid
	call sendGridDimensions to spectator's addr with game grid dimensions
	call sendGoldInfo to spectator address with justCollected = 0, purse = 0, and remaining gold in grid
	call sendDisplay with pointer to spectator

##### `handlePlayMessage`

	if the game's nextPlayerIndex equals MaxPlayers
		call sendQuit to the new player, explaining that the game's full
		return
	store the given username in memory
	if isBlankLine() returns true for the username
		sendQuit to player with explanation
		free the username
		return
	call formatName on the username
	create a new player with player_new
	if we failed create the player
		free the username, log error, and return
	store the player at the nextPlayerIndex in gameData
	increment game data's nextPlayerIndex
	call grid_addPlayerRandom to place the player at random room spot in grid
	call grid_updatePlayerMap
	sendGridDimensions() to player
	sendGoldInfo() to palyer with justCollected = 0, purse = 0, and remaining gold
	sendDisplay() to player
	update spectator's and each player's map, send it to them

##### `handleKeyMessage`

	parse next letter 
	if we have more than a letter or no letter
		sendQuit, with explanation
		return false (don't exit message loop)
	call getPlayerFromAddress() with address to matching player pointer
	if we failed to find a match
		log the error
		return false
	take the key from the message
	switch with key char as conditional
		case Q
			call sendQuit, w/ explanation
			if the player pointer points to the same place as spectator
				sendQuit with "Thanks for watching!"
				call player_delete on spectator
				set spectator pointer to NULL
			else
				call grid_deletePlayer()
				sendQuit with "Thanks for playing!"
				// do NOT delete quit players - only spectator
				// we need to keep player data for the leaderboard!
				update player's maps with player gone
				send new map to each player
				update spectator map
				send new map to spectator
			return false (don't exit msg loop)
		cases (h, l, j, k, y, u, b, n) or any of their capitalized forms
			return handleMovementCase with the address, player, and key
		default case (invalid)
			call handleUnrecognizedMessage
			sendError with "unknown keystroke"
	return false


##### `handleMovementCase`

	given from address, player, key pressed
	save the player's current purse value
	store the result of the player's attempt to move from grid_movePlayer
	if the player successfully moved
		if the player found gold
			get the remaining gold in the grid
			if there's no remaining gold
				return true (to exit message loop & trigger game over)
			else
				calculate gold just collected by subtracting old purse from new purse
				sendGoldInfo to player with just collected gold, new purse, remaining gold
				loop through player array
					if the player is not null
						sendGoldInfo to the player, with 0 just collected, their purse, gold remaining
				if we have a spectator
					sendGoldInfo, with 0 just collected, 0 purse, gold remaining
		loop through each player
			if the player's not null
				call grid_updatePlayerMap on the player with game's grid
				call sendDisplay to that player
		if we have a spectator
			call grid_updateSpectatorMap
			call sendDisplay to the specator
	return false


##### `handleUnrecognizedMessage`

	log that we could not recognize message


##### `sendError`

	send message of form "ERROR explanation" to client

##### `sendOK`

	send message of form "OK k" to client, where k is a given char


##### `gameOver`

	create special game over string to send, calculating buffer large enough to hold it
	loop through player array in game data
		if the player's not null
			add a single line to the game over string with their ID, purse, and name
	loop through player array in game data
		if the player is not null
			sendQuit to their address with the game over string 
	if we have a spectator
		sendQuit to their address with game over string


##### `indexToPlayerID`

	if the index is NOT within the range of 0..MaxPlayers
		return null character
	return 'A' + index given

##### `isBlankLine`

	for each char in given string
		if isspace(char) does not return true
			return false
	return true


##### `formatName`

	if the length of the name exceeds MaxNameLength
		set the character in the name string at MaxNameLength to be the terminating null char
	loop through index in name, until we hit its length
		if neither isgraph(char) and isblank(char)
			set that char in the string to be an underscore


##### `gameData_delete`

	if the grid in the game data is null
		log error, exit program
	for each index in player array from [0..MaxPlayers)
		if the player's not null
			call player_delete on each player
	free player array's memory
	call player_delete on the spectator, if we have one
	call grid_delete on the grid stored in game data
	free global gameData's allocated memory


##### `getPlayerFromAddress`

	if we cannot validate the address with message_isAddr
		return null player pointer
	if player array in game data is not null
		loop through each player
			if the player isn't null
				if message_eqAddr(given address, player address)
					return pointer to that player
	if we have a spectator
		if message_eqAddr(given address, spectator address)
			return pointer to the spectator
	return null

##### `sendUpdatedMaps`

	for each player from 0 to MaxPlayers in gameData
		if the player is not null
			update player's map
			send it to them
	if the spectator in gameData isn't null
		update spectator's map
		send it to them

---

### Grid module

The `grid` module allows us to construct and track a grid of *gridpoints*, where each gridpoint stores the amount of gold it has, what player is on it (if applicable), and what character the it should display. `grid` also provides several methods that interact with the `player` module, allowing the grid of gridpoints to be continually manipulated as players move around and collect gold.

#### Data structures

##### gridpoint

Stores the status of a given gridpoint in the grid. Will constantly be updated as players move onto gridpoints, off of gridpoints, and collect gold.

```c
struct gridpoint{
  int goldCount;
  char gridChar;
  player_t* player;
}
```

##### grid

Stores a grid of gridpoints and the amount of gold remaining in these gridpoints, along with the grid's number of rows and columns. This struct is a crucial part of Server's game struct, as it stores the state of a given nuggets game.

```c
struct grid{
  gridpoint_t** gridpoints;
  int goldRemaining;
  int numRows;
  int numCols;
  int numRooms;
}
```

#### Definition of function prototypes

A function to load a text file into a struct grid.

```c
grid_t* grid_loadMap(char* fileName);
```

A function to insert a player at a given position in a grid. Returns true if player was inserted, false otherwise.

```c
bool grid_addPlayer(grid_t* grid, player_t* player, int row, int col);
```

A function to insert a player at a random room spot in the grid.

```c
void grid_addPlayerRandom(grid_t* grid, player_t* player);
```

A function to delete a given player from a grid. Returns true if the player could be deleted, false otherwise.

```c
bool grid_deletePlayer(grid_t* grid, player_t* player);
```

A function to generate an array of 'num' pointers to 2-slot arrays representing random *(row, column)* room spot coordinates in the grid.

```c
int** grid_randomRoomSpots(grid_t* grid, int num);
```

A function to place n piles of gold (where n is a randomly selected integer between min and max), containing a total of goldTotal gold pieces, at random room spots in a grid.

```c
void grid_placePiles(grid_t* grid, int min, int max, int goldTotal);
```

A function to move a player by one unit in a given direction, then modify the player's purse accordingly. Returns a status code based
on the result of the move attempt. 0 = did not move, 1 = moved, 2 = 
moved and found gold, 3 = moved and hit other player

```c
int grid_movePlayer(grid_t* grid, player_t* player, char dxn);
```

We use the following static local function to help implement grid_movePlayer.
```c
static int movePlayerHelper(grid_t* grid, player_t* player, int row change, int colChange);
```

We use the following static local function to help implement the run feature within grid_movePlayer
```c
static int runHelper(grid_t* grid, player_t* player, int row change, int colChange);
```


A function to add any gold at a player's position to their purse. Returns true if gold is collected, and false if not.

```c
bool grid_collectGold(grid_t* grid, player_t* player);
```

A function to tell us whether a given gridpoint is visible to a player.

```c
bool grid_isVisible(grid_t* grid, player_t* player, int row, int col);
```

A static function to help us determine whether a gridpoint is visible if the line between it and a player is straight.
```c
static bool visibleHelperStraight(grid_t* grid, int playerRow, int playerCol, int row, int col);
```

A static function to help us determine whether a player can see through an intersection.
```c
static bool
intersectionHelper(grid_t* grid, double currentRow, double currentCol, bool isRowCheck)
```

A function to modify a player's map to reflect what they can currently see of the grid.

```c
void grid_updatePlayerMap(grid_t* grid, player_t* player);
```

A function to update the spectator's map to reflect the state of the game.

```c
void grid_updateSpectatorMap(grid_t* grid, player_t* spectator);
```

A function to delete a grid.

```c
void grid_delete(grid_t* grid);
```

#### Detailed pseudo code

##### `grid_loadMap`:

	if the text file can be opened for reading
		create a new grid structure
		count the number of rows and columns in the text file passed into the function, add them to the grid structure as numRows and numCols
		create an empty array gridpoints of dimensions numRows * numCols
		allocate memory for each row, column in grid
		for each character in each line of the map text file
        	set the corresponding gridpoint in 'gridpoints' to have its gridChar equal to that character
    	add 'gridpoints' to the grid structure
    	return the grid structure
	else
		return null

##### `grid_addPlayer`:

	if parameters are not null or out of bounds
		identify the gridpoint in 'gridpoints' at the row and column specified
		if this gridpoint is an empty room spot
			change the player at this gridpoint to the player who is being added
			set the player's row, col to the row, col of this point
			return true
	return false

##### `grid_addPlayerRandom`:

	if parameters are not null
		use grid_randomRoomSpots to generate an array of random room spots of size 1
		take the spot at first index in the array
		set the player's row, col to be the room spot's row, col
		free the room spot 
		free the array holding the single room spot


##### `grid_deletePlayer`:

	if the parameters are not null or out of bounds
		go to the gridpoint at grid[player's row][player's column]
		if that gridpoint's playerID equals the given playerID
			set the gridpoint's playerID to null character
			set the player's row, col to -1, -1
			return true
	return false

##### `grid_randomRoomSpots`:

	given a non null grid and specified number of room spots to return
	create an array of int pointers with length numRows * numCols 
	keep track of the current index position in the pointer array
	for each row in the grid
		for each col in the grid
			if the isEmptyRoomSpot() returns true for the point there
			set current index in ptr array to store that row, col
			increase index for position in array
	create an array of pointers, whose length is the specified number of room spots
	iterate from 0 to the given number of room spots to find
		pick a random room spot from the array of all room spots from above
		for each preceding index in the array of random rooms
			if the random room spot we picked is the same spot as this index
				store that its a duplicate
				break out of this inner loop
		if the spot is a duplicate
			try again at this index in next iteration
		else
			save the unique random room spot we found in our array of pointers
	
	free each row in the array of all room spots
	free the array of all room spots
	
	return our array of random room spots


##### `grid_placePiles`:

	ensure params are not null or out of bounds
	create an int array 'piles' with its number of slots being a randomly selected number between min and max, inclusive
    for each integer between [0, goldTotal)
		pick a random slot between [0, number of slots)
        increment the count of the random slot in 'piles'
	call grid_randomRoomSpots to produce an array of *(row, column)* coordinate pairs that's the same length as 'piles'
	for each point in the array of room spots
        set the goldCount of the corresponding room spot equal to the number at this slot 'piles'
	set goldRemaining equal to goldTotal given
	free each random room row
	free the random room array
	free the array of piles


##### `grid_movePlayer`:

	// makes use of static local funcs, movePlayerHelper and runHelper
	using a switch conditional structure based on given key
		case h
			return movePlayerHelper with dir = left
		case H
			return runHelper with dir = left
		case l
			return movePlayerHelper with dir = right
		case L
			return runHelper with dir = right
		case j
			return movePlayerHelper with dir = down
		case J
			return runHelper with dir = down
		case k
			return movePlayerHelper with dir = up
		case K
			return runHelper with dir = up
		case y
			return movePlayerHelper with dir = up, left
		case Y
			return runHelper with dir = up, left
		case u
			return movePlayerHelper with dir = up, right
		case U
			return runHelper with dir = up, right
		case b
			return movePlayerHelper with dir = down, left
		case B
			return runHelper with dir = down, left
		case n
			return movePlayerHelper with dir = down, right
		case N
			return runHelper with dr = down, right
		default
			return that we did not move


##### `movePlayerHelper`

	calculate new row, col
	if they are out of bounds
		return that we did not move
	if the spot we are moving to is not a room spot or a passage spot
		return that we did not move
	if another player is on that spot
		swap places with that player
		update their row, col
		store that we collided
	update player's row, col with new values
	if grid_collectGold returns true at this spot
		return that we found gold
	if we collided
		return that we collided
	return that we moved


##### `runHelper`

	init last attempt to move's result as didNotMove
	while we are able to move, try to move
		keep track of last attempt to move's result
		if we found gold or hit a player
			break
	return the last attempt's result



##### `grid_collectGold`:

	if the parameters are not null
		identify the gridpoint that the player is located on
		if that gridpoint has goldCount greater than 0
			increment the player's purse by the gridpoint's goldCount
			decrement the grid's goldRemaining by the gridpoint's goldCount
			set the gridpoint's goldCount to 0
			return true
		else
			return false

##### `grid_isVisible`:

	ensure params are not null and within bounds
	store change in row, col from player to given row, col
	if the change in row or col is zero
		call visibleHelperStraight
	
	calculate change in row over change in column between points
	calc change in col over change in row between points

	// case 1
	if the point is below, to the right of the player
		current row = player row + change in row / change in col
		iterate through each column from player -> target col or out of bounds
			call intersectionHelper at the current row, current col
			if we can't see through the intersection
				return false
			increment current row by change in row / change in col
		current col = player's col + change in col / change in row
		iterate through each row from player -> target row or OOB
			call intersectionHelper at current row, col
			if we can't see thru intersect
				return false
			increment current col by change in col / change in row
		return true // nothing obstructed us, we can see it
	
	// case 2
	if point is below, to the left
		perform same operation as case 1, but decrement col from player to target, and since change in row / change in col will be < 0, subtract it from currentRow instead of add 
		(return false if at any point we determine view is obstructed) otherwise true
	
	// case 3
	if point is above, to the left
		perform same operation as case 2, except decrement row from player to target, substracting change in col / change in row from currentCol
		(return false if at any point we determine view is obstructed) otherwise true

	// case 4
	if point is above, to the right
		perform same operation as case 1, except decrement row from palyer to target, substracting change in col / change in row from currentCol
		(return false if at any point we determine view is obstructed) otherwise true
	
	return false

##### `visibleHelperStraight`

	if change in row is zero
		if player col < target col
			loop from player to target
				if at any point view is obstructed, return false
		else
			loop from target to player
				if at any pt view's obstructed, return false
		return true
	if change in col is zero
		if player row < target row
			loop from player to target
				if at any pt view is obstructed, return false
		else
			loop from target to palyer
				if at any pt view is obstructed, return false
		return true
	return true



##### `intersectionHelper`

	if we are checking the intersection between/on rows
		if the current row's an integer
			if we can't see through the pt at current row, col
				return false
		else
			if we cannot see through both the pt at floor(row), col and floor(row) + 1, col
				return false
	else
		if the current col's an int
			if we can't see through pt at current row, col
				return false
		else
			if we cannot see through both row, floor(col) and row, floor(col) + 1
				return false
	return true


##### `grid_updatePlayerMap`:


	for each row in 'gridpoints'
		for each gridpoint in the row
			if isVisible of the gridpoint is true
				if that gridpoint has another player on it
					set the corresponding char in the first player's map to the other player's playerID
				else
					if that gridpoint's gold count > 0
						set the player's map's corresponding char to gold symbol
					else
						set the corresponding char in the player's map to the gridChar of that gridpoint
			else
				if the player's map char is gold or another player
					set the player's map char to whatever the corresponding gridpoint char is

##### `grid_updateSpectatorMap`:

	if the grid and spectator passed in are valid
		for each row in the grid's gridpoint array
			for each gridpoint in that row
				if that gridpoint has a player on it
					set the corresponding gridpoint in the spectator's map to this player's player ID
				else
					if the gridpoint's gold count > 0
						set the spectator's char there to gold symbol
					else
						set the spec's char there to this gridpoint's gridChar

##### `grid_delete`:

	for each row in 'gridpoints'
		for each gridpoint in that row
			delete the gridpoint
	delete the array of gridpoints
	delete the grid
---


### Player

#### Data structures

The player module makes use of a `player` structure, which stores a 2d array of characters representing the portion of the map visible to the player, an int 'purse' storing the amount of gold the player has collected, a two-slot int array 'point' representing the row and column of the player, a char* representing the player's name, a char playerID representing the player's ID, and an addr_t representing the player's address. If the player is a spectator, the purse and point will be set to NULL. 

```c
typedef struct player {
  char** visibleMap; 
  char* name;
  char playerID;
  int purse;
  int row;
  int col;
  addr_t playerAddress;
} player_t;
```

#### Definition of function prototypes

A function `player_new` to allocate a new player struct.
```c
player_t* player_new(char* name, addr_t address, char playerID, int nRows, int nColumns, int startRow, int startColumn);
```

A function `player_setMap` to set the player's visibleMap
```c
void player_setMap(player_t* player, char** map);
```

Function to set player's row
```c
void player_setRow(player_t* player, int row);
```

Func to set player's column
```c
void player_setCol(player_t* player, int col);
```

A function `player_addToPurse` to add some gold to a player's purse
```c
void player_setPurse(player_t* player, int goldAmount);
```

A function `player_getName` to get the player's name.
```c
char* player_getName(player_t* player);
```

A function `player_getMap` to get the player's visibleMap
```c
char** player_getMap(player_t* player);
```

A function `player_getID` to get the player's playerID
```c
char** player_getID(player_t* player);
```

A function `player_getPoint` to get the player's point:
```c
int* player_getPoint(player_t* player);
```

A function `player_getPurse` to get the player's purse:
```c
int player_getPurse(player_t* player);
```

A function `player_getAddr` to get the player's address
```c
addr_t player_getAddr(player_t* player);
```

A function `player_mapToString` to prepare the map to be sent by the server to a client.
```c
char* player_mapToString(player_t* player, int numberOfRows, int numberOfColumns);
```

A function `player_delete` to delete an allocated player struct
```c
void player_delete(player_t* player);
```


#### Detailed pseudo code

`player_new`

	allocate a new player struct
	give the player struct the given name, address, starting row & column
	allocate player's map based on given row and column sizes
	initialize all other values to default

`player_setMap`

	if the given map and player are not NULL
		set that player's visibleMap to the given map

`player_addToPurse`

	if the given player is not NULL
		increase that player's purse by specified amount

`player_getMap`

	if the given player is not NULL
		return that player's map

`player_getPoint`

	if the given player is not NULL
		return that player's point

`player_getID`

	if the given player is not NULL
		return that player's playerID

`player_getPurse`

	if the given player is not NULL
		return that player's purse

`player_getAddr`

	if the player is not NULL
		return their address

`player_mapToString`

	allocate a `char*` whose size = (number of rows * (number of cols + 1)) + 1
	string_index = 0
	for each row
		for each column
			if the point on the player's map is the player's ID
				set the char here in the string to be @
			else
				set the char here in the string to be the char at this point on the map
			increment string_index
		set current string_index to new line char
		increment string_index
	set last index in string to be null char
	return result char


`player_delete`

	deallocate player's map
	deallocate player struct

