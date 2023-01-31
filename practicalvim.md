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
When a Visual mode command is repeated, it
affects the same range of text 

todos:

- explore vim tutor and make notes
- write medium article about the power of (.) and (*) in vim, and tips & tricks of these symbols when in visual mode
- write medium article about the REPLACE mode

Pro tips
- pressing A to enter insert mode is better than pressing i

  ...We're waiting for copy before the site can go live, ,
  ...If you are copy with this, let's go ahead with it...
  ...We'll launch as soon as we have the copy..k

today
- dailyinterviewpro -> Analyse problem from alreadySolved folder
- finish and publish my article
- push part 1 of lightning web app

tomorrow
- learnbtcfrmcommandline


one thing i particularly find amusing about vim is its modes. Unlike other editor where Insert/Editing mode
is the default. Normal mode is Vim's default and most times you don't need to switch from Normal mode to get stuff done

going through the bitcoin code base is the ultimate 'Alice in Wonderland' experience. One particular area i found interesting to 
dig deep into is coin selection in the bitcoin wallet...

