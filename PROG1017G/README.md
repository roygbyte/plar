# Overview #

This file contains documentation towards NBCC class PROG1017G. 

## Competencies ##

### [Competency 1](#c-1) ###
Utilize an object-oriented language implemented using a development environment and associated tools.

Demonstrate how to create programs using the integrated development environment and tools, including: 

1. [Toolbox](#c-1-toolbox) 
2. [Solution Explorer](#c-1-solution-explorer)
3. [Properties](#c-1-properties)
4. [Breakpoints](#c-1-breakpoints)
5. [Debugging tools](#c-1-debugging)
6. [Use of menu](#c-1-use-of-menu)
7. [Use of Shortcut icons](#c-1-use-of-shortcut-icons)

### Competency 2 ###
Apply problem solving abilities in developing program designs from specifications.

This competency includes has 2 objectives to be demonstrated.

1. [Demonstrate how to develop algorithms using pseudocode and flow charts to solve problems.](#c-2-1) (WIP)
2. [Demonstrate how to translate specifications to program code.](#c-2-2) (WIP)

### [Competency 3](#c-3) ###
Write programs according to specifications using fundamental programming constructs including selection structures, loops, sub procedures and functions.

This competency includes 3 objectives to be demonstrated.

1. [Demonstrate use of more than one selection structure.](#c-3-1)
2. [Demonstrate use of more than one loop.](#c-3-2)
3. [Demonstrate how to use functions by creating your own methods to send and return data.](#c-3-3)


# Overview of Project #

In my spare time, I have been working on a plugin for my KOReader-powered Kobo. [KOReader](https://koreader.rocks), for the unitiated, is an open-source document viewer for e-ink devices. When I got my Kobo in November 2020, the promise of developing my own plugins using KOReader was far more exciting to me than actually, you know... reading.

KOReader's front-end is written in Lua, which is a scripting language. Using a scripting language for building user-interface widgets and other front-end services was a great choice for this project. Scripting languages do not need to be compiled. Instead, they are interpreted a runtime. For me, as a developer, this means that I can work in an iterative manner, writting code and testing it quickly without having to compile. It also means I don't have to worry (too much) about low-level system resources like memory. 

My first plugin for KOReader was a [weather forecaster](https://github.com/roygbyte/weather.koplugin). Weather is fetched from [WeatherAPI](https://weatherapi.com) using a postal code set by the user. There are a number of views for seeing weekly forecasts, daily forecasts, and hourly forecasts. By no means is it very sophisticated. In fact, it's quite ugly... most of the views are constructed as two-column lists. I'd love to make it more pretty in the future by writing fancy views with icons and whatnot. But that's not what I'm up to now.

My current KOReader project is a crossword plugin. I've been developing it for a number of weeks, picking on this or that feature whenever I am in the mood. It's a fairly complicated endeavor and has required some clever thinking about how to best render views and how to store game data in an intuitive, non-spaghetti manner. In the next section I've outline the project structure vis-a-vis the files that make up the various composers, views, and data models.

# Project Structure #

Currently, the code is distributed into the following files:

- main.lua
  - gameview.lua
	- puzzle.lua
		- solve.lua
	- gridview.lua
		- gridrow.lua
		- gridsquare.lua
- softkeyboard.lua

`main.lua` extends `WidgetContainer`, which is the KOReader base class for making plugins. `softkeyboard.lua`, `gameview.lua`, `gridview.lua`, `gridrow.lua`, and `gridsquare.lua` extend  `InputContainer`, which is a KOReader class that can be used for registering input events (`InputContainer` actually extends `WidgetContainer`). `puzzle.lua` and `solve.lua` dont extend any class... they are essentially models, containing data on the active crossword puzzle. 

## main.lua ##
The most top-level file, so to speak. This file includes logic for starting a new crossword and selecting the folder to use for game data. There are number of methods required by KOReader to propagate the plugin in the menu, and to create the menu entries for the plugin. There are also two key methods used for starting a new crossword. The methods are: 

- `Crossword:loadPuzzle()`
- `Crossword:loadGameView()`

`loadPuzzle()` is the more interesting of these two methods. It includes a bit of I/O to open a JSON file containing all of the information required to build a puzzle. I didn't create these JSON files myself. They are from a project by `doshea` on GitHub, called [nyt_crosswords](https://github.com/doshea/nyt_crosswords/). Anyways, once opened, the file contents are read into a variable and passed on to a new instantiation of the `Puzzle` object, whose main purpose is to cerate and maintain a single list corresponding to the current state of the puzzle. 

```lua
function Crossword:loadPuzzle()
    local file_path = ("%s/%s"):format(self.puzzle_dir, "/1990/01/01.json")
    local file, err = io.open(file_path, "rb")

    if not file then
        return _("Could not load crossword")
    end

    local file_content = file:read("*all")
    file:close()

    local Puzzle = require("puzzle")
    local puzzle = Puzzle:new{}
    puzzle:init(json.decode(file_content))
    return puzzle
end
```

## gameview.lua ##
This file contains the `InputContainer` for managing the player's game. `GameView` is responsible for receiving user input (such as gestures and keyboard input) and deciding which actions to invoke. It is also responsible for rendering the view widgets.

- `GameView:render()`
- `GameView:refreshGameView()`
- `GameView:advancePointer()`
- `GameView:leftChar()`
- `GameView:toggleDirection()`

## puzzle.lua ##
This file contains the class which holds onto all of the information about the current crossword. `Puzzle` doesn't directly interact with the view (`GameView`). Instead, it has a number of methods that return information like the color of the squares, the letters inside the squares, and the active square. Here are some examples of what methods this file includes:

- `Puzzle:getGrid()`
- `Puzzle:setActiveSquare()`
- `Puzzle:getSquareAtPos(row, col)`
- `Puzzle:getNextCluePos(row, col, direction)`
- `Puzzle:createSolves(clues, answers, direction, grid_nums)`

And here are some excerpts from the more interesting methods:

```lua
function Puzzle:getSquareAtPos(row, col)
    local index = ((row - 1) * self.size.rows) + col
    return self.solves[index]
end
```

```lua
function Puzzle:getNextIndexForDirection(index, direction)
    index = index + 1
    local solve = self.solves[index]
    -- If solve is nil, we're probably at the end of the list,
    -- so call this method again from start of list (but we call the
    -- method with '0' because the function will advance the index to 1.
    if solve == nil then
        return self:getNextIndexForDirection(0, direction)
    end
    -- If solve direction does not match desired direction, advance
    -- the index and call this method again.
    if solve.direction ~= direction then
        return self:getNextIndexForDirection(index, direction)
    end
    -- If we made it this far then we have the next solve.
    return solve
end
```

<a name="c-3-3"></a>
> **Competency 3**
> Demonstrate how to use functions by creating your own methods to send and return data

`Puzzle:getNextIndexForDirection(index, direction)` is kinda interesting, since it includes a bet of recursion to deal with edge cases where the index given to this method exceeds the list of solves ("solves" are what I call the clue/grid number/answer combos, e.g.: 2. (Down) Beta predecessor). I should probably rename the method, though, as I see now that it returns the next solve and not the next index, as the method signature deceptively indicates. 

## solve.lua ##
This file is used to model each of the clue/grid/answer combos that make up any given crossword. One of the early challenges of this project was figuring out how to eliminate redundancy in the data models. It would get pretty confusing, I thought, if I had separate models for the grid and for the solves. So I decided to avoid having a grid model. Instead, the grid is generated by `Puzzle:getGrid`, which basically iterates through the list of solves and builds the grid by looking at field called `grid_indices` and another called `grid_num`. `grid_indices` contains an integer representing each index on the grid that is occupied by one of a solve's letters. For example: if the word "ALPHA" begins at the first square on the puzzle and runs across, `grid_indices` would contain: `[1,2,3,4,5]`. Likewise, if the word "APPLE" began at the first square on a puzzle and runs down (on a 10 by 10 puzzle), `grid_indices` would contain `[1, 11, 21, 31, 41]`. `grid_num` contains the first value of this list (so `1`). Probably, that's redundant and could be removed. However: this value is supplied by the data that generates the crossword puzzle. It's a very important tidbit so I guess I'd like to keep it around.

So, how is `grid_indices` constructed for each solve? Great question! I found this problem a bit of a brain jogger and when I finally nailed it, I felt like superwoman. The only values that need to be known are:

- The size of the crossword (rows and columns)
- The index corresponding to the solve's first letter
- The length of the solve

For solves going down, the solution can be expressed declaratively as what follews: the grid indices occupied by the word are those indices whose value is the same as the value found by adding the the grid index corresponding to the first letter of the word to the width of the crossword grid multiplied by the given index of the word's character. (Thinking declaratively is new for me, so in exchange for your patience I give you my apologies!) Here is part of the method that populates `grid_indices` for a given solve:

```lua
function Solve:init(puzzle_size, puzzle_grid_nums)
    ...
	
    -- Compute the grid indices. I.e.: each index on the grid that
    -- is occupied by one of this solve's letters.
    self.grid_indices = {}
    local word_length = string.len(self.word)
    local width = puzzle_size.cols
    local height = puzzle_size.rows

    if self.direction == Solve.DOWN then
        local index = 0
        for char in string.gmatch(self.word, "[A-Z]") do
            local grid_index = self.grid_num + (index * width)
            table.insert(self.grid_indices, grid_index)
            index = index + 1
        end
    elseif self.direction == Solve.ACROSS then
        local index = 0
        for char in string.gmatch(self.word, "[A-Z]") do
            local grid_index = self.grid_num + index
            table.insert(self.grid_indices, grid_index)
            index = index + 1
        end
    end
end
```
<a name="c-3-1"></a>
<a name="c-3-2"></a>
> **Competency 3**
> Demonstrate use of more than one selection structure.  
> Demonstrate use of more than one loop.

Expressed imperatively, the answer can be found by the following steps:

1. Get the length of the solve's answer (e.g.: `5` for "APPLE").
2. Iterate over each character in the word (e.g.: "A", "P", "P", ...).
3. Find the grid index by adding the location of the first letter to the width of the puzzle muliplied by the current character index.

Tada! And that's basically all this file is for.

# gridview.lua, gridrow.lua, gridsquare.lua #

The job of building the puzzle view, with its grid of white and black squares containing little numbers and letters, is distributed among three classes: `GridView`, `GridRow`, and `GridSquare`. The names of these classes are directly indicative of the part of the puzzle for which they are responsible. `GridSquare` is the smallest component of the puzzle, representing a single square that can be empty (black) or filled with a letter. `GridRow` is the next smallest component of the puzzle, representing a single row containing a number of `GridSquares`. `GridView` is the largest component of the puzzle, representing the entirety of the grid containing a number of `GridRows`.

Coming up with this structure was easy, but finding the right measurements for each UI component was challenging. I decided to put the code for most important measurement, the square height and width, inside `GridView` because the measurement would be the same for each `GridSquare`, and I didn't see why I should duplicate that code inside a number of objects. It also makes the measurement more easily available for `GridRow`, which needs to know the square height as well.

```lua
function GridView:init()
    self.dimen = Geom:new{
        w = self.width or Screen:getWidth(),
        h = self.height or Screen:getHeight(),
    }
    self.outer_padding = Size.padding.large
    self.inner_padding = Size.padding.small
    self.inner_dimen = Geom:new{
        w = self.dimen.w - 2 * self.outer_padding,
        h = self.dimen.h - self.outer_padding, -- no bottom padding
    }
    ...
    self.square_width = math.floor(
        (self.dimen.w - (2 * self.outer_padding) - (2 * self.inner_padding))
        / self.size.cols)
    self.square_height = self.square_width
	...
end
```

The other interesting bit of thinking is found in the registration of the touch zones for each grid square. These values aren't actually used anymore (and are commented out in the code), because I found an alternate way to register touch events. It's simple math, but satisfying, I find, nonetheless!

```lua
...
   screen_zone = {
      ratio_x = (self.square_width * (col_num)) / self.dimen.w,
      ratio_y = (self.square_height * (row_num)) / self.dimen.h,
      ratio_w = ((self.square_width * (col_num)) + self.square_width ) / self.dimen.w,
      ratio_h = ((self.square_height * (row_num)) + self.square_height) / self.dimen.h,
   }
...
```

# Using Emacs #

Emacs has been my editor of choice since Spring 2021. Previously, I used Visual Code Studio. I forget exactly why I made the switch. Knowing myself, I would suspect it was out of stubborness and curiosity. Stubborn because I can be stubbornly contrarian and critical of the "normal ways of doing things". This same stubborness incited me to switch my keyboard layout to Dvorak in grade 12. And it striked again when I felt similarily incited to switch to Emacs, an editor with a near vertical learning curve. Nonetheless, I have been able to find footing in the editor and use it with delightful proficiency and efficiency. 

An initial challenge I faced while moving to Emacs was the apparent absence of a menu (some variations of the editor do have a menu, I believe. I run emacs from a virtual terminal, so there is no menu for me). Instead of using the mouse to select a menu item and then choosing an option (like "compile" or "save"), key bindings are mapped to functions. The most important key bindings of all is `C-h m`. This will open a buffer containing all key bindings that are valid for the current mode. For those new to Emacs, let me explain some jargon in those last two sentences. 

First: `C-h m` translates to "Hold down control and press the 'h' key. Then release those keys and press the 'm' key." All key bindings in Emacs are represented in this format. It's how folks talk about things on message boards, in tutorials, and when they write documentation for their Emacs extensions. `C` is called a "modifier key". The other  modifier key that is used in Emacs is the meta (or alt) key. For key bindings, it is represented as `M`, as in `M-x desktop-change-dir`, which translates to "hold down the meta key and press x; then type 'desktop-change-dir' into the mini buffer. Sometimes, the letters used for key bindings are indicative of the command they run. The `h` in `C-h m` means "help". The `x` in `M-x desktop-change-dir` means "execute," which is exactly what it does: it executes the given command, which in this case is a command to change the active directory used by the Desktop extension. 

Second: "buffers" are ["an interface between Emacs and a file or process."](https://smythp.com/emacs_buffers/) They hold the text of whatever it is you are looking at. If it's a Markdown file, like this document, you would see all of the special characters and formatting. If it's a PNG file, you would see all of the garbly-goop byte code (there are extensions you can install to see the visual representation of the PNG file, too). Buffers don't have to be visible to be active. Most often, I have dozens of open buffers representing all of the files in use by a project. To display a buffer in the window, I call it up using `C-x b <filename>`, where "filename" is the name of the file I want to see (and also the unique ID of the buffer [all buffers have a unique ID]). A neat thing about buffers is that a single buffer can be shown multiple times in an Emacs frame. For instance, I could split the Emacs frame into two side-by-side windows by using `C-x 3`. Then, I could open this file in the first buffer by invoking `C-x b README.md`, switch to the second buffer with `C-x o`, and open the buffer in the second window by invoking `C-x b README.md` again. Then I can have the first window viewing the top of the file (perhaps looking at the instructions I've set out for myself), and have the second window to me the area of the file I'm actively writting. 

<a name="c-1-use-of-menu"></a>
<a name="c-1-use-of-shortcut-icons"></a>
>**Competency 1**
>Demonstrate how to create programs using the integrated development environment and tools, including:
>
> - Use of menu
> - Use of Shortcut icons.

Compared to some folks, I run Emacs pretty lean. I don't take full advantage of Emacs extensions which would be analogues of Visual Studio's "Toolbox" or "Solution Explorer." To be fair, I'm just getting started in my Emacs journey, so what those analogues would be, I'm not entirely sure. Way back when I used Intellij IDEA (before I had even heard of Visual Studio Code), there was a method explorer I used when developing Android apps. I recall finding this handy, sure. Given the size of Android apps (with their complex Views, Contexts, and whatever), it was a welcomed feature. However, not having this feature for PHP web development and Lua development doesn't seem to bother me at the moment. Because of Emacs' ability to easily open multiple views into a single file, I am able to maintain a workflow where I have my reference to the code I'm working on displayed on one side of the frame, and the work area on the other side.

<a name="c-1-properties"></a>
<a name="c-1-toolbox"></a>
<a name="c-1-solution-explorer"></a>
>**Competency 1**
>Demonstrate how to create programs using the integrated development environment and tools, including:
>
> ~~- Properties~~
> ~~- Toolbox~~
> ~~- Solution Explorer~~

The one Emacs extension that I could not live without, however, is [Magit](https://magit.vc/). Magit is "a complete text-based user interface to Git." Using Magit feel superheroic. It eliminates hundreds of keystrokes in my day and has turned me into a moderately confident Git user. Like everything in Emacs, it is accessed via a key binding. `C-x g` reveals a buffer containing information on the git repository found in the active directory (where "active directory" is the same directory as active buffer's open file). For this repository, the buffer currently displays:

```
Head:     master Add initial documentation of crossword.koplugin
Push:     origin/master Add initial documentation of crossword.koplugin

Untracked files (4)
.emacs.desktop
MULT1083/Comp_1/
PROG1017G/Comp_4/
PROG1017G/Comp_5/

Unstaged changes (2)
modified   PROG1017G/Comp_2/README.md
modified   README.md

Recent commits
99f7285 origin/master Add initial documentation of crossword.koplugin
9a3c79f Add initial folder structure.
```

Missing from the above are the color-codings of key words like "master", and "Untracked file". But hopefully you get the idea! From this buffer, git commands can be invoked by pressing keys like "b c", which would checkout a new branch. It is also possible to reveal a help menu by pressing "h". It's a bit difficult to convey the true elegance of Magit here, using simple text. So check out the [visual walk-through](https://emacsair.me/2017/09/01/magit-walk-through/) to get a better grasp of the tool.

# Lua workflow #

So, how does one put together everything I've been writing about up until now? How does one develop a KOReader plugin written in Lua using Emacs? I'll cover the direct workflow of writing and testing code below. For resources on installing Lua, enabling Lua-mode in Emacs, and installing KOReader, please see these links:

- [Installing Lua](https://innovativeinnovation.github.io/ubuntu-setup/lua/) (Also see "LuaRocks and install that, too)
- [Installing Luacheck](https://github.com/lunarmodules/luacheck)
- [KOReader Development Guide](https://github.com/koreader/koreader/blob/master/doc/Building.md)
- [Installing Lua-mode for Emacs](lua-mode.luaforge.net) (Note: At the time of writting I was unable to open this page.)

## Setting up project files ##

If you've followed the KOReader Development Guide and installed the build enviroment, then somewher in your file system you have a folder called `koreader`. For me, this folder is located at `/home/scarlett/Development/_KOReader/koreader`. As per the guide, you should be able to launch the emulator by calling `./kodev run` inside of the `koreader` folder. A window will pop up and you will see the delightful little application that is KOReader. Neat!

If you want, you can absolutely start hacking away at the plugins by opening any of the files found in `../koreader/plugins`, like `../koreader/plugins/newsdownloader.koplugin/main.lua`. You could also start yourself a new plugin by copying the "hello, world!" plugin, `hello.koplugin`, to a new folder by running `cp -r hello.koplugin mynewplugin.koplugin`. 

My preferred process is to create my plugin folder in a directory outside of the emulator and use a soft-link to link the two together. My directory looks like the following (note: I've truncated the full directory by omitting my home folder).

```
/_KOReader
	/koreader
		/plugins
			crossword.koplugin -> [...]/_KOReader/koreader/crossword.koplugin
	/crossword.koplugin
		main.lua
		gameview.lua
		...
```

To create the soft-link, navigate to the plugin folder (`_KOReader/koreader/plugins`). From this directory, execute the following command (note: change the directories to match your filesystem):

```sh
# You must supply an absolute reference to the project folder where your plugin files are contained. Otherwise, the emulator trips up and breaks.
ln -s /home/scarlett/Development/_KOReader/crossword.koplugin crossword.koplugin
```

Run the emulator to make sure everything is hunky-dory. 

```sh
# Run from ../koreader
./kodev run
```

## Development workflow ##

For my actual development (i.e.: coding, linting, debugging, testing) process, it's pretty straight forward. First, I outline my work in pseudo-code. Then I slowly turn that over into code-code. Before running the emulator and testing the code, I will check the files over for correct syntax using `luacheck`. I run that program from the command line inside of my plugin directory. I pass `*` as an argument to the command so it checks the every file found in the directory. (As I'm writing this, I see that Luacheck can be integrated directly into Emacs through the [FlyCheck](https://www.flycheck.org/en/latest/) extension. I am absolutely going to be installing this extension next time I work on my plugin!) After linting/syntax analysing, I run the emulator with `./kodev run` and proceed to test the plugin by hand. Then it's a simple matter of rinsing and repeating those steps over and over again.

When it comes time to testing the plugin on a e-ink device, such as my Kobo Libra H2O, I use `scp` to transfer the files. First, I enable the wifi connection on KOReader. Then I launch KOReader's SSH daemon (Settings > Network > SSH server), set the port to 22 and "login without password". Then I run `scp` with the following arguments (note: you'll have to change the local address to whatever your device indicates):

```sh
scp -r --exclude='.*' crossword.koplugin/ root@192.168.2.16:/mnt/onboard/.adds/koreader/plugins/crossword.koplugin/
```

<a name="c-1-breakpoints"></a>
<a name="c-1-debugging-tools"></a>
>**Competency 1**
> Demonstrate how to create programs using the integrated development environment and tools, including:
>
> ~~- Breakpoints~~
> - Debugging tools
