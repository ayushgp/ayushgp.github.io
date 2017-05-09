---
layout: post
title: Tic Tac Toe using Socket.IO
---

I'll teach you how to create a real time [Tic-Tac-Toe game](https://www.wikiwand.com/en/Tic-tac-toe) using [socket.IO](http://socket.io). 

_Note: This project uses [skeleton](http://getskeleton.com/), a minimal css framework to keep the UI pretty. It's not required to build this app._
The code for this application is available on [GitHub](https://github.com/ayushgp/tic-tac-toe-socket-io) and you can check out the [demo on Heroku](https://tic-tac-toe-realtime.herokuapp.com).

## Prerequisites

- Socket.IO is a framework on Node.js. You'll need node installed to follow along this tutorial. Follow the instructions from [node.js](https://nodejs.org/en/) to install it. 

- You should know intermediate JavaScript. You should be comfortable with things like object prototypes, event handlers, callbacks, IIFE, etc.  

- You should also have some [jQuery](http://jquery.com) knowledge to follow along this tutorial. If you don't, you can check out [Beginners Guide to DOM Selection with jQuery](https://www.sitepoint.com/beginners-guide-dom-selection-jquery/). 

## Getting started

Socket.IO uses events for communication between client and server. We open connections on the client to connect to the server. Once connected to the server, we have a two way pipeline through which both client and server can send data to each other. The main advantage of this is that the server can send data to the client without the client even requesting it! This is particularly helpful for scenarios where we need to push data to the client based on some events. 

We'll be creating our own events to help build our game. Here is a brief overview of how our game will work. One player will **create** the game (Player 1 or X) and will provide the other player with a game Id. The second player will join the game (Player 2 or O). This game will exist till we have a winner or the game is tied.

Now lets get started by creating a directory for our app and installing the dependencies. 

###Initialize your application using npm
Create a new directory called `tic-tac-toe`. Create `package.json`  file in this directory with the following content: 
```json
{
  "name": "tic-tac-toe",
  "version": "1.0.0",
  "description": "A realtime online multiplayer game.",
  "main": "index.js",
  "scripts": {
    "start": "node index.js"
  },
  "author": "Your Name Here",
  "license": "ISC",
  "dependencies": {
    "express": "^4.14.0",
    "jquery": "^3.1.1",
    "skeleton-css": "^2.0.4",
    "socket.io": "^1.7.1"
  }
}
```

Now run `npm install` to install the following dependencies we defined in the `package.json` file:

* **Express**: Framework to set up our server.
* **jQuery**: Library easily manipulate the DOM.
* **Skeleton**: CSS framework for fonts and layout.
* **Socket.IO**: Real time framework for communicating with players.

## Setting up the server
We'll create a file `index.js` in the root of `tic-tac-toe`, for the server code. This code will handle all events we receive from the clients, manage game rooms and serve static files to the clients.

Add the following to the `index.js` file:
```javascript
var express = require('express');
var app = express();
var server = require('http').Server(app);
var io = require('socket.io')(server);

var rooms = 0;

app.use(express.static('.'));

app.get('/', function (req, res) {
  res.sendFile(__dirname + '/game.html');
});

server.listen(5000);
```

We've created the server. Lets create a basic HTML file, `game.html`, we'll be using for our UI. Add the following code to it: 
```html
<!DOCTYPE html>
<html>
  <head>
    <title>Tic Tac Toe</title>
    <link rel="stylesheet" href="node_modules/skeleton-css/css/skeleton.css">
  </head>
  <body>
    <div class="container">
    </div>
    <script src="node_modules/jquery/dist/jquery.min.js"></script>
    <script src="/socket.io/socket.io.js"></script>
  </body>
</html>
```
Now we need to handle what happens whenever any client connects to our server. Socket.IO provides an event called `connection` that it automatically triggers. Add the following code to the `index.js` file above `server.listen` call to handle connection events: 
```javascript
io.on('connection', function(socket){
	console.log('A user connected!'); // We'll replace this with our own events
});
```
Whenever a user creates a game, we need to keep track of it. Doing so manually is tedious. Socket.IO allows us to create **rooms**. We'll use rooms to create pairs of users. All communication that happens in a room is limited to the people who've joined it. 

### Internals of our server
The following steps will take place before players can actually play a game: 

1. The server is making the first player join the game room as soon as the user creates a game. 
2. Then the server sends the user's browser an event `newGame` with the room Id so that the user can tell the second player this Id to join the game. 
3. When the second player will join the game, the second player will receive the `joinGame` event. The server will search for the room the user wants to join. The server will allow the user to join only if it exists and has only one person in it. 
4. Then the server tells the players about their roles and emit events `player1` and `player2` accordingly.

When users will play their turns, the server will receive the `playTurn` event. The server will send `turnPlayed` event to the second player about this move so that both users' UI are updated and the second player can make a move.

The user who plays the turn checks if he won or game is tied. If he wins, the socket emits `gameEnded` event which then is handled to let the other user know about the status of the game. 


### Event handlers
Now let's implement the above plan and get our server working. We'll create event handlers for `createGame`, `joinGame`, `playTurn` and `gameEnded` events. 
Replace the log statement within the connection event handler with the following code: 
```javascript
/**
 * Create a new game room and notify the creator of game. 
 */
socket.on('createGame', function(data){
  socket.join('room-' + ++rooms);
  socket.emit('newGame', {name: data.name, room: 'room-'+rooms});
});

/**
 * Connect the Player 2 to the room he requested. Show error if room full.
 */
socket.on('joinGame', function(data){
  var room = io.nsps['/'].adapter.rooms[data.room];
  if( room && room.length == 1){
    socket.join(data.room);
    socket.broadcast.to(data.room).emit('player1', {});
    socket.emit('player2', {name: data.name, room: data.room })
  }
  else {
    socket.emit('err', {message: 'Sorry, The room is full!'});
  }
});

/**
 * Handle the turn played by either player and notify the other. 
 */
socket.on('playTurn', function(data){
  socket.broadcast.to(data.room).emit('turnPlayed', {
    tile: data.tile,
    room: data.room
  });
});

/**
 * Notify the players about the victor.
 */
socket.on('gameEnded', function(data){
  socket.broadcast.to(data.room).emit('gameEnd', data);
});
```
The following information will help you understand how `socket` handles events and takes care of communication:

* `socket.on('event', function)` listens for events by each connected client and executes the associated function when events are triggered.
* `socket.emit('event', data)` emits the event to the client who invoked the event containing this call. 
* `socket.broadcast.to(room)` broadcasts the event to everyone in the room except the person who sent the event which triggered this function. For example, in the `gameEnded` event handler, the person who emitted this event won't be sent the `gameEnd` event. Everyone else in that room will receive this event.

## Setting up the UI
Now that we've set up our server, its time for us to create the UI of the game. We'll first create a form that allows users to create and join games. Edit the `game.html` file and add the following code between the body tags:

```html
<div class="container">
  <div class="menu">
    <h1>Tic - Tac - Toe</h1>
    <h3>How To Play</h3>
    <ol>
      <li>Player 1: Create a new game by entering the username</li>
      <li>Player 2: Enter another username and the room id that is displayed on first window.</li>
      <li>Click on join game. </li>
    </ol>
    <h4>Create a new Game</h4>
    <input type="text" name="name" id="nameNew" placeholder="Enter your name" required>
    <button id="new">New Game</button>
    <br><br>
    <h4>Join an existing game</h4>
    <input type="text" name="name" id="nameJoin" placeholder="Enter your name" required>
    <input type="text" name="room" id="room" placeholder="Enter Game ID" required>
    <button id="join">Join Game</button>
  </div>
</div>
```
We also need to create the game board, include the jQuery script and our own script that'll run our game. Enter the following code after the closing `div` tag(of class menu).
```html
<div class="gameBoard">
  <h2 id="userHello"></h2>
  <h3 id="turn"></h3>
  <table class="center">
    <tr>
      <td><button class="tile" id="button_00"></button></td>
      <td><button class="tile" id="button_01"></button></td>
      <td><button class="tile" id="button_02"></button></td>
    </tr>
    <tr>
      <td><button class="tile" id="button_10"></button></td>
      <td><button class="tile" id="button_11"></button></td>
      <td><button class="tile" id="button_12"></button></td>
    </tr>
    <tr>
      <td><button class="tile" id="button_20"></button></td>
      <td><button class="tile" id="button_21"></button></td>
      <td><button class="tile" id="button_22"></button></td>
    </tr>
  </table>
</div>
<script src="node_modules/jquery/dist/jquery.min.js"></script>
<script src="/socket.io/socket.io.js"></script>
<script src="main.js"></script>	
```

## Creating a new game
We'll set our `main.js` file that will handle all of our game logic. Create a new file, `main.js` in the root of `tic-tac-toe` and enter the following code in it:

```javascript
(function(){

  // Types of players
  var P1 = 'X', P2 = 'O';
  var socket = io.connect('http://localhost:5000'),
    player,
    game;

  /**
   * Create a new game. Emit newGame event.
   */
  $('#new').on('click', function(){
    var name = $('#nameNew').val();
    if(!name){
      alert('Please enter your name.');
      return;
    }
    socket.emit('createGame', {name: name});
    player = new Player(name, P1);
  });

  /** 
   *  Join an existing game on the entered roomId. Emit the joinGame event.
   */ 
  $('#join').on('click', function(){
    var name = $('#nameJoin').val();
    var roomID = $('#room').val();
    if(!name || !roomID){
      alert('Please enter your name and game ID.');
      return;
    }
    socket.emit('joinGame', {name: name, room: roomID});
    player = new Player(name, P2);
  });
})();
```
We've created an [IIFE](https://en.wikipedia.org/wiki/Immediately-invoked_function_expression) that runs as soon as it is loaded. We'll put all our code within this IIFE. 

- We have created the connection to the server and stored the object in the `socket` variable. 
- We've also attached click event listeners to both of our buttons, **Create Game** and **Join Game**. 
- We've put validation checks and notified the server about either of these events.

### Player Object
Now we need to create 2 objects `Game` and `Player`. These objects will handle various properties of the game and the player. They'll help us keep our code organised. 

Let us start by defining the `Player` object. Enter the following code after the variable declarations:

```javascript
/**
 * Player class
 */
var Player = function(name, type){
  this.name = name;
  this.type = type;
  this.currentTurn = true;
  this.movesPlayed = 0;
}

/**
 * Create a static array that stores all possible win combinations
 */
Player.wins = [7, 56, 448, 73, 146, 292, 273, 84];

/**
 * Set the bit of the move played by the player
 */
Player.prototype.updateMovesPlayed = function(tileValue){
  this.movesPlayed += tileValue;
}

Player.prototype.getMovesPlayed = function(){
  return this.movesPlayed;
}

/**
 * Set the currentTurn for player to turn and update UI to reflect the same.
 */
Player.prototype.setCurrentTurn = function(turn){
  this.currentTurn = turn;
  if(turn){
    $('#turn').text('Your turn.');
  }
  else{
    $('#turn').text('Waiting for Opponent');
  }
}

Player.prototype.getPlayerName = function(){
  return this.name;
}

Player.prototype.getPlayerType = function(){
  return this.type;
}

/**
 * Returns currentTurn to determine if it is the player's turn.
 */
Player.prototype.getCurrentTurn = function(){
  return this.currentTurn;
}
```

### Game Class
Now let us create the `Game` class. We'll be using the above methods(`Player` class) to keep track of turns, tiles played, etc.

```javascript
/**
 * Game class
 */
var Game = function(roomId){
  this.roomId = roomId;
  this.board = [];
  this.moves = 0;
}

/**
 * Create the Game board by attaching event listeners to the buttons. 
 */
Game.prototype.createGameBoard = function(){
  for(var i=0; i<3; i++) {
    this.board.push(['','','']);
    for(var j=0; j<3; j++) {
      $('#button_' + i + '' + j).on('click', function(){
        if(!player.getCurrentTurn()){
          alert('Its not your turn!');
          return;
        }

        if($(this).prop('disabled'))
          alert('This tile has already been played on!');

        var row = parseInt(this.id.split('_')[1][0]);
        var col = parseInt(this.id.split('_')[1][1]);

        //Update board after your turn.
        game.playTurn(this);
        game.updateBoard(player.getPlayerType(), row, col, this.id);

        player.setCurrentTurn(false);
        player.updateMovesPlayed(1 << (row * 3 + col));

        game.checkWinner();
        return false;
      });
    }
  }
}

/**
 * Remove the menu from DOM, display the gameboard and greet the player.
 */
Game.prototype.displayBoard = function(message){
  $('.menu').css('display', 'none');
  $('.gameBoard').css('display', 'block');
  $('#userHello').html(message);
  this.createGameBoard();
}

/**
 * Update game board UI
 */
Game.prototype.updateBoard = function(type, row, col, tile){
  $('#'+tile).text(type);
  $('#'+tile).prop('disabled', true);
  this.board[row][col] = type;
  this.moves ++;
}

Game.prototype.getRoomId = function(){
  return this.roomId;
}

/**
 * Send an update to the opponent to update their UI.
 */
Game.prototype.playTurn = function(tile){
  var clickedTile = $(tile).attr('id');
  var turnObj = {
    tile: clickedTile,
    room: this.getRoomId()
  };
  // Emit an event to update other player that you've played your turn.
  socket.emit('playTurn', turnObj);
}

/**
 *
 * To determine a win condition, each square is "tagged" from left
 * to right, top to bottom, with successive powers of 2.  Each cell
 * thus represents an individual bit in a 9-bit string, and a
 * player's squares at any given time can be represented as a
 * unique 9-bit value. A winner can thus be easily determined by
 * checking whether the player's current 9 bits have covered any
 * of the eight "three-in-a-row" combinations.
 *
 *     273                 84
 *        \               /
 *          1 |   2 |   4  = 7
 *       -----+-----+-----
 *          8 |  16 |  32  = 56
 *       -----+-----+-----
 *         64 | 128 | 256  = 448
 *       =================
 *         73   146   292
 *
 *  We have these numbers in the Player.wins array and for the current 
 *  player, we've stored this information in player.movesPlayed.
 */
Game.prototype.checkWinner = function(){		
  var currentPlayerPositions = player.getMovesPlayed();
  Player.wins.forEach(function(winningPosition){
    // We're checking for every winning position if the player has achieved it.
    // Keep in mind that we are using a bitwise AND here not a logical one.PlaysArr
    if(winningPosition & currentPlayerPositions == winningPosition){
      game.announceWinner();
    }
  });

  var tied = this.checkTie();
  if(tied){
    socket.emit('gameEnded', {room: this.getRoomId(), message: 'Game Tied :('});
    alert('Game Tied :(');
    location.reload();	
  }
}

/**
 * Check if game is tied
 */
Game.prototype.checkTie = function(){
  return this.moves >= 9;
}

/**
 * Announce the winner if the current client has won. 
 * Broadcast this on the room to let the opponent know.
 */
Game.prototype.announceWinner = function(){
  var message = player.getPlayerName() + ' wins!';
  socket.emit('gameEnded', {room: this.getRoomId(), message: message});
  alert(message);
  location.reload();
}

/**
 * End the game if the other player won.  
 */
Game.prototype.endGame = function(message){
  alert(message);
  location.reload();
}
```

- When starting a new game, we first remove the current menu from the DOM and add our game board using the `displayBoard` function. 
- Then we are attaching event listeners to every tile on the board using the `createGameBoard` function.
- Every time a player plays a move, we update the board(`updateBoard` function) and notify the opponent by emitting the `turnPlayed` event. 
- At the end of every turn, we check if a player has won the game or it is tied. We notify the players accordingly using the `announceWinner` function. The `endGame` handles the event emitted by `announceWinner`. 

### Front end event handlers
Now we need to add event handlers for the events we are broadcasting/emitting from our server. The events are: 

 * `newGame`: Create a new game room and notify the creator of game.
 * `player1`: Notify the creator of game that other player has joined. 
 * `player2`: Notify player 2 that game has started.
 * `err`: Notify player that room they are trying to join is full. 
 * `turnPlayed`: Notify either player that opponent has played their turn. 
 * `gameEnd`: Notify players that game has ended and which player has won.

Lets create the event handlers for these events to our *main.js* file within the IIFE:

```javascript
/** 
 * New Game created by current client. 
 * Update the UI and create new Game var.
 */
socket.on('newGame', function(data){
  var message = 'Hello, ' + data.name + 
    '. Please ask your friend to enter Game ID: ' +
    data.room + '. Waiting for player 2...';

  // Create game for player 1
  game = new Game(data.room);
  game.displayBoard(message);		
});

/**
 * If player creates the game, he'll be P1(X) and has the first turn.
 * This event is received when opponent connects to the room.
 */
socket.on('player1', function(data){		
  var message = 'Hello, ' + player.getPlayerName();
  $('#userHello').html(message);
  player.setCurrentTurn(true);
});

/**
 * Joined the game, so player is P2(O). 
 * This event is received when P2 successfully joins the game room. 
 */
socket.on('player2', function(data){
  var message = 'Hello, ' + data.name;

  //Create game for player 2
  game = new Game(data.room);
  game.displayBoard(message);
  player.setCurrentTurn(false);	
});	

/**
 * Opponent played his turn. Update UI.
 * Allow the current player to play now. 
 */
socket.on('turnPlayed', function(data){
  var row = data.tile.split('_')[1][0];
  var col = data.tile.split('_')[1][1];
  var opponentType = player.getPlayerType() == P1 ? P2 : P1;
  game.updateBoard(opponentType, row, col, data.tile);
  player.setCurrentTurn(true);
});

/**
 * If the other player wins or game is tied, this event is received. 
 * Notify the user about either scenario and end the game. 
 */
socket.on('gameEnd', function(data){
  game.endGame(data.message);
  socket.leave(data.room);
})

/**
 * End the game on any err event. 
 */
socket.on('err', function(data){
  game.endGame(data.message);
});
```

## Playing the game
Phew, that was a lot of code! Let us run and play the game now. To do so, run the following command in your terminal: 

```bash 
$ npm start
```

Our server will start running on [http://localhost:5000/](http://localhost:5000/). Open 2 windows and go to this address to play the game.

## Conclusion
We've created a multiplayer game with our own server in less than 400 lines of code. Pretty neat, huh? Please note that this is a minimal implementation for demonstrating how you can use Socket.IO and it doesn't cover every corner case you might run into while running something like this in production. 

You can try the game [here](http://tic-tac-toe-realtime.herokuapp.com). If you find any issues in the code, file an issue on its [repository](https://github.com/ayushgp/tic-tac-toe-socket-io). If you want, you can submit a PR as well.
