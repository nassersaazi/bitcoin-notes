practical vim

Chapter 1

.(dot) command repeats the last change
u command undoes the last change
A command appends at end of current line
x command deletes the character under the cursor
s command deletes the character under the cursor and get into Insert mode
$ command moves you to end of line
dd command deletes current line
>G increases indentation from current line to end of file
f{char} command tells Vim to look ahead for the next occurrence of the specified character and then move the
cursor directly to it if a match is found
F{char} command tells Vim to look backwards for the previous occurrence of the specified character and then move the
cursor directly to it if a match is found
; command will repeat the last search that the f/F command performed
, command will repeat the last search that the f/F command performed in the reverse direction
* command executes a search for a word under a cursor
:vs command does a vertical split
Ctrl + W + (hjkl) -> changes to different window in any direction
Ctrl + W + Ctrl + W -> toggles btn open windows
vim path/to/folder -> takes you to folder path and prompts file selection
The Dot Formula: One keystroke to Move, One keystroke to Execute

Chapter 2

vim provides a modal user interface. this means that the result of pressing any key on the keyboard may differ depending on which mode is active at
the time. it’s vital to know which mode is active and how to switch between vim’s modes.

vap, then u -> converts paragraph to lowercase
vap, then U -> converts paragraph to uppercase

Chapter 3

J command joins the current and next lines together
Replace mode is identical to Insert mode, except that it overwrites existing text
in the document.

chapter 4(visual mode)
gv command reselects last visual selection
o command in visual mode  -> go to other end of highlighted text
r- command in replaces all selected text with -

When a Visual mode command is repeated, it affects the same range of text 

The i and a commands both switch from Normal to Insert mode,  positioning the cursor in front of or after the current
character, respectively. The I and A commands behave similarly, except that they position the cursor at the start or end of the current line.


Chapter 5(command line mode)

The :copy command (and its shorthand :t) lets us duplicate one or more lines from one part of the document to another, while the :move command lets us place them somewhere else in the document

:%s/Practical/Pragmatic/ -> substitute every occurence in the file of Practical with Pragmatic
:4t. -> copy contents of line 4 to current(where cursor is)
:t$ -> copy current line to end of file
:m command is similar to :t command; where :t copies,:m cuts
:sp {file} command splits window horizontally, loading {file} into new window
:vs {file} command splits vertically, loading {file} into new window

:e# takes you to the previous file stored in the buffer
:!ls shows contents of current directory 

q: accesses the command-line window
q/ Open the command-line window with history of searches

recall commands from history with up and down arrows in command mode or search mode

when a visual selection is made(eg Vjj) norm . in command mode executes the last normal mode command on the selected lines

When duplicating a distant line, the :t command is usually more efficient yyp command

On Vim’s command line, the % symbol is shorthand for the current file name

Ctrl-z suspends vim and gets to the bash terminal .fg command brings back a recently suspended vim session

Chapter 6 (Manage multiple files)

Files are stored on the disk, whereas buffers exist in memory. When we open a file in Vim, its contents are read into a buffer, which takes the same name as the file. Initially, the contents of the buffer will be identical to those of the file, but the two will diverge as we make changes to the buffer. If we decide that we want to keep our changes, we can write the contents of the buffer back into the file. Most Vim commands operate on buffers, but a few operate on files, including the :write, :update, and :saveas commands

The :ls command gives us a listing of all the buffers that have been loaded into memory . We can switch to the next buffer in the list by running the :bnext command

We can traverse the buffer list using four commands—:bprev and :bnext to move backward and forward one at a time, and :bfirst and :blast to jump to the start or end of the list. 

The argument list (:args) represents the list of files that gets passed as an argument when we run the vim command.We can change the contents of the argument list at any time, which means that the :args listing doesn’t necessarily reflect the values that were passed to the vim command when we launched the editor

The argument list is simpler to manage than the buffer list, making it the ideal place to group our buffers into a collection. With the :args {arglist} command, we can clear the argument list and then repopulate it from scratch with a single command. We can traverse the files in the argument list using :next and :prev commands. Or we can use :argdo to execute the same command on each buffer in the set

Command           | Effect

<c-w>w            | Cycle between open windows
<c-w>h            | Focus the window to the left
<c-w>j            | Focus the window below
<c-w>h            | Focus the window above
<c-w>l            | Focus the window to the right
<c-w>c            | Close the active window
<c-w>o            | Keep only active window ,closing all others
:tabe {filename}  | Open {filename} in a new tab
:tabc             | Close current tab page and all of its windows
:tabo             | Keep the active tab page, closing all others

todos:

- dailyinterviewpro -> analyse problem from alreadysolved folder
- explore vim tutor and make notes
- write medium article about the power of (.) and (*) in vim, and tips & tricks of these symbols when in visual mode
- write medium article about the replace mode
- write medium article about the story of the etymology of vim -> refer to chpt 5 of practical vim
-
pro tips

- pressing a to enter insert mode is better than pressing i


today
- dailyinterviewpro -> analyse problem from alreadysolved folder
- finish and publish my article
- push part 1 of lightning web app

tomorrow
- learnbtcfrmcommandline


one thing i particularly find amusing about vim is its modes. Unlike other editor where Insert/Editing mode
is the default. Normal mode is Vim's default and most times you don't need to switch from Normal mode to get stuff done

going through the bitcoin code base is the ultimate 'Alice in Wonderland' experience. One particular area i found interesting to 
dig deep into is coin selection in the bitcoin wallet...

  ...We're waiting for copy before the site can go live, ,;
  ...If you are copy with this, let's go ahead with it...;
  ...We'll launch as soon as we have the copy..k;
