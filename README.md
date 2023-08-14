# Minesweeper Hack

(mozda sve stavi u jedno vreme - sadasnje, kao da sada trazis ili proslo, kao da si vec naso pa opisujes sta si radio)

This game is small enough not to be overwhelming, yet complex enough to make it a fun challenge.
While there are definitely easier ways to crack this code, I've chosen a learn-as-I-go approach which might not be the quickest, but undoubtedly helps me delve deeper into the essence of reverse engineering.

***

![Minesweeper](https://raw.githubusercontent.com/realbugdigger/MinesweeperHack/main/Minesweeper.png)

When we open the game we can see that it consists of a minefield. 
The way I would program this game is to make minefield as a matrix and obviously mines should be loaded at random fields everytime we start a game.
So I am going to start with that assumption.

Since minesweeper is a GUI application, it uses graphics function which prints those graphics to the screen.
Idea is to find this function and pass it the argument corresponding to flag whenever there is a mine - we're done.

After I opened Ghidra dissasembler and loaded the executable, in `Symbol Tree` window there are function imports from external libraries so that's where i looked.
I see two functions that are looking interesting so that's what i'm going to examine further.

![Symbol Tree](https://raw.githubusercontent.com/realbugdigger/MinesweeperHack/main/symbol_tree.png)

First function that I tried is `BeginPaint` in `USER32.DLL` and when looking at dissasembler looks like it has only one cross reference so it has been used only once.
Unfortunately the decompiled function where its been used is a little to complex so i'll skip it for now and get back to it later if stuck.

I'll take a look now at `BitBlt` function in `GDI32.DLL` which is microsoft windows graphics device interface (GDI) library that enables applications to use graphics.

Looking at the [MSDN](https://learn.microsoft.com/en-us/windows/win32/api/wingdi/nf-wingdi-bitblt) and at the Ghidra decompiled function output looks like *source handle device context* is what is needed.

![Ghidra BitBtl](https://raw.githubusercontent.com/realbugdigger/MinesweeperHack/main/bitblt.png)

![draw_update](https://raw.githubusercontent.com/realbugdigger/MinesweeperHack/main/draw_update.png)
Looking at the two cross refernces for BitBlt I see that the `FUN_010026a7` has some loop and I suspect that that is used for drawing tiles and minefield **only** at the start of the game so I'll rename it to `draw_initial`.
And the `FUN_01002646` which I will rename to `draw_update` I guess that is called when GUI update is needed (displaying mines, opening tiles or setting flags).

I'll put breakpoint at appropriate addresses in debugger and test my hipotesis.

![draw_update_breakpoint](https://raw.githubusercontent.com/realbugdigger/MinesweeperHack/main/drawUpdate_breakpoint.png)


![draw_inital_breakpoint](https://raw.githubusercontent.com/realbugdigger/MinesweeperHack/main/drawInitial_breakpoint.png)

Aaaand my hipotesis turns out to be true. `010026E2` is hit when we start the game and you can see in the next two pictures that first time we hit the breakpoint no tiles are drawn
but as I keep pressing F9 to continue execution till next breakpoint tilles are beggining to show.

![initial hit]()

![second hit on draw_initial]()

After the whole minefield has been initialized and drawn and application is waiting for our input as soon as I press a random tile, breakpoint `010026E2` has been hit (which is our breakpoint for `draw_update`).
So this confirms our theory.

Now when i take a closer look at the assembly section where draw_inital breakpoint was hit, there is `[eax+esi]` which to me looks like `esi` is an offset which is being updated throughout the loop and `eax`
has a fixed location in memory, so this fixed location can probably be start of minefield array in memory!!!

If we look memory section where `eax` is pointing to we can see some interesting stuff.

![Minefield memory]()

