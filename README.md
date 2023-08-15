# Minesweeper Hack

(mozda sve stavi u jedno vreme - sadasnje, kao da sada trazis ili proslo, kao da si vec naso pa opisujes sta si radio)

This game is small enough not to be overwhelming, yet complex enough to make it a fun challenge.
While there are definitely easier ways to crack this code, I've chosen a learn-as-I-go approach which might not be the quickest, but undoubtedly helps me delve deeper into the essence of reverse engineering.

***

![Minesweeper](https://raw.githubusercontent.com/realbugdigger/MinesweeperHack/main/Minesweeper.png)

### Imported Libraries

Since minesweeper is a GUI application, it uses graphics function which prints those graphics to the screen.
Idea is to find this function and print flag whenever there is a mine.

After I open Ghidra and load the executable, in `Symbol Tree` window there are function imports from external libraries so that's where I'll look.
I see two functions that are looking interesting so that's what i'm going to examine further.

![Symbol Tree](https://raw.githubusercontent.com/realbugdigger/MinesweeperHack/main/symbol_tree.png)

#### BeginPaint

First function that I will try is `BeginPaint` in `USER32.DLL` and when looking at dissasembler looks like it has only one cross reference so it has been used only once.
Unfortunately the decompiled function where its been used is a little complex so i'll skip it for now.

#### BitBlt

I'll take a look now at `BitBlt` function in `GDI32.DLL` which is microsoft windows graphics device interface (GDI) library that enables applications to use graphics.

Looking at the [MSDN](https://learn.microsoft.com/en-us/windows/win32/api/wingdi/nf-wingdi-bitblt) and at the Ghidra decompiled function, looks like *source handle device context* is what is needed.

![Ghidra BitBtl](https://raw.githubusercontent.com/realbugdigger/MinesweeperHack/main/bitblt.png)

![draw_update](https://raw.githubusercontent.com/realbugdigger/MinesweeperHack/main/draw_update.png)
Looking at the two cross refernces for BitBlt I see that the `FUN_010026a7` has some loop and I suspect that is used for drawing tiles and minefield **only** at the start of the game so I'll rename it to `draw_initial`.
And the `FUN_01002646` which I will rename to `draw_update`, I guess that is called when GUI update is needed (e.g. displaying mines, opening tiles or setting flags).

### Debugger

I'll put breakpoint at appropriate addresses in debugger and test my hipotesis.

![draw_update_breakpoint](https://raw.githubusercontent.com/realbugdigger/MinesweeperHack/main/drawUpdate_breakpoint.png)


![draw_inital_breakpoint](https://raw.githubusercontent.com/realbugdigger/MinesweeperHack/main/drawInitial_breakpoint.png)

Aaaand my hipotesis seems to be true. `010026E2` is hit when we start the game and you can see in the next two pictures that first time we hit the breakpoint no tiles are drawn
but as I keep pressing F9 to continue execution till next breakpoint tilles are beggining to show.

![initial hit](https://raw.githubusercontent.com/realbugdigger/MinesweeperHack/main/initalHit.png)

![second hit on draw_initial](https://raw.githubusercontent.com/realbugdigger/MinesweeperHack/main/secondHit.png)

After the whole minefield has been initialized and drawn and application is waiting for our input, as soon as I press a random tile breakpoint `010026E2` has been hit (which is our breakpoint for `draw_update`).
So this confirms the hipotesis.

#### Minefield memory

Now when I take a closer look at the assembly section where draw_inital breakpoint was hit, there is `[ebx+esi]` where `esi` looks like an offset which is being updated throughout the loop and `ebx`
is a fixed value (location in memory), so as this function is printing tiles throughout the loop this fixed value can probably be start of minefield array in memory!!!

If we look memory section where `ebx` is referencing to we can see some interesting stuff.

![Minefield memory](https://raw.githubusercontent.com/realbugdigger/MinesweeperHack/main/minefield_memory.png)

`ebx` is referencing to the start of the minefield and `eax` will contain address of the current tile that is being printed.
The grey marked byte is the location of a first tile in the minefield.
We can now come to some conclussion based on this memory region:
- `10` bytes are probably some kind of row delimiters
- Since we are playing on minefield with grid size 9x9, there are 10 mines which need to be found. If you look closely there are ten `8F` in the provided memory region so this must be our mines!
- `0F` is just an empty tile
 
## Code cave

Now my goal is when starting a patched game to see all the mines locations with flags on them.
This can be accomplished with technique called [Code cave](https://en.wikipedia.org/wiki/Code_cave)

The game is performing AND operation with `0x1F` on line `10026E9` (see x64dbg screenshot snippet above), both `0x0F` and `0x8F` end up being `0x0F` which is empty tile.
 
So what i need to do is change the logic:
```
if minefield[tile] == 0x8F
   draw flag
else
   draw tile
```

The present AND instruction takes 3 bytes of opcode while my logic is more than that. That's why code cave is needed.

This is the assembly my logic needs:
```
CMP AL, 8F
JNZ 01004D08
MOV AL, 0E
PUSH DWORD PTR DS:[EAX*4+1005A20]
JMP 010026F3
```

**How did i get `0E` byte?**
By flaging tile in minefield and looking into memory to see what was before and turned into `0E`.

For my assembly code to run, I need to JMP from the original code. But as JMP opcode is more than 3 bytes, meaning I have to override not only the AND instruction but also the one following it (that's why 4th line exists in my assembly).
The remaining bytes are padded with NOPs.

![patched minesweeper](https://raw.githubusercontent.com/realbugdigger/MinesweeperHack/main/patched.png)

## Alternate ways of approaching the game

When we open the game we can see that it consists of a minefield. 
The way I would program this game is to make minefield as a matrix and making mines load at random fields everytime we start a game.
So you can start with that assumption.
Look for some kind of "random" function in PE imports and I think that can get you rolling.

When you start playing the game there are sounds being made: time ticker and mine explosion (possibly even more)
Again look in PE Imports for some "sound" function because it's very unlikely they have made their own sound function for the game and just used existing one (possibly from Windows API).

## Conclusion

I hope that this writeup has helped someone and provided them some insight about reverse engineering.
Inspiration and a lot of accumulated knowledge needed to complete this project came from https://www.begin.re/hacking-minesweeper

