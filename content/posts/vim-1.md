---
author: Taylor Deckard
title: Vim Intro
date: 2022-02-08
description: How to get started with Vim
tags: ["programming"]
---

![Vim](/blog/images/vim-1/vim-logo.png)

I've been using Vim as my primary editor for ~7 years. When first starting to use it, I compared it to learning a musical instrument. When learning chord progressions I'd start by playing slowly at first, thinking for a bit about which fingers go on which positions. Similarly, I had to think for a few seconds about which keys to press for each edit I was about to make in Vim. As with an instrument, the number of times you repeat the sequences, the more fluid the motions become. After several weeks of coding in vim, I no longer had to pause.

The more I learned about things you can do with Vim, the more I wanted to use it... and there is a lot you can do. For this post, I'll go over how to get started using Vim from scratch.

I spend a majority of my day in the macOS Terminal, which is one of the reasons I like using Vim so much. In order to start from a completely clean slate though, I'll install vim fresh in a docker container running ubuntu linux. To do that in Terminal the command is:
```sh
docker run -ti ubuntu
```

In the ubuntu shell, I'll install vim with:
```sh
apt-get update && apt-get install -y vim
```

Then I'll run Vim with `vi`. It looks like this:
![Vim Splash](/blog/images/vim-1/vi-splash.png)

Now what? I'll pretend for a moment that I have zero experience with Vim. The splash screen shown above gives a little context. Use `:q<Enter>`. Use `:help<Enter>` for on-line help. The Vim documentation is very good. You can also search for topics by using `:help word<Enter>` (where "word" is the topic you want to search for.)

