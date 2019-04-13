# Baba Modding Guide

## Print Statements in Windows

It’s much easier to tell what’s going on when you can print things to the console. In Windows however it's quite hard to actually see the text that is printed. Open PowerShell, `cd` to your Baba Is You directory and start Baba using `& '.\Baba Is You.exe' | echo` and you’ll be able to see them.

I recommend adding [inspect.lua](https://github.com/kikito/inspect.lua) during development so you can log tables.

## Lily’s Modloader
[fill in how to make a mod compatible with Lily’s modloader]

## Units
All in-game objects, both regular objects and text objects, are called units. Units have an ID, which typically looks something like `2.0000000679657`. To get the complete unit structure given its ID, use `mmf.newObject(unitId)`. This returns the unit if it exists, otherwise `nil`. A unit ID of 2 corresponds to an "empty", but I'm not exactly sure how empty works.

The most useful properties of the unit structure are:

`unit.values[XPOS], unit.values[YPOS]` - the coordinates of the tile the unit is currently on. For more details see the "coordinate system" section. These parameters should not be modified directly as this could break undo and the unitmap; instead use the `update` function found in tools.lua, or `addaction`, or some existing function which results in the values being updated.

`unit.values[DIR]` - a number from 0 to 3 giving the unit’s facing direction. 0 is right, 1 is up, 2 is left and 3 is down; 4 represents no movement, i.e. the wait command. A useful array is `ndirs` which is defined in values.lua:

```lua
ndirs = {{1,0},{0,-1},{-1,0},{0,1},{0,0}}
```

You can use this to get a unit vector pointing in the same direction:

```lua
local directionVector = ndirs[unit.values[DIR] + 1]
local ox, oy = directionVector[1], directionVector[2]
```

Now `<ox, oy>` is a unit vector pointing in the unit’s direction, and the tile in front of the unit is at `(unit.values[XPOS] + ox, unit.values[YPOS] + oy)`.

`unit.strings[UNITNAME]` - this tells you what kind of unit it is. Text objects have “text_” in front of their name. Example values are "baba", "rock", "text_baba" and "text_is". When interacting with rules, it might be helpful to use `getname(unit)` instead, which returns “text” for all text objects.

## Coordinate System
The width and height of the current level are stored in `roomsizex` and `roomsizey`, and coordinates range from 0 to `roomsize<x|y> - 1`. However the playable area is surrounded by a 1-tile wide frame of "edge" tiles, so the actual playable area is `roomsizex - 2` tiles wide and `roomsizey - 2` tall. The playable area ranges from `(1,1)` to `(roomsizex - 2, roomsizey - 2)`. Increasing x moves to the right and increasing y moves down.

[Insert picture?]

## The Rule Parser and Features
The rule parser is found in the `code` function in rules.lua. It is re-run at certain times during each turn, but only if `updatecode` is 1 which usually happens because text was pushed. The output of the rule parser is a set of features. All features have the same structure:

```lua
{baseRule, conditions, unitIds}
```

All features have a base rule which always has three elements, the subject, verb and object. Some examples are `{"baba", "is", "you"}`, `{"baba", "has", "keke"}`, `{"not rock", "is", "keke"}` and `{"me", "is", "not you"}`. "And" is handled by making multiple features. For example, **rock and box is push and pull** makes four features: **rock is push**, **rock is pull**, **box is push** and **box is pull**.

Features have an array containing 0 or more conditions, which apply to the subject of the base rule. Some examples of conditions are `{"lonely", {}}`, `{"on", {"tile"}}` and `{"not near", {"baba"}}`. There is also the special "never" condition which will be explained later. Given an array of conditions, you can test if a unit satisfies them using `testcond(conditions, unitid)`, defined in conditions.lua.

Features contain an array of arrays containing the unit IDs of the text objects which gave rise to it. I think the game uses these to highlight active rules and cross out rules which were cancelled out by **x is x** or rules involving **not**.

## Default Features
There are always two features **text is push** and **level is stop**. These don't correspond to in-level rules, but they can be modified using **not**.

## All, Group and Not X
A rule with **all** in the subject, like **all is push**, will, in addition to the `{"all", "is", "push"}` feature, generate one feature for each kind of object present in the level (not including text), like **baba is push**, **rock is push** and so on. **Group** and **Not X** work similarly. These additional features do not appear in the pause menu.

## Not Cancelling
If you have a feature **rock is push** and another feature **rock is not push**, `postrules` will add the special "never" condition to the first rule, effectively disabling it. If the **not** rule has a condition (e.g. **rock on tile is not push**), it is negated and added to the first rule (giving **rock not on tile is push**).

## Querying Features
You can test if a unit has a feature (say, if it **is dance**) using `hasfeature(getname(unit), "is", "dance", unitid)`. [Also mention the featureindex, and other ways to query for features.]

## the unit type structure [name pending]
[a section explaining the structure in values.lua and what all the fields mean]

## Function Hooks
When using Lily’s modloader, it might be possible to add your functionality without needing to entirely replace files, using the hook pattern. This means your mod is less likely to break if the game is patched. It’s also easier to compose hooks from multiple mods than it is to manually merge together code changes from multiple mods. Sometimes the changes you need to make are too deep inside the existing functions, though. Example:

```lua
local vanillaMoveblock = moveblock
local function myMoveblock() {
	-- my own code
	vanillaMoveblock()
	-- more of my own code
}

function mod.load(dir)
	-- ...
	moveblock = myMoveblock
	-- ... 
end

function mod.unload(dir)
	-- ...
	moveblock = vanillaMoveblock
	-- ...
end
```

## Anatomy of a Turn
A turn starts in `command` from syntax.lua, which is never called from any other lua code; presumably it's called directly from Multimedia Fusion. This just determines which button was pressed, and then calls `movecommand` to resolve the actual turn. You'll also notice some leftover code for a co-op mode which was never completed.

`movecommand` contains all of the actual logic to resolve a turn. It starts with the infamously convoluted movement code, where most of the movement mechanics are resolved. Then it re-parses the rules if necessary. Then it resolves conversion rules (**noun is noun**). Then it re-parses rules again. Finally it calls `moveblock` which resolves the remaining mechanics. It also calls out to `MF_mapcursor` which presumably moves the cursor if it exists.

In many cases, units are not updated immediately. Instead they are queued up by `addaction` in update.lua and all executed at once in `doupdate`. This helps ensure that the order in which rules are processed does not affect the outcome.

### The Dreaded Movement Code
The movement code runs in three *takes*. A take starts by determining a list of `moving_units`:

* In the first take, it attempts to move all units which are **you** in the direction the players chose.
* In the second take, it attempts to move all units which are **move**, **chill** and which **fear** other units. Chill and fear are mechanics which the vanilla game doesn't use.
* In the third take, it attemps to move all units on a unit which **is shift** in the direction the shifter is facing.

It's possible for a unit to be a `moving_unit` more than once in the same take, for example if a unit **is move and move** or **is shift and shift**. These are all combined into a single `moving_unit` record with a `moves` property greater than 1.

Then it goes through all of the `moving_units` and somehow determines which movements are blocked by solid units and which units will be pushed, pulled or swapped as a result. There's also some code to make **shut** and **open** work. I don't really understand all of this [pull requests welcome 🙃]. This is done in `check`, `trypush` and `dopush`. The result is a `movelist` containing those `moving_units` which were not blocked, and all resulting pushes, pulls or swaps. These moves are actually executed by the `move` function.

Each `moving_unit` will move once in a take. If some of the `moving_units` needed to move more than once, that take will have multiple "sub-takes" for the additional moves. The code calls these `finaltake`s but that name is really misleading. The `moving_units` in a sub-take are those `moving_units` which were `still_moving` at the end of the previous take or sub-take. Only once there are no more `still_moving` units does it move onto the next proper take.
