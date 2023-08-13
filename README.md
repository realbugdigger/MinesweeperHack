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
Unfortunately the decompiled function where its been used is a little to complex so i'll skip it for now and get back to it later if needed.

I'll take a look now at `BitBlt` function in `GDI32.DLL` which is microsoft windows graphics device interface (GDI) library that enables applications to use graphics.

Looking at the [MSDN](https://learn.microsoft.com/en-us/windows/win32/api/wingdi/nf-wingdi-bitblt) and at the Ghidra decompiled function output looks like source handle device context is what is needed.

![Ghidra BitBtl]()