### Executing Commands
First, note that typing `:` in normal mode (I'll talk more about modes in a bit) will display the `:` at the bottom of the window. Any characters you type next will also be displayed here. This is the command-line input. Notice `:help` and `:q` are commands that do different things. There are many different commands. You can even run shell commands from Vim.

For now, just remember that `:q!<Enter>` quits Vim without saving. Notice I've added a `!`. This forces Vim to quit even if there are unsaved edits. Alternatively, `:w<Enter>` saves, and `:wq!<Enter>` quits and saves.

### Moving Around (part 1)

Learning how to move the cursor around efficiently is probably one of the biggest challenges when starting out. The arrow keys can be used, but it is recommended to instead use `h`, `j`, `k`, and `l`. See the below mapping.
```
Up: k
Down: j
Left: h
Right: l
```
Typing each of these keys individually will move one character for `h` or `l` and one line for `j` or `k`. If you want to move down 20 lines, you don't have to press `j` 20 times. Instead, you can type `20j`. 

With that said, knowing which line to move down to can be difficult if you can't see the line numbers. You can show them with `:set number<Enter>`. To make jumping around even easier, you can `:set relativenumber<Enter>` which will still display the line number of the current line, however all other lines display the number of lines away from the current line. See the example below.
{{< video-player src="/blog/images/vim-1/vim-line-nos.mp4" >}}

I like to have the **relativenumber** setting active every time I open a file. Luckily, there is an easy to make Vim remember all of your settings.

### Configuring ~/.vimrc
To store Vim configurations, I'll create a .vimrc file.
```sh
vi ~/.vimrc
```

Now I want to add the setting to show relative line numbers to the .vimrc file.

```
set relativenumber
```

There is a problem though. I haven't mentioned how to insert text yet. To demonstrate, I'll enter this key sequence: `iset relativenumber<Esc>:wq!<Enter>`. Now, to explain...

### Modes
Vim has a few different editor modes that you can switch between:
- **Normal**: Cursor navigation, entering other modes.
- **Command**: Command execution mode. (Initiated with the `:` key from Normal mode)
- **Insert**: Text editing mode.
- **Replace**: Anything you type will replace the underlying text.
- **Visual**: Used to select text. There are also Visual Line and Visual Block modes which allow text selection in different ways.

Consider the key sequence `iset relativenumber<Esc>:wq!<Enter>`. Vim starts in **Normal** mode. The `i` key initiates **Insert** mode. At this point, `set relativenumber` is recorded as it is typed. Then, `<Esc>` exits **Insert** mode and Vim is back in **Normal** mode. Typing `:` enters **Command** mode and `wq!<Enter>` triggers the command to save and quit. When actively using Vim, I am switching between modes almost constantly.

### Mapping Keys
An awesome feature of Vim is the ability to map a key or key sequence to an action. This allows you customize Vim so that common tasks are easier to execute.

For instance, a mapping that I use is:
```
inoremap jk <esc>
```
This makes it so typing `jk` (quickly) in **Insert** mode will trigger the Escape keypress to enter **Normal** mode. I find this is easier than reaching to the Escape key for such a commonly performed action. There are very few scenarios when I actually need to insert the letters `jk` and on those occasions I can type slow so that the mapping isn't triggered. You can add the above line to your `~/.vimrc` to try it out.

The mapping might seem confusing at first. The first term, `inoremap` is the type of mapping. We can break this down further into three parts:
1. `i` - denotes **Insert** mode
2. `nore` - not recursive (other mappings cannot trigger this mapping)
3. `map` - basic map command

The second term, `jk`, is the keypress sequence that triggers the mapping (input.) The third term, `<esc>`, is the key that the mapping executes (output.)

Another mapping that I use frequently is:
```
nnoremap <C-n> :Explore<cr>
```
Note the first term begins with a `n` which denotes the mapping will only trigger in **Normal** mode. The `<C-n>` represents a combined keypress of the `Ctrl` and `n` keys. The last term, `:Explore<cr>`, enters **Command** mode and executes **netrw**, the native Vim file explorer. (`<cr>` is "carriage return", also known as the `Enter` key.)

You also have the option to make use of a "leader" key. This is a key of your choosing that you can use in mappings with the syntax: `<leader>`. I set my leader ket to be `;` by adding the following line in my `~/.vimrc`:
```
let mapleader = ";"
```
Now, the leader can be used in mappings like this:
```
nnoremap <leader>w :w!<cr>
```
This mapping allows for quick saving with `;w` in **Normal** mode.

You can also disable default mappings. In the past, to train myself to stop using the arrow keys for navigation, I added the following to my `~/.vimrc`:
```
nnoremap <Up> <nop>
nnoremap <Down> <nop>
nnoremap <Left> <nop>
nnoremap <Right> <nop>
vnoremap <Up> <nop>
vnoremap <Down> <nop>
vnoremap <Left> <nop>
vnoremap <Right> <nop>
inoremap <Up> <nop>
inoremap <Down> <nop>
inoremap <Left> <nop>
inoremap <Right> <nop>
```
This overrides the arrow keys to be no-ops in **Normal**, **Visual**, and **Insert** modes. With these set, you _must_ use the `h`, `j`, `k`, and `l` keys instead. 

There are many alternative ways to move the cursor around in Vim though...

### Moving Around (part 2)
When reading a large file, moving the cursor line by line is not ideal. Instead of using `j`/`k` to move, you can move half the page at a time with `Ctrl-D`/`Ctrl-U`.To go to the last line in the file, simply press `G`. To go back to the first line, press `gg`.
{{< video-player src="/blog/images/vim-1/vim-movement.mp4" >}}

On a long line, you may want to move forward and backward by word instead of by character. For this, use `w` to move to the next word and `b` to move to the previous. Similarly to moving down 20 lines with `20j`, you can move forward 20 words with `20w`.

Entering `^` moves the cursor to the first non-whitespace character on a line and `$` moves it to the last character.

If there is a particular character you need to reach on a line, `f` followed by the character will move the cursor to the next occurence of the character on the line. Using `F` will move to the previous occurence.

You can also easily perform a search to move your cursor to a word anywhere in the file. Simply type `/` (to search forward, `?` to search backward) followed by the word you're searching for, then `Enter`. When searching, `n` moves to the next occurence of the searched term, while `N` moves to the previous occurrence. Add the following line to your `~/.vimrc` for Vim to highlight the search matches for you:
```
set hlsearch
```
Also, I like to use the following settings:
```
set ignorecase
set smartcase
```
This makes it so searching for a word with all lowercase characters ignores case. However, a search including uppercase letters will be case-sensitive.
{{< video-player src="/blog/images/vim-1/vim-search.mp4" >}}

When the cursor is placed on a character of a pair, like opening and closing brackets, `%` will move the cursor to the opposite character. For example, if you are editing a text file that looks like `{ "an": "object" }` and your cursor is on `{`, pressing `%` will move the cursor to `}`.

If at any time you make a bad move, you can go back to your previous position with `Ctrl-o`.

### Editing Text
I've mentioned that the `i` key from **Normal** mode switches to **Insert** mode in which text is recorded as you type. There are other ways to enter **Insert** mode as well. All of the keys below will enter **Insert** mode and also move the cursor to a new position:
- `a`: Moves the cursor one character to the right.
- `A`: Moves the cursor to the end of the line.
- `I`: Moves the cursor to the beginning of the line.
- `o`: Inserts a line below the current line.
- `O`: Inserts a line above the current line.

Some keys enter **Insert** mode and also delete characters:
- `c`: Used in combination with a motion will delete characters within the motion. For instance, `cw` will delete the word all text from the cursor to the next word.
- `cc`: Deletes the entire line.
- `C`: Deletes all text on the line after the cursor.
- `s`: Deletes a single character.

Alternatively, some keys delete characters without changing from **Normal** mode:
- `d`: Works similarly to `c` except does not enter **Insert** mode.
- `dd`: Deletes the entire line (same as `cc`.)
- `D`: Deletes all text on the line after the cursor (`C`.)
- `x`: Deletes a single character.

If at any point you make a mistake, you can undo the action with `u`. (To redo the undo, use `Ctrl-r`.)

### Selecting Text
Text selection is made possible in Vim with **Visual** mode. Ways to enter visual mode are:
- `v`: Selects a single character. 
- `V`: Selects the entire line.
- `Ctrl-v`: Selects a single character, however subsequent cursor movement selects blocks of text vertically.

Moving the cursor after entering **Visual** mode will add to selection until exiting **Visual** mode with `Escape` or `Ctrl-C`.

Once in visual mode, you can perform actions on the selected text such as:
- `d`/`x`: Delete the selected text.
- `c`/`s`: Delete the selected text and enter **Insert** mode.
- `gu`/`gU`/`~`: Change the case of all letters in the selection.
- `<`/`>`: Indent the selected text left or right.
- `:norm @<register>`: Execute a macro on all lines (read about these a bit further down.)
- `y`: Yank (copy) the selected text.

Note that the above may not apply to **Visual Block** mode. Here's an example of that:
{{< video-player src="/blog/images/vim-1/vim-v-block.mp4" >}}
The above video shows how to convert a ordered list to an unordered list using **Visual Block** mode. The key sequence is `<Ctrl-v>Gls-<Esc>` Starting with the cursor in the top-left position, `Ctrl-v` enters **Visual Block** mode. Then, `G` moves the cursor to the bottom of the file, creating a selection of the entire first column. `l` moves the cursor to the right one character, expanding the selection to the first two columns. `s` then deletes the selected text and enters
**Insert** mode. It is important to note that **Insert** mode from **Visual Block** affects all rows that were selected. This means that `-` inserts the dash and `Esc` exits **Insert** mode while inserting the "-" on all lines that were selected in the **Visual Block**.

### Copying and Pasting
After selecting some text, you can yank (copy) it with `y` and then paste it with :
- `p`: Paste text immediately after the cursor.
- `P`: Paste text immediately before the cursor.

When yanked, text is stored in a register - a space in memory. Specifically, when yanked, text is stored in the default register (`"`).

Using a delete key (`x`, `d`, etc.) is actually similar to a "cut" in that some text is deleted and also stored in a register so that it can later be pasted.

### Registers
You can view the contents of a register by pressing `Ctrl-r` in **Insert** mode followed by the name of the register. You can try this out by selecting some text and typing `yi<Ctrl-r>"`. This output is the same as paste. 

You can also yank and paste to a specified register. Select some text and type `"ay`. This yanks the selection to register `a`. Now, you can paste the contents of the `a` register with `"ap`. You could also use `<Ctrl-r>` to do the same, as described above. (In **Insert** mode, type `<Ctrl-r>a`.)

### Macros
Macros are a useful feature that also make use of registers. They allow you to record a sequence of key presses and repeat the sequence any number of times.

In the below example, I use a macro to create an ordered list:
{{< video-player src="/blog/images/vim-1/vim-macro.mp4" >}}
First, `i1. ` is entered on the first line. Then, `qa` begins recording the macro. Any key pressed at this point will be recorded until `q` is pressed in **Normal** mode. `^` moves the cursor to the begining of the line. `v` enters visual mode and `f ` moves the cursor to the first space character. Then `y` yanks "1. " and moves the cursor back to the beginning of the line. `j` moves the cursor down. `P` pastes the "1. " before the cursor. `^` moves to the beginning of the 2nd line and
`<Ctrl-a>` increments the number under the cursor by 1. Finally, `q` ends recording of the macro. The macro stored in the `a` register looks like this: `^vf yjP^<c-a>`. 

To use the macro, select the lines you want to execute the macro on and type `:norm @a<Enter>`. Another way to do this is to count the number of lines you need to apply the macro to and with the cursor on the first line in **Normal** mode type `7@a`. (The number of lines is `7` for the example above.)

If you make a mistake when recording a macro, you can paste the register, correct the text, and then yank the fixed text back to the register. Then play the macro as you normally would.

### Plugins
Check out [VimAwesome](https://vimawesome.com/) for plugins. I personally use:
- [ALE](https://vimawesome.com/plugin/ale) - Linting
- [YouCompleteMe](https://vimawesome.com/plugin/youcompleteme) - Autocompletion
- [incsearch.vim](https://vimawesome.com/plugin/incsearch-vim) - Highlight text as you search
- [vim-airline](https://vimawesome.com/plugin/vim-airline) - Enhanced Status Bar
- [vim-commentary](https://vimawesome.com/plugin/commentary-vim) - Mappings to add code comments
- [vim-fugitive](https://vimawesome.com/plugin/fugitive-vim) - Git Integration
- [vim-surround](https://vimawesome.com/plugin/surround-vim) - Mappings to easily delete, change and add parentheses, brackets, quotes, XML tags, and more surroundings in pairs.

For an example, I'll install [vim-airline](https://vimawesome.com/plugin/vim-airline) using Vim's native packages.

First, I'll need git:
```
apt install -y git
```

Now, I'll create a directory for the plugins:
```
mkdir -p ~/.vim/pack/plugins/start
```
Next, clone the [vim-airline](https://vimawesome.com/plugin/vim-airline) repository into the directory that was just created.
```
git clone https://github.com/vim-airline/vim-airline.git ~/.vim/pack/plugins/start/vim-airline
```
Finally, add the following line to the `~/.vimrc` to set the maximum number of colors that can be displayed by the host terminal:
```
set t_Co=256
```
To check that the plugin is installed and working, restart Vim. It should now look like this:
![Vim Plugin](/blog/images/vim-1/vim-plugin.png)
