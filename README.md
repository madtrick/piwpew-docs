## About

Reference documentation for [PiwPew](https://www.piwpew.com).

## Game

[PiwPew](https://www.piwpew.com) is a shoot 'em up type game. The players can move freely in the game arena and engage in battles with other players.

Your role in this game, differently to others, is not to control the player but to code it. You have to program how your player is going to behave (shoot, move, rotate, ...) during the game.

![](./assets/bots.gif)

## :warning: Warning :warning:

The game is under development and breaking changes might occur without previous announcement at any moment.

## Getting started

What do you need:

* Programming skills. Any language can be used as long as it can serialize/deserialize JSON payloads and can use WebSockets.
* Some trigonometry notions.

## Game states

A game in [PiwPew](https://www.piwpew.com) can be in two states:

* Not started. In this state players can register in the game and wait for it to start.
* Started. In this state players can join the game only if the game has been configured to allow for players to join after it has started.

## Game flow

Interactions between the players and the game engine is done via requests/responses and notifications.

* Requests are sent by players requesting to perform an action. Examples of these actions are: move forward, rotate or shoot. Responses are sent to the player by the game engine in response to their requests.
* Notifications are sent by the game engine to let players know about asynchronous events. For example when they are hit by a shot.

## Game ticks

The game runs on an endless loop. On each iteration of the loop (tick from now on) the game engine does the following:

* updates the game state. For example: move shots, determine if players are hit by a shot, do a radar scan for each player, etc.
* takes the requests it has received from the players and evaluates them with the updated game state.
* sends responses.
* sends notifications.

Players should only send one request per tick. Any further request received during a tick will override the previous one.

## Game arena

The game takes place in the game arena. Players can move freely within the boundaries of the arena but can't go beyond the borders.

The bottom left corner of the arena is called the origin, the horizontal bottom line that spans from the origin is the abcissa or "x" and the vertical line that spans from the origin is the ordinate or "y". Player coordinates are expressed relative to the origin.  The position of a player is expressed as a pair of values in the x and y axis. The arena origin has the coordinates `(0, 0)`.

Player rotation is expressed in degrees and is relative to the origin. It increases counter clockwise and the valid values are in the range `[0, 360]`.

## Player

Players have the following properties:

* rotation, the orientation of the player with respect to the origin. The rotation of the player determines the direction the player will move and also the direction of its shots.
* position, the location of the player with respect to the origin.
* life, being hit with a shot or colliding with a mine lowers the life of the player.
* tokens, see [player tokens](#player-tokens).

Players can perform the following actions:

* Move forward or backwards. See the  [move request](#move) for more details.
* Rotate 360˚. See the [rotate request](#rotate) for more details.
* Shoot. See the [shoot request](#shoot) for more details.
* Deploy a mine. See the [deploy a mine request](#deploy-mine) for more details.

Each player has a radar that scans its surroundings on each tick. Players will receive a notification with the results of the radar scan. The scan results includes the players, shoots and mines within range of the radar. It also includes other objects which are not close enough for the radar to determine their type. See the [radar scan notification](#radar-scan) for more details.

## Player tokens

Tokens are the currency of the game and as such they are used to pay for certain actions: shoot, deploy mines or apply a turbo to a movement request.

On each tick tokens are added to the balance of each player and are substracted any time one of the previous requests are sent. If the player doesn't have enough tokens to perform the request the game engine will respond with an error.

## Connecting to the game

Players have to open a websocket connection with `wss://game.piwpew.com/ws/player`.

## Game protocol

Current version: **2.1.2**

### Requests

#### Register

A player must send this request before joining a game. It registers the player in the game.

##### request payload

```json
{
  "type": "Request",
  "id": "RegisterPlayer",
  "data": {
    "game": {
      "version": "1.0.0"
    },
    "id": "player-1"
  }
}
```

| Property            | Type   | Required | Description                                                  |
| ------------------- | ------ | -------- | ------------------------------------------------------------ |
| `data.game.version` | string | yes      | The protocol of the game the player uses                     |
| `data.id`           | string | yes      | The id of the player. This value will be printed in the UI to identify the player |

The game version follows the semver principles:

* Bots requesting to register with a version which has a `minor` or a `patch` lower than the current server version will be allowed to register.
* Bots requesting to register with a version which has `major` minor than the current server version won't be allowed to register.

##### response payload

successful request

```json
{
  "type": "Response",
  "id": "RegisterPlayer",
  "success": true,
  "details": {
    "game": {
      "version": "1.0.0"
    },
    "position": {
      "x": 34.23,
      "y": 26.12
    },
    "rotation": 45.45,
    "life": 100,
    "tokens": 100
  }
}
```

| Property               | Type    | Description                          |
| ---------------------- | ------- | ------------------------------------ |
| `success`              | boolean | `true` for successful requests       |
| `details.game.version` | string  | The game version run by the server   |
| `details.position.x`   | float   | initial `x` coordinate of the player |
| `details.position.y`   | float   | initial `y` coordinate of the player |
| `details.rotation`     | float   | initial rotation of the player       |
| `details.life`         | float   | initial life of the player           |
| `details.tokens`       | float   | initial tokens of the player         |

unsuccessful request

```json
{
  "type": "Response",
  "id": "RegisterPlayer",
  "success": false,
  "details": {
  	"msg": ""
  }
}
```

| Property      | Type    | Description                       |
| ------------- | ------- | --------------------------------- |
| `success`     | boolean | `false` for unsuccessful requests |
| `details.msg` | string  | human readable error description  |

#### Move

A player must send this request when it wants to move. One request corresponds to only one movement.

If a player collides with another player or with a wall the position will not change but the query will still be considered a success.

##### request payload

```json
{
  "type": "Request",
  "id": "MovePlayer",
  "data": {
    "movement": {
      "direction": "forward",
      "withTurbo": true
    }
  }
}
```

| Property                  | Type    | Required | Description                                                  |
| ------------------------- | ------- | -------- | ------------------------------------------------------------ |
| `data.movement.direction` | string  | yes      | Wether the player should move forward or backward. Valid values are `forward` or `backward` |
| `data.movement.withTurbo` | boolean | no       | Wether the turbo should be applied to the movement so it's faster. Note that using the turbo requires additional tokens. |

##### response payload

successful request

```json
{
  "type": "Response",
  "id": "MovePlayer",
  "success": true,
  "data": {
    "component": {
      "details": {
        "position": {
          "x": 100.0,
          "y": 100.1
        },
        "tokens": 111
      }
    },
    "request": {
      "cost": 10,
      "withTurbo": true
    }
  }
}
```

| Property                            | Type    | Description                                              |
| ----------------------------------- | ------- | -------------------------------------------------------- |
| `success`                           | boolean | `true` for successful requests                           |
| `data.component.details.position.x` | float   | `x` coordinate of the player position after the movement |
| `data.component.details.position.y` | float   | `y` coordinate of the player position after the movement |
| `data.component.details.tokens`     | integer | remaining number of tokens after the request             |
| `data.request.cost`                 | integer | request cost in tokens                                   |
| `data.request.withTurbo`            | boolean | wether the turbo was applied or not                      |

unsuccessful request

```json
{
  "type": "Response",
  "id": "MovePlayer",
  "success": false,
  "details": {
    "msg": ""
  }
}
```

| Property      | Type    | Description                       |
| ------------- | ------- | --------------------------------- |
| `success`     | boolean | `false` for unsuccessful requests |
| `details.msg` | string  | human readable error description  |

#### Rotate

A player must send this request if it wants to change its orientation. One rotation request corresponds to only one rotation being made. The requested rotation is the new value of the player rotation, it's not added or substracted from the current rotation.

##### request payload

```json
{
  "type": "Request",
  "id": "RotatePlayer",
  "data": {
    "rotation": 50
  }
}
```

| Property                 | Type  | Required | Description                                                  |
| ------------------------ | ----- | -------- | ------------------------------------------------------------ |
| `data.movement.rotation` | float | yes      | Value indicating the new rotation of the player. Valid values are in the range `[0, 360]`. |

##### response payload

successful request

```json
{
  "type": "Response",
  "id": "RotatePlayer",
  "success": true,
  "data": {
    "component": {
      "details": {
      	"rotation": 180.0,
      	"tokens": 33
    	}
    },
    "request": {
      "cost": 3
    }
  }
}
```

| Property                          | Type    | Description                                              |
| --------------------------------- | ------- | -------------------------------------------------------- |
| `success`                         | boolean | `true` for successful requests                           |
| `data.component.details.rotation` | float   | `x` coordinate of the player position after the movement |
| `data.component.details.tokens`   | integer | remaining number of tokens after the request             |
| `data.request.cost`               | integer | request cost in tokens                                   |

unsuccessful request

```json
{
  "type": "Response",
  "id": "RotatePlayer",
  "success": false,
  "details": {
    "msg": ""
  }
}
```

| Property      | Type    | Description                       |
| ------------- | ------- | --------------------------------- |
| `success`     | boolean | `false` for unsuccessful requests |
| `details.msg` | string  | human readable error description  |

#### Shoot

A player must send this request when it wants to shoot. One shoot request corresponds to only one shot being fired. The direction of the shot will the same as the player's when the shot was fired.

##### request payload

```json
{
  "type": "Request",
  "id": "Shoot"
}
```

##### response payload

successful request

```json
{
  "type": "Response",
  "id": "Shoot",
  "success": true,
  "data": {
    "component": {
      "details": {
        "tokens": 100
      }
    },
    "request": {
      "cost": 6
    }
  }
}
```

| Property                        | Type    | Description                                  |
| ------------------------------- | ------- | -------------------------------------------- |
| `success`                       | boolean | `true` for successful requests               |
| `data.component.details.tokens` | integer | remaining number of tokens after the request |
| `data.request.cost`             | integer | request cost in tokens                       |

unsuccessful request

```json
{
  "type": "Response",
  "id": "Shoot",
  "success": false,
  "details": {
    "msg": ""
  }
}
```

| Property      | Type    | Description                       |
| ------------- | ------- | --------------------------------- |
| `success`     | boolean | `false` for unsuccessful requests |
| `details.msg` | string  | human readable error description  |

#### Deploy mine

A player must send this request if it wants to deploy a mine. One mine deployment request corresponds to only one mine being deployed.

Mines are deployed at 180˚ of the current rotation or put in a different way opposite to the current direction of the player.

Players can suffer damage if they collide with their own mines.

##### request payload

```json
{
  "type": "Request",
  "id": "DeployMine"
}
```

##### response payload

successful request

```json
{
  "type": "Response",
  "id": "DeployMine",
  "success": true,
  "data": {
    "component": {
      "details": {
        "tokens": 123
      }
    },
    "request": {
      "cost": 10
    }
  }
}
```

| Property                        | Type    | Description                                  |
| ------------------------------- | ------- | -------------------------------------------- |
| `success`                       | boolean | `true` for successful requests               |
| `data.component.details.tokens` | integer | remaining number of tokens after the request |
| `data.request.cost`             | integer | request cost in tokens                       |

unsuccessful request

```json
{
  "type": "Response",
  "id": "DeployMine",
  "success": false,
  "details": {
  	"msg": ""
  }
}
```

| Property      | Type    | Description                       |
| ------------- | ------- | --------------------------------- |
| `success`     | boolean | `false` for unsuccessful requests |
| `details.msg` | string  | human readable error description  |

### Notifications

Notifications are messages sent by the game engine which are not the response to a player request.

#### Join game

Notification sent to the player once it has joined a game. Sent only after successfully registering and once the game starts (or if the game was already started when the player registered).

##### notification payload

```json
{
  "type": "Notification",
  "id": "JoinGame",
  "details": {
    "game": {
      "settings": {
        "playerSpeed": 10,
        "shotSpeed": 11,
        "arenaWidth": 100,
        "arenaHeight": 100,
        "playerRadius": 16,
        "radarScanRadius": 80,
        "turboMultiplier": 2
      }
    }
  }
}
```

| Property                                | Type    | Description                                                  |
| --------------------------------------- | ------- | ------------------------------------------------------------ |
| `details.game.settings.playerSpeed`     | integer | Speed of the player when moving (backward or forward)        |
| `details.game.settings.shotSpeed`       | integer | Speed of the shot                                            |
| `details.game.settings.turboMultiplier` | integer | Speed multiplier applied when movements use the `withTurbo` option |
| `details.game.settings.playerRadius`    | integer | Radius of the player                                         |
| `details.game.settings.radarScanRadius` | integer | Radius of the radar scan                                     |
| `details.game.settings.arenaWidth`      | integer | Width of the arena                                           |
| `details.game.settings.arenaHeight`     | integer | Height of the arena                                          |

#### Tick

Notification sent to the player after the game state has been updated, the requests have been processed and their responses as well as any other notification have been sent. Once this notification has been received players can send new requests without having to worry about overriding previously sent requests.

The game sends this notification as soon as the connection with the player has been stablished.

##### notification payload

```json
{
  "type": "Notification",
  "id": "Tick"
}
```

#### Radar scan

Notification which includes the results of the radar scan. The radar scans 360˚around each player. The game sends only one radar scan notification per tick.

##### notification payload

```json
{
  "type": "Notification",
  "id": "RadarScan",
  "data": {
    "players": [{
      "id": "player-1",
      "rotation": 34.0,
      "position": {
        "x": 11.34,
        "y": 22.45
      }
  	}],
    "unknown": [
      {
      	"position": {
          "x": 23,
          "y": 32
        }
    	}
    ],
    "mines": [
      {
      	"position": {
          "x": 23,
          "y": 32
        }
    	}
    ],
    "shots": [
      {
      	"rotation": 10,
      	"position": {
        	"x": 10,
          "y": 14
      	}
    	}
    ]
  }
}
```

| Property       | Type                           | Description                                               |
| -------------- | ------------------------------ | --------------------------------------------------------- |
| `data.players` | array of scanned players       | Scanned players in the surroundings of the player         |
| `data.shot`    | array of scanned shots         | Scanned shots in the surroundings of the player           |
| `data.mines`   | array of scanned mines         | Scanned mines in the surroundings of the player           |
| `data.unknown` | array of unknown scanned items | Scanned items which are not close enough to be identified |

##### scanned player

| Property     | Type   | Description                                                  |
| ------------ | ------ | ------------------------------------------------------------ |
| `id`         | string | `id` of the scanned player                                   |
| `rotation`   | float  | rotation of the player at the moment of the scan             |
| `position.x` | float  | x coordinate of the player's position at the moment of the scan |
| `position.y` | float  | y coordinate of the player's position at the moment of the scan |

##### scanned shot

| Property     | Type  | Description                                                  |
| ------------ | ----- | ------------------------------------------------------------ |
| `rotation`   | float | rotation of the shot                                         |
| `position.x` | float | x coordinate of the shot's position at the moment of the scan |
| `position.y` | float | y coordinate of the shot's position at the moment of the scan |

##### scanned mine

| Property     | Type  | Description                         |
| ------------ | ----- | ----------------------------------- |
| `position.x` | float | x coordinate of the mine's position |
| `position.y` | float | y coordinate of the mine's position |

##### scanned unknown

| Property     | Type  | Description                                                  |
| ------------ | ----- | ------------------------------------------------------------ |
| `position.x` | float | x coordinate of the unknown object's position at the moment of the scan |
| `position.y` | float | y coordinate of the unknown object's position at the moment of the scan |

#### Hit

Notification sent by the game to each player hit by a shot or a mine. One notification is sent per hit.

##### notification payload

```json
{
  "type": "Notification",
  "id": "Hit",
  "data": {
  	"damage": 1
  }
}
```

| Property      | Type  | Description                                                  |
| ------------- | ----- | ------------------------------------------------------------ |
| `data.damage` | float | Damage taken by the player. The life of the player will be reduce in an amount equal to the value of the damage |
