# Cart Life source code: a linear walkthrough

*2026-10-05T21:34:53Z by Showboat 0.6.1*
<!-- showboat-id: 39fdf37b-44f1-4178-9e71-89f6f7caf4df -->

This document walks through the source of **Cart Life** (Richard Hofmeier, 2011) in the order the game itself runs: project manifest, engine entry points, boot, title screens, the simulation clock, the city, the vending loop, conversation, end of day, and endings.

Every code block below is a real shell command run from the repository root, followed by its captured output, so each claim can be checked against the lines it quotes. Snippets are trimmed with `sed -n`, `grep` and `cut` because many source lines are several hundred characters wide; `tr -d '\r'` strips the Windows line endings.

A note on tone before starting. The author says this in the first lines of the global script, and it is the right frame for everything that follows: the code is enormous, repetitive and written by someone learning as they went. The aim here is to explain how it works, not to grade it.

## 1. What is in the repository

Cart Life is an [Adventure Game Studio](https://www.adventuregamestudio.co.uk/) (AGS) project. AGS games are made of one big XML project file, a set of script modules written in AGS Script (a C-like language), one script plus one binary per room, and binary asset bundles. Counting tracked files by extension shows that shape.

```bash
git ls-files cartlife | sed 's/.*\.//' | sort | uniq -c | sort -rn | head -8

```

```output
     76 ogg
     62 wav
     47 asc
     40 crm
     10 ttf
      8 ash
      7 ico
      3 rar
```

- `.asc` / `.ash` are script bodies and script headers. This is the source code proper.
- `.crm` are compiled rooms (background art, walkable areas, regions, object placement). Each `roomN.crm` has a matching `roomN.asc`.
- `.ogg` / `.wav` under `AudioCache/` are music and sound effects.
- The sprite bundle (`acpsprset`) is shipped as a two-part `.rar` that has to be unpacked before the AGS editor can build the game.

The script files, sorted by size, show where the weight is.

```bash
cd cartlife && wc -l GlobalScript.asc DialogRequesting.asc CustomFunctions.asc AskOnly.asc customertalk+listen.asc FadingThingsNonBlocking.asc Parallax_ASH.asc KeyboardMovement_102.asc
echo "--- all 40 room scripts together:"
cat room*.asc | wc -l

```

```output
  14606 GlobalScript.asc
  12204 DialogRequesting.asc
  10129 CustomFunctions.asc
   3759 AskOnly.asc
    614 customertalk+listen.asc
    411 FadingThingsNonBlocking.asc
    294 Parallax_ASH.asc
    216 KeyboardMovement_102.asc
  42233 total
--- all 40 room scripts together:
9334
```

About 41,000 lines live in five hand-written modules, and roughly 9,300 more are spread across the room scripts. The three small files at the bottom (`FadingThingsNonBlocking`, `Parallax_ASH`, `KeyboardMovement_102`) are community modules the author pulled in.

Each big module opens with a comment that states its job. Those comments are the best table of contents the code has. (The files are Windows-1252 encoded, hence the `iconv`.)

```bash
cd cartlife
sed -n '3,11p' GlobalScript.asc | iconv -f cp1252 -t utf-8 | tr -d '\r'
echo ...
sed -n '1,4p' CustomFunctions.asc | tr -d '\r'
echo ...
sed -n '1,4p' DialogRequesting.asc | tr -d '\r'
echo ...
sed -n '1,6p' AskOnly.asc | tr -d '\r'

```

```output
//==// WELCOME TO CART LIFE'S OPEN SORES. //==//
//   This is the global script, where most of the nonsense happens.
//   For cart life, the GlobalScript is mainly used for the following general processes:
//   • Setting up the game with the "game_start" function.
//   • Responding to mouseclicks.
//      - Things like menu buttons, inventory items, the cash register, all Vinny's cooking stuff, etc.
//   • Time Math (time of day, day of week, speed adjustments, etc).
//   • Debug functions!
//   • Lots of other junk.
...
//==// CUSTOMFUNCTIONS //==//
// Stuff like handling the saved games, animating the faces while talking/listening
// and each character's buying / selling dialogs (including Thales and Johns issuing fines, etc).
// Each menu item's sale functions begin here, too.
...
//==// DIALOG REQUESTING //==//
// All of the dialog options in cart life are numbered with an "xvalue" integer,
// and the responses are listed here, from lowest xvalue to the highest.
// Using AGS's built-in dialog tools, each dialog response points here, as in speakmind(xvalue);
...
//ASKONLY//
//  This script provides the dialog during most conversations.
//  The "disavow" function is called when a character doesn't know how to respond to the player's question
//with distinct responses for either a person, place, or thing.
//  The "qverbatim" function has the player character articulate the question.
//  Mainly, though, "askabout" has all of the ASK X ABOUT Y dialog.
```

## 2. `Game.agf`: the project manifest

`cartlife/Game.agf` is a 10 MB XML file written by the AGS editor. It is not code, but the code cannot be read without it, because it declares every name the scripts use without defining: characters, GUIs and their buttons, inventory items, dialogs, views (animations), audio clips and several hundred global variables.

The header settings show the engine contract. The game runs at 640x400 but scripts address the screen in low-resolution coordinates (320x200), which is why every coordinate in the scripts is small. Scripts are compiled against the AGS 3.4 API with 3.2.1 compatibility.

```bash
grep -E -m8 '<(GameName|ColorDepth|CustomResolution|UseLowResCoordinatesInScript|ScriptAPIVersion|ScriptCompatLevel|DialogOptionsGUI|SpeechStyle)>' cartlife/Game.agf | tr -d '\r' | sed 's/^ *//'

```

```output
<ColorDepth>TrueColor</ColorDepth>
<CustomResolution>640,400</CustomResolution>
<UseLowResCoordinatesInScript>True</UseLowResCoordinatesInScript>
<ScriptAPIVersion>v340</ScriptAPIVersion>
<ScriptCompatLevel>v321</ScriptCompatLevel>
<DialogOptionsGUI>36</DialogOptionsGUI>
<SpeechStyle>SierraWithBackground</SpeechStyle>
```

Counting a few element types gives a sense of how much of the game is data declared here.

```bash
for tag in Character GUIMain InventoryItem Dialog View GlobalVariable; do
  printf '%-15s %s\n' "$tag" "$(grep -c "<$tag>" cartlife/Game.agf)"
done

```

```output
Character       67
GUIMain         91
InventoryItem   227
Dialog          194
View            306
GlobalVariable  605
```

### Script module order

AGS compiles script modules top to bottom, and a module can only call functions from modules above it (via `import` lines in the `.ash` headers). The order recorded in the project file is therefore the dependency order of the whole game.

```bash
grep -n '<FileName>.*\.asc</FileName>' cartlife/Game.agf | tr -d '\r' | sed 's/<[^>]*>//g; s/  */ /g'

```

```output
211035: FadingThingsNonBlocking.asc
211066: KeyboardMovement_102.asc
211093: Parallax_ASH.asc
211117: customertalk+listen.asc
211141: CustomFunctions.asc
211165: AskOnly.asc
211189: DialogRequesting.asc
211213: GlobalScript.asc
```

Reading that list bottom-up gives the call hierarchy:

- `GlobalScript` (engine events, GUI click handlers, the clock) calls into everything.
- `DialogRequesting` holds `speakmind()`, the single giant dispatcher for scripted conversation outcomes.
- `AskOnly` holds the "ask X about Y" topic system.
- `CustomFunctions` holds the shared gameplay vocabulary: selling, queueing, pop-ups, endings.
- `customertalk+listen` animates talking heads.
- The three library modules sit at the top and depend on nothing.

The headers make the cross-module surface explicit. `GlobalScript.ash` is included into every script, rooms included, and it is tiny, because nearly everything shared is declared in `CustomFunctions.ash` instead.

```bash
cd cartlife
grep -v '^ *//' GlobalScript.ash | grep -v '^\s*$' | tr -d '\r'
echo "--- CustomFunctions.ash exports $(grep -c '^import' CustomFunctions.ash) names, e.g.:"
grep -n -E 'import function (BuyorTalk|Make|Botch|Permitcheck|places|closeupshop|curtaincall|sellRandom)\b|import int' CustomFunctions.ash | tr -d '\r'

```

```output
import function DoorHandle(this Character*);
import function DoorHandle2(this Character*);
import function gohome();
import function Overwriter();
import float tiprecord[]; 
--- CustomFunctions.ash exports 136 names, e.g.:
49:import function Make();
50:import function Botch();
57:import function Permitcheck();
60:import function places();
62:import function BuyorTalk();
101:import function sellRandom(this Character*);
142:import function curtaincall();
144:import function closeupshop();
156:import int hunger[];
157:import int que[10];
158:import int maxque;
159:import int queposition;
```

### Rooms

AGS organises a game into numbered rooms. The scripts refer to rooms by number only (`player.ChangeRoom(5, ...)`), so this table from the project file is the legend for the rest of the walkthrough.

```bash
sed -n '785,950p' cartlife/Game.agf | tr -d '\r' | grep -o '<Number>[0-9]*</Number>\|<Description>[^<]*</Description>' \
  | sed 's/<[^>]*>//g' | awk '/^[0-9]+$/{n=$0; next} {printf "%3s %s\n", n, $0}' | paste - - - | column -t -s$'\t'

```

```output
  1 Char_Select         2 Mel'sHouse       3 DreamSq
  4 Map(Stand)          5 Franklin         6 Vinny'sHouse
  7 Andrus'-Place       8 Logan's_house    9 Logan's_Bedroom
 10 Walk_Map           11 FullScreen      12 Machinery
 13 BrennanBridge      14 Downtown        15 Bigby_Groc
 16 O'Fagans           17 Downtown2       18 Courthouse
 19 Notgeld_Mall       20 Florin          21 Superstore
 22 13th               23 Main_Menu       24 Credits
 26 Collins            27 Roastierry      28 Breezys
 29 Dream1             30 Cass            31 Matty's
 32 Clamant_Building   33 Pawnshop        34 Park
 36 WalkThru           37 DemarcoVawl     38 School
 40 PROFILE                              
```

Three groups matter: menu and framing rooms (1 character select, 23 main menu, 24 credits, 40 loading/profile, 29 the dream you visit while asleep, 10 the travel map), the outdoor streets where a cart can be set up (5 Franklin, 13 Brennan Bridge, 14 and 17 Downtown, 20 Florin, 22 13th, 26 Collins, 34 the park), and interiors (homes, shops, the courthouse, the school).

### Global variables and custom properties

The scripts use bare names such as `Money`, `hour`, `rep` and `saleitem` that are never declared in any `.asc` file. They are AGS "global variables" declared in the project file and visible everywhere.

```bash
tr -d '\r' < cartlife/Game.agf | awk '
  /<GlobalVariable>/ {n=""; t=""; d="(none)"}
  /<Name>/ && n=="" {gsub(/<[^>]*>| /,""); n=$0}
  /<Type>/ {gsub(/<[^>]*>| /,""); t=$0}
  /<DefaultValue>/ {gsub(/<[^>]*>|^ */,""); d=$0}
  /<\/GlobalVariable>/ {print n, t, d}' \
 | grep -E '^(Money|hour|minute|ampm|dayofweek|dayspassed|rep|repmod|saleitem|salebuyer|saleprice|readytosell|SaleInProgress|clockspeed|melplot|superd|cheatbiking|Customer) ' | column -t

```

```output
minute          int         0
hour            int         1
ampm            int         0
dayofweek       int         1
dayspassed      int         0
rep             int         0
repmod          float       0.00
clockspeed      int         0
Money           float       (none)
saleprice       float       (none)
saleitem        String      (none)
salebuyer       String      (none)
Customer        Character*  (none)
SaleInProgress  int         0
cheatbiking     bool        false
melplot         int         (none)
readytosell     bool        false
superd          int         0
```

The project also defines *custom properties*: per-character numbers set in the editor and read with `GetProperty()`. Cart Life uses them as each customer's economic personality: the most they will pay for each product, in cents, plus taste preferences and whether they will wait in an over-long line. The values below are the schema defaults; individual characters override them in the editor.

```bash
grep -A3 '<CustomPropertySchemaItem>' cartlife/Game.agf | tr -d '\r' | grep -E '<(Name|DefaultValue)>' | sed 's/<[^>]*>//g; s/^ *//' | paste - - | column -t -s$'\t'

```

```output
PxPos                0
limitCoffee_S        1100
limitCoffee_a        400
limitCoffee_B        350
limitCoffee_C        300
limitCoffee_D        250
limitChai            350
limitCocoa           300
limitBagel_p         200
limitMilk            100
limitSoda            200
prefCoffee           4
prefBagel_p          4
limitHotdog          300
prefHotdog           3
willstopabovemaxcue  0
```

### Dialogs are thin shims

AGS has a built-in dialog-tree editor. Cart Life uses it only for presenting choices. Every option's script is a single `run-script N` line, which makes the engine call `dialog_request(N)` in the global script. This is the first dialog in the file.

```bash
sed -n '28165,28184p' cartlife/Game.agf | tr -d '\r'
echo "..."
echo "run-script lines in the whole project: $(grep -c 'run-script' cartlife/Game.agf)"

```

```output
          <Dialog>
            <ID>0</ID>
            <Name>dAlice_1v</Name>
            <ShowTextParser>False</ShowTextParser>
            <Script><![CDATA[// Dialog script file
@S  // Dialog startup entry point
return

@1
//Yes Script
run-script 1
stop
//return

@2
//NO script
run-script 2
stop
//return
]]></Script>
...
run-script lines in the whole project: 638
```

And this is the entire receiving end in `GlobalScript.asc`: it forwards the number to `speakmind()` in `DialogRequesting.asc`.

```bash
sed -n '11430,11433p' cartlife/GlobalScript.asc | tr -d '\r'

```

```output
function dialog_request (int xvalue) {
mouse.Visible=true;
speakmind(xvalue);
}//Enddialog
```

## 3. How the engine calls into the scripts

There is no `main()`. AGS owns the game loop (40 ticks per second by default) and calls well-known function names if a script defines them. Listing those names across the modules shows every engine entry point outside the rooms.

```bash
cd cartlife && grep -n -E '^function (game_start|repeatedly_execute|repeatedly_execute_always|on_key_press|on_mouse_click|on_event|dialog_request) *\(' \
  FadingThingsNonBlocking.asc KeyboardMovement_102.asc Parallax_ASH.asc CustomFunctions.asc GlobalScript.asc | tr -d '\r' | cut -c1-90

```

```output
FadingThingsNonBlocking.asc:161:function repeatedly_execute_always() {
FadingThingsNonBlocking.asc:356:function on_event (EventType event, int data) {
KeyboardMovement_102.asc:62:function repeatedly_execute() {
KeyboardMovement_102.asc:135:function on_key_press(int keycode) {
KeyboardMovement_102.asc:210:function on_event(EventType event, int data) {
Parallax_ASH.asc:190:function on_event (EventType event, int data){
Parallax_ASH.asc:229:function game_start(){
Parallax_ASH.asc:237:function repeatedly_execute_always() {
CustomFunctions.asc:91:function game_start() {
GlobalScript.asc:57:function game_start() {  //DAWN
GlobalScript.asc:2502:function repeatedly_execute_always() {
GlobalScript.asc:3246:function repeatedly_execute() {
GlobalScript.asc:6391:function on_key_press(int keycode){}
GlobalScript.asc:6393:function on_mouse_click(MouseButton button) // called when a mouse b
GlobalScript.asc:11430:function dialog_request (int xvalue) {
```

What each one means:

- `game_start()` runs once per module, in module order, before the first room loads.
- `repeatedly_execute()` runs every tick while the game is *not* blocked. A blocking call such as `Wait()`, a blocking animation or a running dialog suspends it.
- `repeatedly_execute_always()` runs every tick no matter what. Cart Life puts anything that must keep moving during dialogue here: mini-game input, GUI animation, the customer queue.
- `on_mouse_click` / `on_key_press` receive input events. `on_key_press` in the global script is empty: the game polls keys with `IsKeyPressed()` inside the tick functions instead.
- `dialog_request(n)` is called by `run-script n` in a dialog, as shown above.

Two functions in `GlobalScript.asc` are therefore the heart of the program, and they are very large.

```bash
cd cartlife && awk '
  /^function repeatedly_execute_always\(\)/ {a=NR}
  /^function RemoveBoxItem/ {print "repeatedly_execute_always: lines " a "-" NR-1 " (" NR-a " lines)"}
  /^function repeatedly_execute\(\)/ {b=NR}
  /END REPEX/ {print "repeatedly_execute:        lines " b "-" NR " (" NR-b+1 " lines)"}' GlobalScript.asc

```

```output
repeatedly_execute_always: lines 2502-3233 (732 lines)
repeatedly_execute:        lines 3246-6376 (3131 lines)
```

The other 11,000 lines of the global script are mostly GUI event handlers: functions named `<control>_OnClick` or `<control>_OnActivate` that the editor wires to buttons, sliders and text boxes.

```bash
cd cartlife
echo "GUI handlers in GlobalScript.asc: $(grep -c -E '^function [A-Za-z0-9_]+_(OnClick|OnActivate|OnSelectionChanged?|OnChange)\b' GlobalScript.asc)"
echo "inventory/hotspot handlers (_Look, _Talk, ...): $(grep -c -E '^function [A-Za-z0-9_]+_(Look|Talk|Interact|UseInv)\b' GlobalScript.asc)"

```

```output
GUI handlers in GlobalScript.asc: 398
inventory/hotspot handlers (_Look, _Talk, ...): 325
```

Rooms have their own event functions, wired per room in the editor: `on_event` with `eEventEnterRoomBeforeFadein` (set the scene), `room_AfterFadeIn`, `room_RepExec` (per-tick logic while in that room), `room_Leave`, and `regionN_Standing` / `regionN_WalksOnto` for floor triggers such as doors. Room 5 (Franklin) is a typical example.

```bash
grep -n -E '^function ' cartlife/room5.asc | tr -d '\r'

```

```output
2:function on_event(EventType event, int data) {
108:function Crowbustout(){
115:function CatIntro()
137:function room_AfterFadeIn(){
169:function CrowMannerism(){
179:function killtim(){
199:function region1_Standing()
234:function region2_WalksOnto(){
241:function region2_Standing(){
267:function room_RepExec(){
419:function room_LeaveLeft()
435:function room_LeaveRight()
451:function room_Leave(){places();}
453:function region5_Standing(){
```

## 4. The three borrowed modules

**FadingThingsNonBlocking** fades objects, characters and GUIs in the background. Callers register a fade in a table; the module's own `repeatedly_execute_always` steps every registered fade each tick. The game uses it for doors, title cards and the black overlay GUI `gFullblack`.

```bash
cd cartlife && sed -n '7,17p;36,39p' FadingThingsNonBlocking.asc | tr -d '\r'
echo "..."
echo "call sites outside the module: $(cat GlobalScript.asc CustomFunctions.asc DialogRequesting.asc AskOnly.asc room*.asc | grep -c -E 'Fade(Object|Character|Gui)(In|Out)_NoBlock')"

```

```output
struct FadeStuff {   // Setting up a struct.
  int Transparency;  //
  int Speed;         // Setting up variables
  int Limit;         // to use within struct.
  int Counter;       //
  int Timer;         //
  int Fadeout;       //
  int Fadetime;      //
  int Fadeamount;    //
  int Visible;       //
};
function FadeObjectOut_NoBlock (Object *Objectpoint, int Value, int Speed, int Timer) {
// if (Obj[Fading1].Fadeout) return 1; //aborts if fadeout is currently being executed.
 if (Obj[Fading1].Fadeout == 1) return;
  Fadeobj[Fading1] = Objectpoint; // Setting up Script O-name.
...
call sites outside the module: 191
```

**Parallax_ASH** ("Smooth Scrolling + Parallax") takes over camera scrolling and shifts room objects at different rates depending on a `PxPos` custom property (the first property in the schema listed earlier). Background skyline layers such as `mtns`, `bg_bldgs` and `fore_bldgs` in the street rooms are ordinary room objects moved by this module.

```bash
cd cartlife && sed -n '3,5p' Parallax_ASH.ash | tr -d '\r'; echo ...; sed -n '56,68p' Parallax_ASH.asc | tr -d '\r' | sed 's/\t/  /g' | grep -v '^[[:space:]]*$'

```

```output
// Requires a number property called 'PxPos' with a default value of 0.
// Positive values up to 3 will put an object in the foreground.
// Negative values down to -3 will put an object in the background.
...
  else if (pxObj[loop].GetProperty("PxPos")==-2) {
    pxObj[loop].X=FloatToInt(IntToFloat(pxObjOriginX[loop])+(screenCentreX/2.0), eRoundNearest);
    pxObj[loop].Y=FloatToInt(IntToFloat(pxObjOriginY[loop])+(screenCentreY/2.0), eRoundNearest);
    }
  else {
    pxObj[loop].X=FloatToInt(IntToFloat(pxObjOriginX[loop])+screenCentreX, eRoundNearest);  
    pxObj[loop].Y=FloatToInt(IntToFloat(pxObjOriginY[loop])+screenCentreY, eRoundNearest); 
```

**KeyboardMovement_102** is a stock arrow-key walking module. It is compiled in, but its mode defaults to "none" and nothing ever switches it on, so it does nothing at runtime.

```bash
cd cartlife && grep -n 'KeyboardMovement_Mode = ' KeyboardMovement_102.asc | tr -d '\r' | cut -c1-150
echo "SetMode calls anywhere else: $(cat GlobalScript.asc CustomFunctions.asc DialogRequesting.asc AskOnly.asc room*.asc | grep -c 'KeyboardMovement.SetMode')"

```

```output
36:KeyboardMovement_Modes KeyboardMovement_Mode = eKeyboardMovement_None; // stores current keyboard control mode (disabled by default)
47:	KeyboardMovement_Mode = mode;
SetMode calls anywhere else: 0
```

Instead, the global script moves the player itself with a small platformer routine inside `repeatedly_execute`: `xmove` is a horizontal velocity nudged by the arrow keys (or A/D) and bled off by friction, `gravity` pulls the character down one pixel at a time until a walkable area stops it, and `in_midair` records whether they have landed. AGS key codes 375 and 377 are the left and right arrows; 65 and 68 are A and D.

```bash
cd cartlife && sed -n '3999,4006p;4049,4050p' GlobalScript.asc | tr -d '\r' | cut -c1-175 | grep -v '^[[:space:]]*$'
echo ...
sed -n '3908,3921p' GlobalScript.asc | tr -d '\r' | cut -c1-150 | grep -v '^[[:space:]]*$'

```

```output
//managing movement values and gravity--
if (gravity < 6) {gravity=gravity+1;}
if (gMake.Visible==false){
if ((GetGlobalInt(1)!= 3)&&(player.Room!=36)&&(player.Room!=29)){
if (((IsKeyPressed(375)==1)||(IsKeyPressed(65)==1))&&(xmove>-2)) {xmove=xmove-1;} // left arrow
if (((IsKeyPressed(377)==1)||(IsKeyPressed(68)==1))&&(xmove<2))  {xmove=xmove+1;} // right arrow
...
//down----------------------------------
int a;
while ((gravity>0)&&(a < gravity)){
 player.y=player.y+1; //move one down
 if (GetWalkableAreaAt(player.x-GetViewportX(),player.y-GetViewportY())==0) {player.y=player.y+1;} //1 down if no walkable area found
 else
 {
     in_midair = false;
// blocked by a walkable area below, so player has just landed
 }
 player.y=player.y-1; //move one up
```

## 5. Boot: the two `game_start` functions

`game_start` runs in module order, so `CustomFunctions.asc` goes first. Before it, at file scope, sits the one piece of state that survives across playthroughs: a four-integer "score" array persisted to `data.dat`. It records, per playable character, how their story ended, and a fourth slot says whether the third character (Vinny) is unlocked.

```bash
sed -n '20,24p;29p;51,59p;64,73p;79,83p' cartlife/CustomFunctions.asc | tr -d '\r'

```

```output
  //playstate: 0-Unplayed 1-InProgress 2-LeftOnATrain 3-Lost 4-Won
  //playstate_andrus=score[0];
  //playstate_melanie=score[1];
  //playstate_vinny=score[2];
  //playstate_bonus=score[3];
int score[4];
function ReadScores () {
    int numbercutter[4];
    String namecutter[4];
    int i;
    File *f = File.Open ("data.dat",eFileRead);
    if (f) {
    while (i < 4) {
       numbercutter[i] = f.ReadInt();
       score[i] = numbercutter[i];
       i++;
    }
    f.Close ();
    }
    //playstate: 0-Unplayed 1-InProgress 2-LeftOnATrain 3-Lost 4-Won
    playstate_andrus = numbercutter[0];
    playstate_melanie = numbercutter[1];
    playstate_vinny = numbercutter[2];
    playstate_bonus = numbercutter[3];
}
function NewScores(int Slot, int Newvalue) {
    score[Slot]=Newvalue;
    WriteScores();
    ReadScores();
}
```

Its `game_start` then allocates the per-character arrays used by the customer system and zeroes a long list of counters. The four declarations just above it are the customer queue, which becomes important in section 11.

```bash
sed -n '85,91p;96,101p;117,121p' cartlife/CustomFunctions.asc | tr -d '\r'

```

```output
int hunger[];
int que[10]; //Ten is the absolute maximum.
int maxque; //"maxque" starts low and goes up with reputation.
int queposition;//This is the "end of the line", or entry place for new customers.
bool driptalked[];

function game_start() {
    topitm_charname.Visible = false;
    queposition=0;
    hunger = new int[Game.CharacterCount];//Array definition
    driptalked = new bool[Game.CharacterCount];//No coffee. How 'bout an Americano?

    rentpaid=0;//0: first week 1: second 2: third, etc.
    readspeed = 3; //1:Sssllloowww 2:Slow 3:Regular 4: Fast 5: Fst!
    pbspeed = 2; //Higher is slower, lower is faster. This is the Pace Bar speed (ie: dialog display speed).
    barspeed = 1; //Same as above - for the topbar gui

    clockspeed=27;//cyclecounter is at 27 by default - It's "2" when sleeping for a timer of 800.
```

`GlobalScript.asc`'s `game_start` runs last and is about 690 lines long. It does five things in sequence.

**(a) Engine settings and the unlock flag.** It loads the score file and then, in this "everything edition" build, unconditionally writes slot 3 so Vinny is unlocked. The commented alternative is the freeware build.

```bash
sed -n '57,58p;64,65p;72,76p;117p;123,130p' cartlife/GlobalScript.asc | tr -d '\r'

```

```output
function game_start() {  //DAWN

    readytosell = false;
    queposition = 0;//Starts at zero
    SetMusicMasterVolume(100);
    SetDigitalMasterVolume(100);
    game.skip_display=0;
    SetSkipSpeech(0);
    game.text_speed=50;
    ReadScores();//Need this
    /////////===//////// USE THE FOLLOWING FOR A TEST COMPILE FIRST, THEN REMOVE THE NEWSCORES(X,X); COMMANDS AND COMPILE AGAIN //===/////
    //==// MEGA-BONUS EVERYTHING EDITION //==//
    end_restart.NormalGraphic=8005;//Author's Site (instead of "Buy the Mega-Bonus Everything Edition)
    NewScores(3, 1); //Unlock Vinny!

    //==// MEGA-BORING FREEWARE EDITION //==//
    //end_restart.NormalGraphic=9372;//"Buy the Mega-Bonus Everything Edition" button
    //NewScores(3, 0); //Lock Vinny!
```

**(b) Hide the interface.** All 91 GUIs exist from the start; the game shows and hides them rather than creating them. Boot hides everything that should not be on the title screen.

```bash
sed -n '134,146p' cartlife/GlobalScript.asc | tr -d '\r'

```

```output
    //=====================   INTERFACE DISABLERY  ======================================================
    gui[0].Visible = false;
    gui[1].Visible = false;
    gui[2].Visible = false;
    gItemDesc.Visible = false;
    gCook2.Visible = false;
    gTopics2.Visible = false;
    gThirstex.Visible = false;
    gMake.Visible = false;
    gGui2.Visible = false;
    gDialog.Visible = false;
    mouse.Visible = false;
    nameplate.Visible = false;
```

**(c) Initialise the numbered global integers.** Alongside the named global variables, the game leans heavily on AGS's legacy `SetGlobalInt(index, value)` / `GetGlobalInt(index)` store. The comments in `game_start` are the only documentation of what each index means, which makes this block the key to reading every `GetGlobalInt` call elsewhere. The most important indices:

```bash
grep -n -E 'SetGlobalInt\((1|50|51|52|90|98|99|100|101|102|103|108|325|411),' cartlife/GlobalScript.asc | awk -F: '$1<340' | tr -d '\r' | sed 's/^\([0-9]*\): */\1: /'
echo ...
sed -n '237,247p' cartlife/GlobalScript.asc | tr -d '\r'

```

```output
229: SetGlobalInt(1,0); // Player Character Not Yet Chosen
230: SetGlobalInt(108,0); // Nutrition Tickdown
233: SetGlobalInt(90, 0); // 0:Matty's & Dompactor    1:Eddie's & Pawnshop
236: SetGlobalInt(411, 0);//Barneys Plot Meter
275: SetGlobalInt(325, 0);//0: Permit needs checking. 1:A-OK!
319: SetGlobalInt(90,0); // 0:Matty's / 1:Eddie's
323: SetGlobalInt(98,0); // TRAVEL Origin
324: SetGlobalInt(99,0); // TRAVEL Destination
327: SetGlobalInt(100,0); // Inventory Screen ON/OFF
328: SetGlobalInt(101,0); // Andrus Progress
329: SetGlobalInt(102,0); // Mel Progress
330: SetGlobalInt(103,0); // Vinny Progress
334: SetGlobalInt(50, 0); //Stand Location
335: SetGlobalInt(51, 0); //Stand Type
336: SetGlobalInt(52, 0); //Operating cart?
...
    //1: Plot given to the player
    //2: Eddie's mad.
    //3: Eddie's coming to talk it over.
    //4: Player declines Eddie's offer / turns him away [Eddie's always mad, from now on.]
    //5: Player's got the fakes
    //6: Player tells Barneys about fakes / betrays Eddie
    //7: Bookstore's busted - Barneys get the book
    //8: Roastierry is now addictive!
    //9: Fake delivered
    //10: Fake has taken effect - Roast coffee's terrible!
    //11: Roastierry closes, coffee C now available at Barney's.
```

So `GetGlobalInt(1)` is "which character am I playing" (1 Andrus, 2 Melanie, 3 Vinny, 4 the unfinished Logan), `101`-`103` are each protagonist's story progress, `52` is whether the cart is packed up (1) or open (0), and three-digit indices such as `411` are plot meters for side stories. A second convention appears in the same block: a low index `N` is "is this customer hungry" and `400+N` is "have I talked to them".

```bash
sed -n '259,269p' cartlife/GlobalScript.asc | tr -d '\r'
echo "..."
echo "GetGlobalInt(1) checks across all scripts: $(cat cartlife/*.asc | grep -o 'GetGlobalInt(1) *[=!]=' | wc -l)"

```

```output
    SetGlobalInt(14,0); // Toney Hungry
    SetGlobalInt(414, 0);//Toney Talked

    SetGlobalInt(15,0); // Jenny Hungry
    SetGlobalInt(415, 0);//Jenny Talked

    SetGlobalInt(16,0); // Richard Hungry
    SetGlobalInt(416, 0);//Richard Talked

    SetGlobalInt(17,0); // Troy Hungry
    SetGlobalInt(417, 0);//Troy Talked
...
GetGlobalInt(1) checks across all scripts: 2282
```

**(d) Stock the shops.** There is no shop data structure. Each store's shelf is the AGS inventory of an NPC, filled here with special "b"-prefixed buyable items. The comments say which character stands in for which shelf.

```bash
sed -n '393,397p;416,423p;466,467p' cartlife/GlobalScript.asc | tr -d '\r' | cut -c1-150

```

```output
    //=====================   STORE / MERCHANT INVENTORY ASSIGNMENTS   ================================
    //Using AGS's character-specific inventories to stock the store shelves.

    //Troy's Inv is the Pawnshop's CD Rack.
    Troy.AddInventory(bLobat); Troy.AddInventory(bMat64); Troy.AddInventory(bPocket);Troy.AddInventory(bStu);Troy.AddInventory(bThirsty);
    //Barneys are the Roastierry Inv (Surprised? Nay? SHOCKED?!)
    cBarneys.AddInventory(bCoffee_A); cBarneys.AddInventory(bCoffee_B); 
    cBarneys.AddInventory(bCoffee_C); cBarneys.AddInventory(bSyrups); 
    cBarneys.AddInventory(bChai); 

    // Toney is Superstore Equipment's Inv
    Toney.AddInventory(bEspressomachine); Toney.AddInventory(bMaker); Toney.AddInventory(bCups); 
    Toney.AddInventory(bNapkins); Toney.AddInventory(bHeadunit);
    Nelly.AddInventory(bMaker);//Nelly is Organique's Equipment Inv
    Julian.AddInventory(bCatfood);// Julian is Organique's Item Inv
```

The player's own belongings use the same trick: `cSlot1` and `cSlot2` are invisible characters whose inventories hold the player's equipment and personal items, which is why the code is full of tests like `cSlot2.InventoryQuantity[Permit_franklin.ID]`.

**(e) Debug "direct flights".** If the global variable `superd` is non-zero, boot skips the menus, picks a character, hands out money, items and plot state, and drops the player straight into a street. With the default `superd == 0` it only sets the public slider defaults. `cheatbiking` is the debug-mode flag.

```bash
sed -n '547,553p' cartlife/GlobalScript.asc | tr -d '\r' | cut -c1-140
echo ...
sed -n '736,741p' cartlife/GlobalScript.asc | tr -d '\r'

```

```output
    //============// Superdirect flight pre-emption //==================//
    //Set the "superd" global variable to bypass the opening screen and get right into testing specifics.
    if (superd != 0){ //We're going straight to it.
        cheatbiking = true;//"cheatbiking" is debug mode.
        
        if (superd == 1){//////////////////////ANDRUS VERSION
            SetGlobalInt(1, 1);
...
    //PUBLIC VERSION:
    if (superd==0){
        cheatbiking=false;
        txtspd_slider.Value=150;
        music_slider.Value=80;
    }
```

## 6. From logo to first day: rooms 25, 1, 23 and 40

The project file names character 0 (`cEgo`, who is Vinny's sprite) as the initial player character and starts it in room 25. That room is the logo screen: an invisible player "walks onto" a region, which fades a stamp graphic in and out and arms timer 9; when the timer expires the player is moved to room 1.

```bash
sed -n '22,33p;42,46p' cartlife/room25.asc | tr -d '\r'

```

```output
function region1_WalksOnto() {
    
    //PlaySoundEx(110, 4);
    // this clip is quite quiet, so turning it up a bit.
    System.Volume = 100;
    aSound110_IntroMusicFadeIn.Play();
    
    FadeObjectIn_NoBlock(Stamp, 0, -5,  0);
    //SetTimer(2, 100); <--- Original Quick Timer
    SetTimer(9, 350); // about 8 seconds, at 40 ticks / second
    FadeObjectOut_NoBlock(Stamp, 100, -5,  80);
}
{
    // if our timer has expired, change to char select
    if (IsTimerExpired(9) == 1) {
        SetNextScreenTransition(eTransitionInstant);
        player.ChangeRoom(1, 165,  193);  
```

This pattern, a region the invisible player stands on plus `IsKeyPressed` polling in the region's `Standing` handler, is how every menu in the game is built. Menus are rooms.

**Room 1, character select.** Left and right arrows cycle a `buffet` index through the three protagonists. `showplayer()` picks the portrait from the persisted play state (unplayed, in progress, left town, lost, won), and Vinny shows a LOCKED plate unless `playstate_bonus` is set. Enter stores the choice in global int 1 and calls `Overwriter()`.

```bash
sed -n '94,95p;100,104p;128,134p' cartlife/room1.asc | tr -d '\r' | cut -c1-165

```

```output
function region1_Standing(){
if (IsKeyPressed(eKeyRightArrow)==1){ //Right Arrow//pagedown//
     if (playstate_bonus==0){//Just the two to start with.
     if (buffet==3){showplayer(1); buffet=1; LOCKED.Visible=false;}//Andrus
else if (buffet<=1){showplayer(2); buffet=2; LOCKED.Visible=false;}//Melanie
else if (buffet==2){showplayer(3); buffet=3; LOCKED.Visible=true;}//Vinny
}
if (IsKeyPressed(13)==1){ //ENTER
if (LOCKED.Visible==true){PlaySound(28);}
else if ((buffet!=0)&&(gOverwrite.Visible==false)){
     if (buffet==1) {SetGlobalInt(1, 1); Overwriter();}//ANDRUS
else if (buffet==2) {SetGlobalInt(1, 2); Overwriter();}//MEL
else if (buffet==3) {SetGlobalInt(1, 3); Overwriter();}//Vinny
}}
```

`Overwriter()` (global script) decides between starting fresh and showing a three-button overlay. A character with no history starts immediately; one in progress offers "begin again" or "continue" (which restores that character's fixed autosave slot); one whose story has ended offers to replay the ending.

```bash
sed -n '14194,14202p;14247,14258p' cartlife/GlobalScript.asc | tr -d '\r' | cut -c1-140

```

```output
function Overwriter(){
  int charstate;
  if (GetGlobalInt(1)==1){charstate=playstate_andrus;}
  if (GetGlobalInt(1)==2){charstate=playstate_melanie;}
  if (GetGlobalInt(1)==3){charstate=playstate_vinny;}
  if (GetGlobalInt(1)==4){charstate=0;}
  
  if (charstate==0){newgame();}
  else if (charstate==1){//in progress
 if (ow2.NormalGraphic==9379){//continue
  if (GetGlobalInt(1)==1){RestoreGameSlot(1);}//in progress
  if (GetGlobalInt(1)==2){RestoreGameSlot(2);}//in progress
  if (GetGlobalInt(1)==3){RestoreGameSlot(3);}//in progress
  }
 if (ow2.NormalGraphic==9381){//View Ending
    StopMusic(); PlaySound(67); FadeOut(5); Wait(40);
    if (GetGlobalInt(1)==1){
      if (playstate_andrus==2){ending_andrus_leaves();}
      if (playstate_andrus==3){ending_andrus_dies();}
      if (playstate_andrus==4){ending_andrus_wins();}
    }
```

`newgame()` is where the three stories diverge. For the chosen character it switches the AGS player character, sets the HUD skin, starting inventory, known conversation topics (`boxadd`), starting cash, which meters are visible, and where the *other* protagonists live as NPCs (`npcAndrus_location` and friends are room numbers). Then it sends the player to room 23. Andrus's branch:

```bash
sed -n '14124,14128p;14130,14133p;14136,14140p' cartlife/GlobalScript.asc | tr -d '\r' | cut -c1-150
echo "..."
grep -n -o 'Money=[0-9.]*; balance_start=[0-9.]*' cartlife/GlobalScript.asc | awk -F: '$1>14114'

```

```output
  if (GetGlobalInt(1)==1){//New Game: Andrus
  npcAndrus_location=0; npcLogan_location=13; npcMelanie_location=17; npcVinny_location=22;
  Andrus.ChangeRoom(25, 0, 0);
  SetGlobalInt(1,1); FadeOut(1); cAndrus.SetAsPlayer(); gGui1.BackgroundGraphic=15;
  upgrade_4.NormalGraphic=1278; upgrade_D.NormalGraphic=1273;//Start with Battery
  m_newcart.NormalGraphic=3794; m_newcart.MouseOverGraphic=3795;
  //cSlot2.AddInventory(WatchD); //Buy your watch
  cSlot2.AddInventory(Lighter); cSlot2.AddInventory(Pocketknife);
  TJ.AddInventory(bDiamond); TJ.AddInventory(bWatchC); stand_look=0;
  //cLogan.ChangeRoom(24, 80, 82); 
  cMelanie.ChangeRoom(24, 103, 82); cEgo.ChangeRoom(24, 163, 82); bmood.Visible=false;
  cigarettes_remaining+=2; cSlot2.AddInventory(Cigarettes);//Add Smokes
  Stand.ChangeRoom(5, 335, 160); // Move Newsstand to Franklin
  SetGlobalInt(52, 1); vitality.Width=110; caffeine.Width=110; nutrition.Width=110; kibbles.Width=110;
...
14134:Money=2250.00; balance_start=2250.00
14155:Money=1560.00; balance_start=1560.00
14169:Money=202.60; balance_start=202.60
14184:Money=5000.00; balance_start=5000.00
```

The starting balances are the difficulty setting: Andrus $2,250, Melanie $1,560, Vinny $202.60 (and $5,000 for Logan, a fourth character reachable only through the F2 key on the select screen).

**Room 23, the main menu,** is a street with three doors (start, credits, quit) and silhouetted pedestrians walking past. Pressing up (key 372) or W while standing on a door region plays the door animation and changes room. The start door goes to room 40 and fills the status meters, with Andrus deliberately starting tired.

```bash
sed -n '37,40p;52,56p' cartlife/room23.asc | tr -d '\r' | cut -c1-150

```

```output
function region1_Standing(){
if (gPanel.Visible==false){
if((IsKeyPressed(372)==1)||(IsKeyPressed(87)==1)){
  if (gInfo.Visible==true)InfoStop();
  startdoor.Animate(1, 1, eOnce, eBlock);  
  player.ChangeRoom(40, 165, 193);
  vitality.Width=110; caffeine.Width=110; nutrition.Width=110; kibbles.Width=110;
  if (GetGlobalInt(1)==1){vitality.Width=40; caffeine.Width=70; kibbles.Width=90; nutrition.Width=90; }//Andrus starts tired
  }
```

Notice `vitality.Width`. The four status meters (vitality, nutrition, caffeine/nicotine, and "kibbles" for Andrus's cat) are not variables. They are GUI bar widgets, and the simulation reads and writes their pixel `Width`, from about 0 to 110, directly. The HUD is the data model.

**Room 40** shows the instructions card, then after a short timer branches on the chosen character and that character's progress integer. For a new Andrus game it plays the train-journey cutscene (a sequence of full-screen button animations on the `gPick` GUI, wrapped in `StartCutscene`/`EndCutscene` so it can be skipped), advances his plot counter to 1 and moves him to the first story room.

```bash
sed -n '47p;51,53p;62p' cartlife/room40.asc | tr -d '\r' | cut -c1-150
echo "          ..."
sed -n '105,107p' cartlife/room40.asc | tr -d '\r' | cut -c1-150

```

```output
if (IsTimerExpired(20) == 1) {
//ANDRUS
if (GetGlobalInt(1) == 1) { //ANDRUS
          if (GetGlobalInt(101) == 0){ // Andrus First Stage
          StartCutscene(eSkipAnyKeyOrMouseClick);
          ...
          EndCutscene();//End Cutscene
          
          SetGlobalInt(101,1); cAndrus.ChangeRoom(11, 173,  158);} //Send Andrus Along to Franklin to Buy Stand
```

## 7. The simulation clock and the body meters

Once the player is in the world, time is driven by a block inside `repeatedly_execute`. Every tick increments `cyclecounter`; when it reaches `clockspeed` (27 ticks, about two thirds of a real second at 40 ticks per second) one game minute passes. The clock is kept in 12-hour form with a separate `ampm` flag.

```bash
sed -n '4325,4345p' cartlife/GlobalScript.asc | tr -d '\r'

```

```output
//////////  Time Time Time Time Time Time timetime--------------------------------------------------------------///
if (isclockrunning == 1) {
  cyclecounter++;
    if (cyclecounter >= clockspeed) {// "27" is the default clockspeed number, it's "2" when asleep
    cyclecounter = 0;
    minute++;
    if (minute >= 60) {
      minute = 0;
      hour++;
      if (hour == 12) {
        if (ampm == 1) ampm=0;
        else ampm=1;
      }
      else if (hour > 12) {
        hour = 1;
      }
    }
    timelabeltext = String.Format("%02d:%02d", hour, minute);
    if (ampm == 0) meridian=( " AM");
    else if (ampm == 1) meridian=( " PM");
}}
```

Because `repeatedly_execute` is suspended during blocking calls, the clock stops whenever a blocking conversation or cutscene is running. The consequence in play is that scripted conversation, which is written as a chain of blocking waits, costs no clock time, while making a sale (which runs under non-blocking GUIs) does.

A 24-hour `milhour` is derived from `hour` and `ampm` each tick in `repeatedly_execute_always`, written out as one `if` per hour. This is characteristic of the code's style throughout.

```bash
sed -n '2806,2809p;2817,2821p;2830,2831p' cartlife/GlobalScript.asc | tr -d '\r'

```

```output
    if (ampm==0){
      if (hour==1){milhour=1;}
      if (hour==2){milhour=2;}
      if (hour==3){milhour=3;}
      if (hour==11){milhour=11;}
    if (hour==12){milhour=0;}}
    if (ampm==1){
      if (hour==1){milhour=13;}
      if (hour==2){milhour=14;}
      if (hour==11){milhour=23;}
      if (hour==12){milhour=12;}}
```

Midnight advances the calendar (`dayofweek` 1-7, `dayspassed` since the start), and for Andrus sets `killpapers` so that today's unsold newspapers turn into worthless old papers once any sale in progress has finished. At 5:00 AM a fresh stack is delivered next to his stand if he still holds the supply contract.

```bash
sed -n '4356,4373p' cartlife/GlobalScript.asc | tr -d '\r' | grep -v '^[[:space:]]*$'

```

```output
if ((hour==5)&&(minute==0)&&(ampm==0)&&(GetGlobalInt(1)==1)){//Special Delivery for Andrus
if (cSlot2.InventoryQuantity[Contract_G.ID]!=0){
Stack.ChangeRoom((Stand.Room), (Stand.x-50),  (Stand.y+3));
Stack.Baseline=999; Stack.Transparency=0;
if ((dayofweek==5)&&(GetGlobalInt(1)==1)){//Friday:COntract's up
SetGlobalInt(12, 3); cSlot2.LoseInventory(Contract_G); cSlot2.AddInventory(Contract_Bad);}}}
if ((hour==12)&&(minute==0)&&(ampm==0)) { dayofweek+=1; dayspassed+=1; minute+=1; 
if (GetGlobalInt(1)==1){//Andrus at Midnight
killpapers=true;//Made this a global bool so that it won't strip the stand during a sale.
}
//Single Newspaper gets Tossed Out
if (cSlot2.InventoryQuantity[paper_single.ID]!=0){cSlot2.LoseInventory(paper_single);}
if (dayofweek==8){dayofweek=1;}//Sunday -> Monday
}
```

The body meters drain on the clock. Every five game minutes while awake, nutrition loses a pixel; every ten, the cat's kibbles do too; on the hour, vitality loses five. Characters with an addiction (Andrus smokes, Vinny runs on coffee) lose two pixels of the `caffeine` bar every five minutes. Global int 108 is a twelve-step latch that stops the same minute being charged on more than one tick. The first two of the twelve near-identical blocks:

```bash
sed -n '4375,4382p' cartlife/GlobalScript.asc | tr -d '\r'

```

```output
if (player.Room!=29){//Awake values
if (minute==0)  {if (GetGlobalInt(108)==0){SetGlobalInt(108,1); vitality.Width-=5; nutrition.Width-=1; kibbles.Width-=1;
if ((GetGlobalInt(1)==1)||(GetGlobalInt(1)==3)){caffeine.Width-=2;}
if (withdrawel!=0){SetTimer(19, 160);}} else {}}

if (minute==5) {if (GetGlobalInt(108)==1){SetGlobalInt(108,2); nutrition.Width-=1; //Kibbles only every ten
if ((GetGlobalInt(1)==1)||(GetGlobalInt(1)==3)){caffeine.Width-=2;}
if (withdrawel!=0){SetTimer(19, 160);}} else {}}
```

Asleep (room 29 is the dream), the signs flip: vitality refills by five an hour while nutrition falls twice as fast.

When a meter bottoms out, the game does not end. It interrupts the player with a full-screen vignette on the `gPick` GUI, bumps the meter back to 10, and forces the cart shut via `closeupshop()`. For Andrus a `heirarchy_tired` / `heirarchy_hunger` counter escalates through progressively worse vignettes each time it happens.

```bash
sed -n '4476,4482p;4484p;4489,4491p;4499,4500p' cartlife/GlobalScript.asc | tr -d '\r' | cut -c1-200

```

```output
//Top Cap
if (vitality.Width > 110)vitality.Width=110;
if (nutrition.Width > 110)nutrition.Width=110;
if (caffeine.Width > 110)caffeine.Width=110;

//Bottom Cap + 'tarred declarations
if (vitality.Width <= 1){ vitality.Width=10;
if (GetGlobalInt(1)==1){//ANDRUS IS TIRED!
  if (heirarchy_tired==0){pick.NormalGraphic=8906; gPick.Visible=true; pick.Animate(283, 0, 3, eOnce); heirarchy_tired=1; Wait(120); }//How much sleep did i get last night?
  else if (heirarchy_tired==1){PlaySound(60); pick.NormalGraphic=4219; gPick.Visible=true; pick.Animate(283, 1, 3, eOnce); heirarchy_tired=2; Wait(120); }//Little Tired
  else if (heirarchy_tired==2){PlaySound(60); pick.NormalGraphic=8293; gPick.Visible=true; pick.Animate(283, 2, 3, eOnce); heirarchy_tired=3; Wait(120); }//Not what I once was.
         if ((GetGlobalInt(52)==0)&&(heirarchy_tired>=3)){//Too Tired to Work
         queposition=0; closeupshop();
```

Meters also feed back into play. Vinny's walking speed is a function of his caffeine bar, as shown in the movement code in section 4 and again here, and for Andrus a low bar sets a `withdrawel` level that periodically shakes the screen with a coughing fit.

```bash
sed -n '4313,4318p' cartlife/GlobalScript.asc | tr -d '\r'
echo ...
sed -n '5666,5669p' cartlife/GlobalScript.asc | tr -d '\r'

```

```output
//////////  Caffeine Addiction------------------------------ ----------------------------------------///
if (caffeine.Width > 81){if (GetGlobalInt(1)==3){cEgo.SetWalkSpeed(5, 5);}}
if ((caffeine.Width < 80) && (caffeine.Width > 50)){if (GetGlobalInt(1)==3){cEgo.SetWalkSpeed(4, 4);}}
if ((caffeine.Width < 50)  && (caffeine.Width > 20)){if (GetGlobalInt(1)==3){cEgo.SetWalkSpeed(3, 3);}}
if ((caffeine.Width < 20) && (caffeine.Width > 10)){if (GetGlobalInt(1)==3){cEgo.SetWalkSpeed(2, 2);}}
if (caffeine.Width < 10){if (GetGlobalInt(1)==3){cEgo.SetWalkSpeed(1, 1);}}
...
////==================// nicotine withdrawel coughing //=========================/
if ((GetGlobalInt(1) == 1)&&(caffeine.Width <=50)&&(caffeine.Width >=21)){withdrawel=1;}
if ((GetGlobalInt(1) == 1)&&(caffeine.Width <=20)&&(caffeine.Width >=11)){withdrawel=2;}
if ((GetGlobalInt(1) == 1)&&(caffeine.Width <=10))                       {withdrawel=3;}
```

## 8. Rooms: one street as the template

All the street rooms follow the script of room 5 (5th & Franklin). On entry, before the fade-in, the room picks its look from the clock: a day or night background frame, matching sprites for the parallax skyline layers and props, and day or night music through `NatMusic()`.

```bash
sed -n '21,22p;33,37p' cartlife/room5.asc | tr -d '\r' | cut -c1-190

```

```output
if ((ampm==0)&&((hour<8)&&(hour>=2))){//Early Morning
SetBackgroundFrame(1); mtns.Graphic=1712; fore_bldgs.Graphic=1713;NatMusic(33); trashcan.Graphic=3720; dumpster.Graphic=4680; pawn_door.Graphic=3598;}
if ((ampm==1)&&((hour>=2)&&(hour<=8))){//late day
SetBackgroundFrame(0); mtns.Graphic=124; fore_bldgs.Graphic=125; NatMusic(32); trashcan.Graphic=3719; dumpster.Graphic=4679; pawn_door.Graphic=3610;}

if ((ampm==1)&&(hour>=9)&&(hour!=12)){//Night
SetBackgroundFrame(1); mtns.Graphic=1712; fore_bldgs.Graphic=1714; NatMusic(33); trashcan.Graphic=3720; dumpster.Graphic=4680; pawn_door.Graphic=3598;}
```

The same handler places story characters. The other two protagonists exist in the world as NPCs, and their `npc*_location` variables decide which room they appear in. In room 5 a non-Andrus player finds Andrus at his news stand (asleep in it at night), and Vinny selling bagels in the daytime if his location says so.

```bash
grep -n -A6 '^if (GetGlobalInt(1)!=1){' cartlife/room5.asc | head -8 | tr -d '\r'
grep -n -A1 'npcVinny_location==5' cartlife/room5.asc | tr -d '\r' | cut -c1-150

```

```output
73:if (GetGlobalInt(1)!=1){
74-    if ((npcAndrus_location==5)&&(Andrus.Room!=5)){
75-      Andrus.ChangeRoom(5, dumpster.X+25, dumpster.Y); dumpster.Transparency=100;
76-      if ((milhour<7)||(milhour>19)){Andrus.LockView(293);}
77-      else {Andrus.UnlockView();}
78-      }
79-    if (Andrus.Room==5){Andrus.Baseline=31;}
82:if ((npcVinny_location==5)&&(GetBackgroundFrame()==0)){
83-Vinny.ChangeRoom(5,390,160); npcVinny_moving=false; Vinny.Animate(0, 3, eRepeat, eNoBlock);}
```

On the way out, every room calls `places()`. Characters are global objects in AGS, so anybody who wandered into this room as a pedestrian has to be cleaned up. `places()` first banishes every character except the player, the stand and the newspaper stack to room 0, then sends characters with a fixed job back to their posts.

```bash
grep -n 'function room_Leave' cartlife/room5.asc | tr -d '\r'
sed -n '868,880p' cartlife/CustomFunctions.asc | tr -d '\r'
echo ...
grep -n -E '^if \((Eddie|TJ|Glenda|Bramford)\.Room==player\.Room\)' cartlife/CustomFunctions.asc | tr -d '\r' | cut -c1-150

```

```output
419:function room_LeaveLeft()
435:function room_LeaveRight()
451:function room_Leave(){places();}
function places(){
  
  int w=0; while (w<Game.CharacterCount){
    if ((w!=player.ID)&&(w!=Stand.ID)&&(w!=Stack.ID)){
      character[w].ManualScaling=false;
      character[w].RemoveTint();
      character[w].StopMoving();
      character[w].ChangeRoom(0, 999, 999);
    }
    w++;
  }
  
  readytosell=false;//just in case...
...
910:if (Eddie.Room==player.Room){Eddie.ChangeRoom(31, 184, 144); Eddie.RemoveTint();}
913:if (TJ.Room==player.Room){TJ.ChangeRoom(33, 247, 143); TJ.RemoveTint();}
915:if (Bramford.Room==player.Room){Bramford.UnlockView(); Bramford.ChangeRoom(18, 437, 105); Bramford.RemoveTint();}
918:if (Glenda.Room==player.Room){Glenda.ChangeRoom(18, 183, 152); Glenda.RemoveTint();}
```

## 9. Getting around: the travel map

Leaving a street at its edge opens the map (room 10). The origin and destination are stored in global ints 98 and 99 as room numbers, and `leavemath()` is a hand-written fare table: for each origin, one line per destination giving the taxi fare and the minutes the trip costs.

```bash
sed -n '225,231p' cartlife/room10.asc | tr -d '\r'
echo "..."
echo "fare table rows: $(grep -c 'taxifare=(' cartlife/room10.asc)"

```

```output
function leavemath(){//Whoa boy.

if ((GetGlobalInt(98)==7)||(GetGlobalInt(98)==2)||(GetGlobalInt(98)==6)||(GetGlobalInt(98)==8)){//Home
if (GetGlobalInt(99)==22){ taxifare=(5.00); timefare=10;}//home to Bridge
if (GetGlobalInt(99)==13){ taxifare=(5.00); timefare=10;}//home to Bridge
if (GetGlobalInt(99)==26){taxifare=(10.00); timefare=20;}//hometo Collins
if (GetGlobalInt(99)==14){taxifare=(10.00); timefare=20;}//hometo Downtown
...
fare table rows: 130
```

The travel GUI offers a taxi, the bus (a flat `busfare` of 75 cents, set at boot) or walking. Each has a click handler in the global script. The taxi handler shows the general shape of every purchase in the game: check `Money`, subtract, add to an expense category, append a line to the `expenselist` ledger list box, show a `TopUp` banner, play a short animation, then change room and restart timer 2 (the pedestrian timer, next section).

```bash
sed -n '12367,12372p;12386,12388p' cartlife/GlobalScript.asc | tr -d '\r' | cut -c1-140

```

```output
function taxi_OnClick(GUIControl *control, MouseButton button){//Cab
if (taxifare>Money){Insufficiency();}
else{
Money-=(taxifare); expense_travel+=(taxifare); expense_total+=(taxifare);
expenselist.AddItem(String.Format("%.2f - Taxi Fare",taxifare));
PlaySound(9); TopUp("Taxi!", String.Format("Spent $%.2f on cab fare!",taxifare));
     if (GetGlobalInt(99)==5) {player.ChangeRoom(5,  158, 158); SetTimer(2, 40);}//Franklin
else if (GetGlobalInt(99)==13) {player.ChangeRoom(13,  84, 113); SetTimer(2, 40);}//Brennan Ave Bridge
else if (GetGlobalInt(99)==14) {player.ChangeRoom(14,  17, 161); SetTimer(2, 40);}//Downtown
```

## 10. Opening the cart

Everything so far is scaffolding for the central loop: stand at your cart, serve whoever walks up. The cart itself is a character named `Stand` (so it can be moved between rooms and animated), with a companion character `lockup` drawn over it as the shutter.

Opening is handled in `repeatedly_execute` by polling the down arrow (key 380) or S while the player overlaps the stand. For Andrus the code first refuses if he is too tired or hungry, handles the special case of picking up the morning's newspaper stack, and otherwise opens: it clears the queue, sets `readytosell`, flips global int 52 to 0, plays the shutter animation, seats the player and arms timer 2 with 40 ticks.

```bash
grep -n 'IsKeyPressed(380)==1' cartlife/GlobalScript.asc | head -1 | tr -d '\r' | cut -c1-110
sed -n '4083,4090p' cartlife/GlobalScript.asc | tr -d '\r' | cut -c1-150

```

```output
4058:if (((IsKeyPressed(380)==1)||(IsKeyPressed(83)==1))&&(gSavegame.Visible==false)&&(gLoadgame.Visible==fals
  if (GetGlobalInt(52) == 1){//Andrus Set Up
   SaleInProgress=0; queposition=0; readytosell=true;
  if (GetGlobalInt(101) ==2){} else if (GetGlobalInt(101) ==1){} else {
        lockup.Transparency=0; lockup.ChangeRoom(Stand.Room); lockup.FollowCharacter(Stand, FOLLOW_EXACTLY, 100);
        PlaySound(18); player.LockView(68); player.Animate(1, 3, eOnce, eNoBlock, eForwards);
        lockup.LockView(70); lockup.Animate(0, 2, eOnce, eBlock, eForwards); SetGlobalInt(52, 0); Wait(5);
        player.LockView(71); player.Animate(0, 3, eRepeat, eNoBlock, eForwards); player.x=(Stand.x); player.y=(Stand.y); 
      player.ChangeView(71); cAndrus.SetIdleView(136, 3); SetTimer(2, 40);}
```

Three pieces of state now define "a sale can happen": `readytosell` (the cart is open), `SaleInProgress` (someone is being served) and the queue array. `repeatedly_execute_always` drops `readytosell` the moment the player and stand are in different rooms.

## 11. Pedestrians, hunger and the queue

**Spawning.** AGS timer 2 is the "commerce timer". Each time it expires, the block below picks one named character from a per-room cast list, decides whether they are hungry, teleports them just off-screen and walks them across. The roll for stopping is `Random(100) + rep`, so reputation directly raises the share of passers-by who become customers.

```bash
sed -n '5028,5037p' cartlife/GlobalScript.asc | tr -d '\r'
echo ...
sed -n '5122,5126p;5134,5146p' cartlife/GlobalScript.asc | tr -d '\r' | cut -c1-150

```

```output
//////////  COMMERCE TIMER------////////////  Commerce TIMER------////////////  COMMERCE TIMER------////////////  COMMERCE TIMER------//
if ((IsTimerExpired(2)==1)&&(player.Room!=25)&&(player.Room!=1)){
int left=(GetViewportX()-20);
int right=(GetViewportX()+300);
int side; side=Random(1);
int cmt;

Character* spawnperson;
int stopchance=Random(100)+rep;//if (stopchance>60){hunger[spawnperson.ID]=1;}if (stopchance<=60){hunger[spawnperson.ID]=0;}
int baserando=Random(4)+1;
...
if (player.Room==34){// Tosheroon Park
cmt=Random(8); 

if (cmt==0){spawnperson=Richard;}//Richard A // SetGlobalInt(16, 1);}
else if (cmt==1){spawnperson=Sebastian;}//Sebastian SetGlobalInt(33, 1);}
else if (cmt==8){
  if ((milhour>10)&&(milhour<20)){spawnperson=Johns;}//Johns SetGlobalInt(25, 1); 
  else spawnperson=Harry;}

if (spawnperson.Room!=player.Room){
  spawnperson.ManualScaling=false; 
  if (stopchance>60){hunger[spawnperson.ID]=1;}
  if (stopchance<=60){hunger[spawnperson.ID]=0;}
  spawnperson.ChangeRoom(player.Room, right, player.y); 
  spawnperson.Baseline=(Stand.y + baserando);
  if (side==0){spawnperson.x=right; spawnperson.chasedown();}
  if (side==1){spawnperson.x=left; spawnperson.chasedown();}
}
```

Each street has its own cast list, and some entries depend on the hour: Officer Johns only appears when `milhour` is between 11 and 19, and someone else takes his slot otherwise. A few rolls call `genericspawn()` instead, which draws from a shared pool of minor characters.

After spawning, the timer is re-armed with a random delay, lengthened at night (`streetlights`) and adjusted per street. By that modifier Downtown gets the most foot traffic and Collins the least.

```bash
sed -n '5326,5337p' cartlife/GlobalScript.asc | tr -d '\r'

```

```output
int roomytimemod;
if (player.Room==5){roomytimemod=0;}//Franklin
else if (player.Room==13){roomytimemod=10;}//Bridge
else if (player.Room==14){roomytimemod=-20;}//Downtown
else if (player.Room==20){roomytimemod=0;}//Florin
else if (player.Room==22){roomytimemod=-5;}//13th
else if (player.Room==26){roomytimemod=40;}//Collins
else if (player.Room==23){roomytimemod=-20; streetlights=0;}//Main Menu

int timermath=(Random(200)+streetlights+roomytimemod);
if (timermath<5){SetTimer(2, 5);}
else if (timermath>5){SetTimer(2, timermath);}
```

The same timer drives the silhouettes on the main menu (room 23 is in the list above with a -20 modifier), which is why `room23.asc` arms timer 2 on load.

**Joining the line.** The comment "NEW POETRY" marks the queue logic in `repeatedly_execute_always`. Each tick it scans every character; a hungry, moving character within a small window of the player's x position stops and takes the next slot in `que[]`, provided the line is not already longer than `maxque` or the character has the `willstopabovemaxcue` property (story characters who must not be missed). Character IDs 0-3 and 12 are the protagonists' sprites and the stand.

```bash
sed -n '2952,2971p' cartlife/GlobalScript.asc | tr -d '\r'

```

```output
    //////////  HUNGER CONQUEST!------------------------------ HUNGER CONQUEST!------//
    //NEW POETRY
    int pedestrianid;
    if (readytosell==true){
      while (pedestrianid<Game.CharacterCount){
      if ((pedestrianid!=12)&&(pedestrianid!=0)&&(pedestrianid!=1)&&(pedestrianid!=2)&&(pedestrianid!=3)){//Customer != Stand or PC
        if (((character[pedestrianid].x-player.x)<= 25)&&((character[pedestrianid].x-player.x)>=-5)&&(character[pedestrianid].Room==player.Room)){//Intersecting
          if ((hunger[pedestrianid]!=0)&&(character[pedestrianid].Moving==true)){//I'm Hungry
             bool worthwaiting=false;//If it's important enough, I'll wait in line.
                if (queposition<=maxque){worthwaiting=true; }
                else if (character[pedestrianid].GetProperty("willstopabovemaxcue")==1){worthwaiting=true;}
                if (worthwaiting==true){
                  que[queposition]=pedestrianid;
                  character[pedestrianid].StopMoving();
                  queposition+=1;//This might push the queposition over the maxcue, but that just means that people will walk by.
          }}

        }}
        pedestrianid++;
    }}
```

`maxque` is recomputed from reputation every tick: one more place in line for every ten points.

```bash
sed -n '3268,3273p;3279p' cartlife/GlobalScript.asc | tr -d '\r'

```

```output
//Maxcue adjustus
if (rep<=0){maxque=1;}
if ((rep>=1)&&(rep<=10)){maxque=2;}
if ((rep>=11)&&(rep<=20)){maxque=3;}
if ((rep>=21)&&(rep<=30)){maxque=4;}
if ((rep>=31)&&(rep<=40)){maxque=5;}
if (rep>=91){maxque=9;}
```

**Serving the head of the line.** At the very end of `repeatedly_execute`, "new poetry 2 aka myturn" looks at `que[0]`. If nobody is being served and no dialogue is on screen, that character becomes `Customer`, their name becomes the string `salebuyer`, and control passes either to a bespoke encounter function (for a handful of story characters) or to the generic `salutation()`.

```bash
sed -n '6337,6354p' cartlife/GlobalScript.asc | tr -d '\r'

```

```output
//new poetry 2 aka myturn
int pedestrianid;
while (pedestrianid<Game.CharacterCount){
  if (readytosell==true){
  if ((hunger[pedestrianid]!=0)&&(que[0]==pedestrianid)&&(SaleInProgress==0)&&(gBuyortalk.Visible==false)&&(gDialog.Visible==false)){//My Turn
    character[pedestrianid].StopMoving(); 
    salebuyer=String.Format("%s",character[pedestrianid].Name); 
    Customer=character[pedestrianid];
    if (((salebuyer==("Vinny")&&(GetGlobalInt(1)==2)))){vinnylove(); Display("Vinnylove();");}
    if (((salebuyer==("Vinny")&&(GetGlobalInt(1)==1)))){Vinnyswitch();}
    else if (salebuyer==("Glenda")){Glendabuy();}
    else if (salebuyer==("Suchin")){Suchinbuy();}
    else if (salebuyer==("George")){Georgebuy();}
    //else if (salebuyer==("Thomas")){Thomasbuy();}
    else if ((salebuyer==("Eddie"))&&(GetGlobalInt(411)==3)){Eddiebuy();}
    else{salutation();}
    hunger[pedestrianid]=0;
    }
```

`salebuyer` is the most important variable in the game. Nearly every customer-specific behaviour, in every module, is a chain of `if (salebuyer==("Name"))` string comparisons rather than a lookup on the character object.

```bash
cd cartlife && for f in GlobalScript.asc CustomFunctions.asc DialogRequesting.asc AskOnly.asc customertalk+listen.asc; do
  printf '%-26s %5s\n' "$f" "$(grep -o 'salebuyer *== *("' "$f" | wc -l)"
done

```

```output
GlobalScript.asc              73
CustomFunctions.asc          244
DialogRequesting.asc          96
AskOnly.asc                  509
customertalk+listen.asc      107
```

When a customer is finished with, for any reason, `breakaway()` (an extender function on `Character`) releases them: it clears `SaleInProgress`, shifts every queue entry down one place and walks the character off one side of the screen. Because the next person is now `que[0]`, the "my turn" block picks them up on the following tick.

```bash
sed -n '1283,1294p' cartlife/CustomFunctions.asc | tr -d '\r'
echo "    ..."
sed -n '1310,1315p' cartlife/CustomFunctions.asc | tr -d '\r'

```

```output
function breakaway(this Character*){
    SaleInProgress=0;
    if (queposition>=1) {
        queposition-=1;
    }
    if (queposition>maxque) {
        //Depleting one more, in case of special encounters pushing the line past the limit
        queposition-=1;
    }

    que[0]=que[1];
    que[1]=que[2];
    ...
      else if (runawayside!=0){this.Walk(800, this.y, eNoBlock, eAnywhere);}}

    else if (this.Room==20){//Florin
         if(this.x<=320){ this.Walk(-10, 192, eNoBlock, eAnywhere);}
    else if(this.x>320){  this.Walk(760, 146, eNoBlock, eAnywhere);}}

```

## 12. Conversation

### The talking-heads GUI

Speech does not use AGS's built-in `Say()`. Conversation is drawn on a custom GUI, `gDialog`, with two portrait buttons (`DBG1` for the player, `DBG2` for the other party), a name label `dName` and a text label `dText`. `TalkPop()` slides it in.

```bash
sed -n '1109,1112p;1120,1128p' cartlife/CustomFunctions.asc | tr -d '\r'

```

```output
function TalkPop()
{
  PlaySound(47); Wait(5); dName.Text=(" "); dText.Text=(" ");
  dpb.Visible=false;
  gDialog.Centre();
  gDialog.Y = 7;
  gDialog.Visible=true;
  DBG0.Animate(32, 0, 1, eOnce);
  DBG1.Animate(26, 0, 2, eOnce);
  DBG2.Animate(26, 0, 2, eOnce);
  nameplate.Animate(55, 0, 2, eOnce);
  dName.Visible=true; dText.Visible=true; nameplate.Visible=true;
}
```

Every spoken line in the game is then the same four-step idiom, repeated thousands of times: set who is animating as speaker and who as listener, assign the text, and call a `blab` function to wait. This is the customer greeting from `salutation()`, followed by the player's reply and the hand-off to the choice menu.

```bash
sed -n '1991,1997p' cartlife/GlobalScript.asc | tr -d '\r'
echo "..."
sed -n '2124,2126p;2130,2133p' cartlife/GlobalScript.asc | tr -d '\r'

```

```output
function salutation(){
  if ((gDialog.Visible==false)&&(SaleInProgress==0)){
SaleInProgress=1;
TalkPop(); Wait(40);
PCListen(); 
if (salebuyer==("Tim")){customertalk();dText.Text=("Look who it is!");blab1();}
if (salebuyer==("Nelly")){customertalk();dText.Text=("Ahoy, vendor!");blab1();}
...
PCTalk(); customerlisten(); 
if (GetGlobalInt(1)==1){dText.Text=("Yes. Hello."); blab1();}
if (GetGlobalInt(1)==2){dText.Text=("Hi there!"); blab1();}
dText.Text=(" "); dName.Text=(" "); //Blank
PCListen(); customerlisten();//Shut up
BuyorTalk();//Let's Go!
}}//End Salutation
```

`PCTalk()` / `PCListen()` animate the player's portrait, choosing a face by character and by condition (Andrus has tired, exhausted and drunk faces keyed off `vitality.Width` and `drunkenness`). `customertalk()` / `customerlisten()` in `customertalk+listen.asc` do the same for the other party: one block per character that starts their mouth animation, plays a random voice blip and writes the name plate. The name plate says "Customer" or "Stranger" until that person's `small_*` familiarity counter is non-zero, which is how the game shows that you have learned someone's name.

```bash
sed -n '61,67p' 'cartlife/customertalk+listen.asc' | tr -d '\r' | cut -c1-150

```

```output
function customertalk(){
if (salebuyer==("Richard")){
if (gDialog.Visible==true){dpb.Visible=true; dpb.Animate(110, 0, (pbspeed), eOnce);}
int tks=Random(2); if (tks==0) PlaySoundEx(84, 3); else if (tks==1) PlaySoundEx(85, 3); else if (tks==2) PlaySoundEx(86, 3);
Richard.Animate(3, 3, eRepeat, eNoBlock); DBG2.Animate(58, 0, 3, eRepeat); 
if (small_Richard==0){dName.Text=("Customer: ");}
else if (small_Richard!=0){dName.Text=("Richard: ");}}
```

`blab1()` to `blab5()` once encoded five reading durations; their bodies are now commented out and all five call `newblab()`, which waits for a time proportional to the length of the text on screen and the text-speed slider, and lets a key or click skip ahead. These waits are blocking, which is what pauses the clock during conversation.

```bash
sed -n '742,745p;755,760p;786p' cartlife/CustomFunctions.asc | tr -d '\r'
echo "..."
echo "blabN() calls across all scripts: $(cat cartlife/*.asc | grep -o 'blab[1-5]()' | wc -l)"

```

```output
function newblab(){
  //1, 4, and 7
String speak;
if (gDialog.Visible==true){speak=dText.Text;}
if (readspeed==1){
if ((speak.Length)>=80){lastframe(400);}//WaitMouseKey(400);}
if (((speak.Length)>=60)&&((speak.Length)<80))lastframe(325);//WaitMouseKey(325);
if (((speak.Length)>=40)&&((speak.Length)<60))lastframe(250);//WaitMouseKey(250);
if (((speak.Length)>=20)&&((speak.Length)<40))lastframe(175);//WaitMouseKey(175);
if (((speak.Length)>=10)&&((speak.Length)<20))lastframe(100);//WaitMouseKey(100);
function blab1(){newblab();
...
blabN() calls across all scripts: 12561
```

### Buy, chat, ask or leave

`BuyorTalk()` shows the four-button menu that follows every greeting. It reuses an AGS dialog (`dBuyorTalk`) purely as a set of on/off flags, then skins the buttons accordingly. If the player's vitality or nutrition is at 20 or below, the chat and question buttons are replaced with greyed variants, and clicking them makes the player trail off mid-sentence. Being tired or hungry mechanically costs you the ability to talk to people.

```bash
sed -n '4827,4830p;4846,4851p;4860,4864p' cartlife/CustomFunctions.asc | tr -d '\r'

```

```output
function BuyorTalk(){
dBuyorTalk.SetOptionState(2, eOptionOff);//"Small Talk" off by default
relationshipcheck();
if (small_customer==0){dBuyorTalk.SetOptionState(1, eOptionOff); dBuyorTalk.SetOptionState(2, eOptionOn);}
if ((vitality.Width<=20)||(nutrition.Width<=20)){
  if (vitality.Width<=nutrition.Width){//Tired before Hungry
  tbQuestion.NormalGraphic=7485;
  tbQuestion.MouseOverGraphic=7484;
  tbNames.NormalGraphic=7487;
  tbNames.MouseOverGraphic=7486;}
Mouse.Visible=true;
gBuyortalk.Centre(); gBuyortalk.Y+=33;
gBuyortalk.Visible=true;
PCListen(); customerlisten();
}//End buyortalk
```

The four buttons call `speakmind()` directly with reserved numbers: 350 small talk, 351 start the sale, 352 ask a question, 353 goodbye.

```bash
grep -n -E '^function tb(Sale|Goodbye)_OnClick|else\{speakmind\(352\)|NormalGraphic==6772\)\{speakmind' cartlife/GlobalScript.asc | tr -d '\r' | cut -c1-120
grep -n -E '^if \(xvalue ?== ?35[0-3]\)' cartlife/DialogRequesting.asc | tr -d '\r'

```

```output
13157:else if (tbNames.NormalGraphic==6772){speakmind(350);}
13159:function tbSale_OnClick(GUIControl *control, MouseButton button){speakmind(351);}
13173:  else{speakmind(352);}}
13174:function tbGoodbye_OnClick(GUIControl *control, MouseButton button){speakmind(353);}
5461:if (xvalue == 350) {//Smalltalk
6203:if (xvalue==351){ // STARTSALE
6725:if (xvalue==352){ // Ask Question
6731:if (xvalue==353){ // Nevermind
```

### `speakmind`: one function, 577 branches

`DialogRequesting.asc` is a single function. Every scripted exchange with a choice in it, from buying a cart upgrade to the custody hearing, is a numbered branch. The first branch shows the pattern: check money, play out the lines, mutate state (`Money`, an expense ledger line, a button graphic that doubles as the "has upgrade" flag) and show a banner.

```bash
cd cartlife
echo "branches: $(grep -c -E 'xvalue *== *[0-9]+' DialogRequesting.asc)   highest xvalue: $(grep -o -E 'xvalue *== *[0-9]+' DialogRequesting.asc | grep -o '[0-9]*$' | sort -n | tail -1)"
sed -n '16,17p;19p;26p;31,33p;36,37p' DialogRequesting.asc | tr -d '\r' | cut -c1-150

```

```output
branches: 577   highest xvalue: 618
function speakmind(int xvalue){
  mouse.Visible=false;
    if (xvalue == 1) {//Battery Yes
  if (Money>= (50.00)){//YES! Battery!
  Money-=50.00; PlaySound(4); expense_upgrade+=50.00;
  expenselist.AddItem("50.00 - Cart Upgrade: Battery");
  PCListen(); customertalk(); gmu_text.Text=("This will just take a minute or two... "); blab2();
  PlaySound(9); TopUp("Cart Upgrade!", "Got Battery for $50.00!");
  PCListen(); customertalk(); gmu_text.Text=("Take good care of it, now. "); blab2();
```

That last detail recurs everywhere: the game often stores facts in the appearance of a GUI control. Whether the cart has a cash register is tested as `upgrade_1.NormalGraphic==6083`.

### Asking about topics

Choosing "ask a question" opens `gTopics2`, a list box of every topic the player has heard of. Topics are added by `boxadd()` whenever a line of dialogue mentions something new, with a notification chime; the list is the player's accumulated knowledge of the town.

```bash
sed -n '1267,1271p' cartlife/CustomFunctions.asc | tr -d '\r'
echo "..."
echo "boxadd() calls across all scripts: $(cat cartlife/*.asc | grep -o 'boxadd(' | wc -l)"

```

```output
function boxadd(String Blah) {
    if (! topicbox.Contains(Blah)) {
        topicbox.AddItem(Blah);
        if (gNotify.Visible==false) {
            PlaySound(99);
...
boxadd() calls across all scripts: 326
```

Selecting a topic calls `askabout()` in `AskOnly.asc`. It is a two-level lookup written as nested `if`s: first on the topic (`keyword`, tagged with a `keytype` of person, place or thing), then on who is being asked (`salebuyer`). The player voices the question through `qverbatim()`, phrased in that protagonist's idiom; the matching branch plays the answer, possibly teaching further topics; and `converse()` reopens the topic list.

```bash
sed -n '687,691p;703,712p' cartlife/AskOnly.asc | tr -d '\r' | cut -c1-170
echo "..."
echo "distinct topics handled: $(grep -o -E 'keyword *== *\(?"[^"]*"' cartlife/AskOnly.asc | sed -E 's/.*"([^"]*)"/\1/' | sort -u | wc -l)"

```

```output
function askabout(){
if (topicbox.SelectedIndex==(-1)){PlaySound(28);}
else{
gTopics2.Visible=false; PlaySound(28);
keyword = topicbox.Items[topicbox.SelectedIndex];
if (keyword==("Andrus")){
  keytype=1;
  qverbatim();
  if (salebuyer==("Andrus")){
customertalk(); PCListen(); dText.Text=("I am from originally Ukraine, but staying in the Breezy's motel now."); blab3(); boxadd("Breezy's Motel");
customertalk(); PCListen(); dText.Text=("Enough money is made to be buying of cigarettes and cat food."); blab5();
customertalk(); PCListen(); dText.Text=("That is all I am needing to be happy."); blab5();
converse();}
else if (salebuyer==("TJ")){customertalk(); PCListen(); dText.Text=("Got to respect that guy."); blab1();
boxadd("Breezy's Motel");
...
distinct topics handled: 82
```

If no branch matches, `disavow()` produces an "I don't know" line written per character, varying by whether the topic is a thing, a person or a place, so even the fallback stays in voice.

```bash
sed -n '38,42p' cartlife/AskOnly.asc | tr -d '\r'

```

```output
if (salebuyer==("Andrus")){
if (keytype==0){//Object
if (dunno==0){customertalk(); dText.Text=String.Format("What is a %s?",keyword);blab1();}
if (dunno==1){customertalk(); dText.Text=String.Format("I'm not knowing of what that is.",keyword);blab1();}}
if (keytype==1){//Person
```

## 13. The sale, part one: what do they want and will they pay?

Pressing the sale button runs `speakmind(351)`, which is a switchboard from customer name to that customer's `...buy()` function. Named regulars have their own, with personal patter and story hooks; everyone else gets `Genericbuy()`.

```bash
sed -n '6203,6204p;6208,6212p;6221,6224p' cartlife/DialogRequesting.asc | tr -d '\r'

```

```output
if (xvalue==351){ // STARTSALE
gBuyortalk.Visible=false; mouse.Visible=false;
if (salebuyer=="Winston"){Winstonbuy();}
if (salebuyer=="Richard"){Richardbuy();}
if (salebuyer=="Stephen"){Genericbuy();}
if (salebuyer=="Jenny"){Jennybuy();}
if (salebuyer=="Troy"){Troybuy();}

if (salebuyer==("Chris"))Chrisbuy();
if (salebuyer==("Sebastian"))Sebastianbuy();
if (salebuyer==("Bramford"))Genericbuy();
```

`Genericbuy()` shows the core path with the least decoration. `Spin()` rolls a random index `dice` into the `Menu` list box (the cart's current menu, another GUI control used as a data structure) and copies the item name into `dots` for the dialogue and `saleitem` for the logic. After one of four randomly chosen request lines, `Customer.sellRandom()` dispatches on `dice`.

```bash
grep -n -E '^  Spin\(\);|^if \(generandom==1\)\{|I.ll have a %s|^Customer.sellRandom' cartlife/CustomFunctions.asc | awk -F: '$1>8359 && $1<8430' | tr -d '\r' | cut -c1-120
echo ...
sed -n '5563,5565p;5576,5577p' cartlife/CustomFunctions.asc | tr -d '\r'
echo ...
sed -n '5519,5527p' cartlife/CustomFunctions.asc | tr -d '\r'

```

```output
8391:  Spin();
8402:if (generandom==1){
8403:  PCListen(); customertalk(); dText.Text=String.Format("I'll have a %s.",dots); blab1();
8424:Customer.sellRandom();}
...
function Spin()
{    if (Menu.ItemCount>=14) {dice=Random(13);}
else if (Menu.ItemCount==13) {dice=Random(12);}
else if (Menu.ItemCount==2) {dice=Random(1);}
else if (Menu.ItemCount==1) {dice=0;}
...
function sellMenu0(this Character*){
if(Menu.Items[0] == ("Coffee")) {saleitem=("Coffee");
if (upCoffee == ("S")){this.sellCoffee_S();}
if (upCoffee == ("A")){this.sellCoffee_A();}
if (upCoffee == ("B")){this.sellCoffee_B();}
if (upCoffee == ("C")){this.sellCoffee_C();}
if (upCoffee == ("D")){this.sellCoffee_D();}}
if(Menu.Items[0] == ("Georgetonian")) {this.sellGeorgetonian();}
if(Menu.Items[0] == ("Chai Tea")) {this.sellChai();}
```

There are fifteen `sellMenuN` functions, one per possible menu row, each mapping the item name in that row to a product-specific sell function. "Coffee" is further split by `upCoffee`, the grade (S, A, B, C or D) of the beans currently loaded.

The product functions are where pricing is decided, and they are all the same shape. `sellCoffee_A`:

```bash
sed -n '4883,4895p' cartlife/CustomFunctions.asc | tr -d '\r'

```

```output
function sellCoffee_A(this Character*){saleprice=CoffeePriceA; saleitem=("Coffee"); 
float pricelimit = ((IntToFloat (this.GetProperty("limitCoffee_A"))) / (IntToFloat (100)))+(repmod);
      if ((salebuyer!="George")&&(salebuyer!="Clarence")){pricelimit+=1.00;}
        if (cups_remaining==0){
          if (Customer==Dick){dickscup();}
          else {nocups();  return;}}
        if (cups_remaining!=0){
          if (pricelimit<saleprice){priceiswrong(); return;}
        else{coffee_bonus_countdown-=1;
         if (saleprice<=0.55){coffee_bonus_countdown-=1; priceis_toocheap(); return;}//Too Cheap?
    else if ((pricelimit>=saleprice)&&(saleprice>0.55)){coffee_bonus_countdown-=1; Make();}
    }//End Good Price
}}
```

Reading that as rules:

1. The customer's ceiling is their `limitCoffee_A` property in cents, converted to dollars, plus `repmod` (reputation / 100, so a 50-reputation vendor can charge 50 cents more), plus a flat dollar for everyone except George and Clarence.
2. No cups: the sale fails with a `nocups()` exchange.
3. Asking more than the ceiling: `priceiswrong()` plays a per-character refusal and the customer leaves.
4. Asking 55 cents or less: `priceis_toocheap()`. Underpricing is also handled as its own outcome.
5. Otherwise: `Make()`.

Each attempt also decrements a per-category countdown (`coffee_bonus_countdown`) that eventually triggers a trivia bonus round.

## 14. The sale, part two: making the order

`Make()` opens the `gMake` GUI, a three-step checklist with a countdown clock and a customer-patience bar. Its setup picks a `Phrase` for the order from six per product, resets the patience bar to its full 320 pixels, and starts the timer immediately.

```bash
sed -n '3631,3635p;3638,3639p;3683,3686p' cartlife/CustomFunctions.asc | tr -d '\r'
echo "..."
sed -n '3789,3792p;3806,3808p' cartlife/CustomFunctions.asc | tr -d '\r'

```

```output
function Make(){
if (gDialog.Visible==false){}
else{
relationshipcheck();
getTemperence();
if (small_customer>0){make_customer.Text=String.Format("%s",salebuyer);}
if (small_customer<=0){make_customer.Text=("New Customer");}
if (saleitem==("Plain Bagel")){make_sold=Bagel_PlainSold;
if (phrasalvaria==0)Phrase=("One Bagel coming up!");
if (phrasalvaria==1)Phrase=("Two sliced bagel halves.");
if (phrasalvaria==2)Phrase=("A round circle of boiled bread.");
...
//Reset Values...
patience.Width=320;
cd_miliseconds=0;
cd_seconds=0;
countingdown=1;//Starting countdown and customer patience immediately!
}}}

```

`getTemperence()` sets how patient this customer is, on a per-name scale from 1 (Stephen) to 10 (Alice), then doubles it and adds two; espresso drinks earn three extra points because they take longer.

```bash
sed -n '981,984p;1015p;1023,1027p' cartlife/CustomFunctions.asc | tr -d '\r'

```

```output
function getTemperence(){
if (salebuyer==("Richard")){temperance=4;}
else if (salebuyer==("Troy")){temperance=4;}
else if (salebuyer==("Stephen")){temperance=1;}//Least patient character!
else if (salebuyer==("Seany")){temperance=3;}
else {temperance=4;}//Anybody else?
if ((saleitem==("Americano"))||(saleitem==("Cappuccino"))||(saleitem==("Mocha"))||(saleitem==("Latte"))){temperance+=3;}//Takes longer->More Patience
temperance+=temperance;//I love you, Jenny.
temperance+=2;//I still love you, Jenny.
}
```

The timer runs in `repeatedly_execute`. While `countingdown` is set, every `temperance` ticks the patience bar loses two pixels, and each elapsed second knocks a random amount off `happiness`. If patience reaches zero the sale is cancelled, reputation drops by five, and the customer walks.

```bash
sed -n '3806,3811p;3831,3833p' cartlife/GlobalScript.asc | tr -d '\r'
echo "  ..."
sed -n '3849,3851p' cartlife/GlobalScript.asc | tr -d '\r'

```

```output
if (countingdown == 1) {
  cd_ticker++;
  cd_miliseconds++;
  patiencedecay++;

if (patiencedecay>=(temperance)){patience.Width-=2; patiencedecay=0;}//5-Highest 1-Lowest
  if (patience.Width<=0){//TOOK TOO LONG!
  PlaySound(60); TopUp("Time Overdrawn!","You took too long!");
  rep-=5; countingdown=0; 
  ...
  infolocker.NormalGraphic=5558; orderbotched=0; 
  SaleInProgress=0;
  StopPop(); 
```

The three steps:

**Step 1, take the order.** Clicking it awards starting happiness by product quality (grade S beans are worth 50, grade D only 5) and builds a four-way multiple-choice question: which of these did the customer just ask for? The wrong answers are other items from your own menu when you have them, and nonsense otherwise. The player answers by typing the letter into a text box (`confirmbox_OnActivate`); a correct letter lights step 2.

```bash
sed -n '1623,1627p;1660,1666p' cartlife/GlobalScript.asc | tr -d '\r'

```

```output
function step1_OnClick(GUIControl *control, MouseButton button){
if (step1.NormalGraphic==5517){PlaySound(41); //countingdown=1;//Started Counting Immediately
//BRAGH!
if ((saleitem==("Coffee"))&&(upCoffee==("S")))happiness+=50; happyupdate();
if ((saleitem==("Coffee"))&&(upCoffee==("A")))happiness+=40; happyupdate();
int quiz=Random(3);

if (Menu.ItemCount<=1){
if (quiz==0){OrderAnswer=("A");
confirm_ac.Text=String.Format("%s[Sloppy Joes",saleitem);
confirm_bd.Text=("One Grey Cat[Appletini");}
if (quiz==1){OrderAnswer=("C");
```

**Step 2, prepare it.** For ordinary items a text box appears and the player must type the `Phrase` exactly; the comparison is case-insensitive and a mismatch is a `Botch()`. Faster typing earns more happiness. This typing test is the game's signature mechanic, and its entire implementation is this handler.

```bash
sed -n '1812,1815p;1820,1823p' cartlife/GlobalScript.asc | tr -d '\r'
grep -n 'UpperCase())!=(Phrase.UpperCase())){Botch();}//Botch!' cartlife/GlobalScript.asc | tr -d '\r'

```

```output
function typebox_OnActivate(GUIControl *control){
  String input = typebox.Text;
  Parser.ParseText(input);
  if ((input.UpperCase())==(Phrase.UpperCase())){//Good?
makercheer.Animate(26, 6, 0, eOnce);
PlaySound(41);
happiness+=15; happyupdate();
if (cd_seconds<=1)happiness+=30; happyupdate();
1866:else if ((input.UpperCase())!=(Phrase.UpperCase())){Botch();}//Botch!
```

Espresso drinks replace typing with a rhythm game on the `gBarista` GUI, implemented in `repeatedly_execute_always`. The state of the mini-game is the sprite number currently shown on the `baraction` button: each phase (grind, tap, sweep, tamp, twist, lock, pour, finish, lid, serve) owns a range of sprite numbers, the arrow keys advance the frame, and reaching the end of a range jumps to the start of the next. Holding the key too long in "grind" or "tamp" runs past the good frames into a botch.

```bash
sed -n '2551,2559p' cartlife/GlobalScript.asc | tr -d '\r'
echo "..."
sed -n '2646,2648p' cartlife/GlobalScript.asc | tr -d '\r' | cut -c1-175

```

```output
    if (gBarista.Visible==true){
      if ((baraction.Graphic>=7855)&&(baraction.Graphic<=7874)){//Grind!
        if (IsKeyPressed(eKeyDownArrow)==1){barint++;
        if (barint>=2){barint=0; baraction.NormalGraphic++; aSound68.Play();
            if (baraction.NormalGraphic>=7875){Botch();}
            }}
        if ((baraction.Graphic>=7867)&&(baraction.Graphic<7875)&&(IsKeyPressed(eKeyDownArrow)!=1)){//Good!
        baristarrow.Animate(264, 6, 1, eRepeat);//Arrow Anim
        aSound58.Play(); baraction.NormalGraphic=7876; barint=0; barkey=0; barlist.NormalGraphic++;}//grind->tap
...
          if (saleitem==("Latte")){
                      if (coco_added!=0){Botch();} if (foam_added>1){Botch();} if (watr_added!=0){Botch();}//Wrong!
                      if ((syrp_added!=0)&&(milk_added!=0)){aSound58.Play(); baraction.NormalGraphic=8135; barint=0; barkey=0; barlist.NormalGraphic++; baristarrow.Animate(264
```

The "finish" phase checks the recipe: a latte needs syrup and milk and must not have cocoa, water or more than one foam.

`Botch()` zeroes happiness, counts the mistake, marks the order as botched (which later forfeits the tip), plays one of three failure animations and docks ten pixels of patience.

```bash
sed -n '3809,3814p' cartlife/CustomFunctions.asc | tr -d '\r'
grep -n 'patience.Width-=10' cartlife/CustomFunctions.asc | tr -d '\r'

```

```output
function Botch(){
  happiness=0;
  countingdown=0;
  PlaySound(60);
  total_botched+=1;
  orderbotched=1;
3963:      patience.Width-=10;
```

## 15. The sale, part three: the cash register and checkout

A correct phrase leads straight into payment. The handler rolls how the customer pays: a $5, $10 or $20 note, the price rounded up to the next dollar in singles (`payup()`), or exact change. `ChangeAnswer` is the change owed.

```bash
sed -n '1825p;1827,1830p' cartlife/GlobalScript.asc | tr -d '\r' | cut -c1-120
sed -n '1831,1833p;1837,1839p;1843,1845p' cartlife/GlobalScript.asc | tr -d '\r' | cut -c1-120

```

```output
if ((cd_seconds<=3)&&(cd_seconds>4))happiness+=10;
int billsize=Random(5);
//if (cheatbiking==true){billsize=5;}//Take this out - testing exactchange
if (billsize==0){billvalue=5.00; cr_Paybill0.NormalGraphic=(Random(7)+9822); changebillslot[0]=1; cr_Paybill0.Visible=tr
if (billsize==1){billvalue=10.00; cr_Paybill0.NormalGraphic=(Random(7)+9806); changebillslot[0]=1; cr_Paybill0.Visible=t
if (billsize==2){billvalue=20.00; cr_Paybill0.NormalGraphic=(Random(7)+9792); changebillslot[0]=1; cr_Paybill0.Visible=t
if ((billsize==3)||(billsize==4)){
  billvalue=IntToFloat(FloatToInt(saleprice, eRoundUp));//Rounding Up
}
if (billsize==5){//Exact Change
                 billvalue=saleprice;
    }
changeclimbing=billvalue;
ChangeAnswer=((billvalue)-(saleprice));//BRAGH!
```

`kaching()` then slides in the `gCashRegister` GUI. It is a physical simulation of a till made of buttons. `changeclimbing` is the value of the money currently lying on the counter and starts as the customer's payment. Clicking a note or coin on the counter (`billtake` / `changetake`) drops it into the drawer and subtracts its value; clicking a denomination in the drawer lays one on the counter and adds its value. The denomination is recognised from the sprite number on the button.

```bash
sed -n '14320,14322p;14327,14330p' cartlife/GlobalScript.asc | tr -d '\r'
echo "  ..."
grep -n 'changeclimbing+=' cartlife/GlobalScript.asc | tr -d '\r' | tr -s ' '

```

```output
function billtake(Button *Control){
  if (Control.Visible==true){
    Control.Visible=false;
    if ((Control.NormalGraphic>=9792)&&(Control.NormalGraphic<=9799)){//20s
      //cr_20.NormalGraphic=Random(7)+9784;}//20s
      cr_20.NormalGraphic=((Control.NormalGraphic-9792)+9784); changeclimbing-=20.00;
      }
  ...
14373: changeclimbing+=20.00;
14396: changeclimbing+=10.00;
14419: changeclimbing+=5.00;
14442: changeclimbing+=1.00;
14503: changeclimbing+=0.25;
14527: changeclimbing+=0.10;//Value?
14551: changeclimbing+=0.05;//Value?
14575: changeclimbing+=0.01;//Value?
```

The first click on the lock opens the drawer. The second submits: if what is left on the counter equals the change owed (to within a cent, since these are floats), the sale completes; anything else is a botch. If the cart has the cash-register upgrade, the correct amount is simply displayed, which is the entire benefit of that upgrade.

```bash
sed -n '14291,14293p;14303,14305p;14312,14317p' cartlife/GlobalScript.asc | tr -d '\r'
echo ...
grep -n 'upgrade_1.NormalGraphic==6083){gm_changeamt' cartlife/GlobalScript.asc | tr -d '\r' | cut -c1-170

```

```output
function cr_lock_OnClick(GUIControl *control, MouseButton button){///
  if ((cranked==false)&&(cr_Lock.Graphic!=9805)){
    cranked=true; 
  else{
      if ((changeclimbing<(ChangeAnswer+0.01))&&(changeclimbing>(ChangeAnswer-0.01))){//Good?
      PlaySound(4); vitality.Width-=1;
      //checkout
      checkout();
      }//End Good Answer
      //Bad!
      else{Botch();}
    }
...
3015:              if (upgrade_1.NormalGraphic==6083){gm_changeamt.Visible=true; gm_changeamt.Text=String.Format("Correct Change:[$%.2f",ChangeAnswer);}//Cash Register Up
```

`checkout()` books the sale. It is one block per product, each doing the same bookkeeping: write a line to the income ledger, add the price to `Money` and the day's and lifetime income, decrement stock (and cups), bump the per-product sold/net/gross counters used by the end-of-day report, cost one pixel of vitality, and raise reputation. `Areatick()` credits the sale to the current street. Better beans earn more reputation per drink.

```bash
sed -n '1198,1202p' cartlife/GlobalScript.asc | tr -d '\r'
echo ...
sed -n '1204,1206p;1211,1214p' cartlife/GlobalScript.asc | tr -d '\r'
echo ...
sed -n '1233,1237p' cartlife/GlobalScript.asc | tr -d '\r'

```

```output
    relationshipcheck();
    String incomestamp;
    if (small_customer>0){incomestamp=String.Format("%.2f - Sold %s to %s",saleprice, saleitem, salebuyer);}
    if (small_customer<=0){incomestamp=String.Format("%.2f - Sold %s",saleprice, saleitem);}
    incomelist.AddItem(String.Format("%s",incomestamp));
...
    if (saleitem==("Coffee")){
    cups_remaining-=1; Areatick(); total_solditems+=1;
    PlaySound(4); vitality.Width-=1; rep+=1;
    if (upCoffee==("A")){
    Money +=(CoffeePriceA);  income_total+=(CoffeePriceA); income_today+=(CoffeePriceA);
    TopSale("Sold a Coffee: Grade A!",String.Format("Made $%.2f!", CoffeePriceA),7);
    coffee_a_remaining-=1; CoffeeSoldA+=1; CoffeeNetA+=(CoffeePriceA); CoffeeGrossA+=(CoffeePriceA);}
...
    if (upCoffee == ("S")){rep+=5; coffee_s_remaining-=1; }
    if (upCoffee == ("A")){rep+=4; coffee_a_remaining-=1; }
    if (upCoffee == ("B")){rep+=2; coffee_b_remaining-=1; }
    if (upCoffee == ("C")){rep+=1; coffee_c_remaining-=1; }
    if (upCoffee == ("D")){rep+=1; coffee_d_remaining-=1; }
```

Then come speed feedback and tips. The tip is drawn from a small table indexed by the final `happiness` band, zero if the order was botched, with hard-coded overrides for particular people. A known customer owed two dollars or less may instead say "keep the change". Tips are tracked per character so the end screen can name your best tipper.

```bash
sed -n '1474,1485p' cartlife/GlobalScript.asc | tr -d '\r'
echo "    ..."
sed -n '1508,1511p;1517p' cartlife/GlobalScript.asc | tr -d '\r'

```

```output
    bt_seconds=nt_seconds;}

    if (orderremaining<=0){//Make Sure Order is Filled
    //TIPS, YO!
    float tipamount=0.00;
    bool keepthechange=false;

    if (orderbotched!=0){tipamount=0.00;}
    if (orderbotched==0){//Make sure: no botches.
    int tipslip=Random(4);
    if (happiness>=90){//Whoa, excellent job!
    if (tipslip==0)tipamount=1.00;
    ...
    if (salebuyer==("The Three")){tipamount=5.00;}
    if (salebuyer==("Basinski")){tipamount=5.00;}
    if (salebuyer==("Toney")){tipamount=0.00;}
    if (salebuyer==("George")){tipamount=0.00;}
    if (salebuyer==("Suchin")){tipamount=1.00; SuchinLove+=1;}
```

Finally `checkout()` calls `priceisright()`, which plays the customer's thank-you, runs `customerbreakaway()` (a name-keyed wrapper around `breakaway()` that stops shopkeepers from wandering out of their own shops), and checks the bonus countdowns. With `que[]` shifted and `SaleInProgress` back at 0, the next tick's "my turn" check serves the next person in line. That closes the loop that began in section 11.

```bash
grep -n 'orderbotched=0; priceisright();' cartlife/GlobalScript.asc | tr -d '\r' | cut -c1-120
sed -n '2732,2734p;2742,2743p;2784,2786p' cartlife/CustomFunctions.asc | tr -d '\r'

```

```output
1564:    if (salebuyer!=(" ")){infolocker.NormalGraphic=5558; orderbotched=0; priceisright();}}
function customerbreakaway(){//==//Let my people go!//==//
if (salebuyer==("Tim")){if (player.Room!=15){Tim2.breakaway();}}
else if (salebuyer==("Stephen")){Stephen.UnlockView(); Stephen.breakaway();}
else if (salebuyer==("Eddie")){if (player.Room!=31){Eddie.breakaway();}}
else if (salebuyer==("Betsy")){if (player.Room!=30){Betsy.breakaway();}}
else if (salebuyer==("Melanie")){if (player.Room!=npcMelanie_location)Melanie.breakaway();}//Only if she's not selling/at home
//Final fuckit command for anybody who isn't included:
else{Customer.UnlockView(); Customer.breakaway();}
```

## 16. Permits and the police

Two of the pedestrians, Deputy Thales and Officer Johns, are customers like anyone else until their purchase is finished. Then `priceisright()` branches: if global int 325 is 0 (permit not yet checked today) the officer asks to see a permit and calls `Permitcheck()`.

```bash
sed -n '2896,2898p;2921,2928p' cartlife/CustomFunctions.asc | tr -d '\r'

```

```output

function priceisright(){ SaleInProgress=0;
if (((salebuyer=="Thales"))||((salebuyer=="Johns"))){
if (bonussed==0){bonus_layover.NormalGraphic=5574; bonussed=1; soda_bonus_countdown=10; Bonus();}}}
//End No-Check
if (GetGlobalInt(325)==0){//Permitcheck
if (GetGlobalInt(1)==1){//Andrus
Thalestalk(); PCListen (); dText.Text=("Alright."); blab1();
Thaleslisten();  PCTalk(); dText.Text=("Yes, have a nice day."); blab1();
boxadd("Permits"); 
Thalestalk(); PCListen (); dText.Text=("Before I go, I'd like to see a permit, please."); blab1();
```

`Permitcheck()` maps the current room to the permit inventory item for that street (the park needs none) and sets `permitted`, then `litmus()` opens the `dThales_plead` dialog with the options that fit: show the permit, or plead. Pleading leads to `speakmind` branches that take a $50 fine.

```bash
sed -n '2858,2861p;2887,2888p' cartlife/CustomFunctions.asc | tr -d '\r'
echo ...
grep -n 'Money-=50.00; total_fines' cartlife/DialogRequesting.asc | head -2 | tr -d '\r'

```

```output
  function Permitcheck(){
if (player.Room==5){//Franklin!
  if (cSlot2.InventoryQuantity[Permit_franklin.ID]==0){permitted=0;}
  else if (cSlot2.InventoryQuantity[Permit_franklin.ID]!=0){permitted=1;}}
if (player.Room==34){//Park!!
    permitted=1;}
...
5263:Money-=50.00; total_fines+=50.00; times_fined+=1; FinePop();
5289:Money-=50.00; total_fines+=50.00; times_fined+=1; FinePop();
```

The player cannot dodge this by shutting the cart when a uniform appears. `closeupshop()`, which every "close" path goes through, first looks for a hungry officer standing at the cart and dismisses everyone waiting in the queue. If an officer is there and the permit is unchecked, it starts the encounter instead of closing (the test excludes Melanie). Otherwise it plays the character-specific closing animation and sets global int 52 back to 1.

```bash
sed -n '9724,9731p;9742,9750p' cartlife/CustomFunctions.asc | tr -d '\r' | cut -c1-170

```

```output
function closeupshop(){
  int copshere=0; int coppercheck=0;
    while (coppercheck<9){
    if (((Thales.x-player.x)<= 20)&&((Thales.x-player.x)>=-5)&&(Thales.Room==player.Room)&&(hunger[Thales.ID]==1)){copshere=1;}
    if (((Johns.x-player.x)<= 20)&&((Johns.x-player.x)>=-5)&&(Johns.Room==player.Room)&&(hunger[Johns.ID]==1)){copshere=2;}
    coppercheck++;}

if (que[0]!=0){character[que[0]].breakaway(); que[0]=0;}
  //Not Getting Away Yet
  if ((copshere!=0)&&(GetGlobalInt(325)==0)&&(GetGlobalInt(1)!=2)){
      TalkPop(); Wait(40); //gBuyortalk.Visible=true;
      if (copshere==1){salebuyer=("Thales"); Thalesbuy(); SaleInProgress=1;}
      if (copshere==2){salebuyer=("Johns"); Johnsbuy(); SaleInProgress=1;}
    }
  else {
    if (readytosell==true){
    readytosell=false; 
```

## 17. Ending the day: brushing teeth, the ledger, the dream

Going to bed is `speakmind(50)`. It fixes the wake-up time at eight hours from now, stops the clock, plays a shower vignette and calls `BrushTeeth()`.

```bash
sed -n '632,637p;640p' cartlife/DialogRequesting.asc | tr -d '\r'
echo ...
sed -n '5794,5800p;5805p' cartlife/CustomFunctions.asc | tr -d '\r'

```

```output
  if (xvalue == 50) {//SLEEEEEEEP

//==// Determining Wakeup Times //==//
wakeminute=minute;
wakehour=milhour+8;
if (wakehour>=24){wakehour-=24; wakeswitch=true;}
//if ((hour+8)<=12){wakehour=(hour+8)-12; wakeswitch=false;}
...
function BrushTeeth(){
  donebrushing=false;
  clockspeed=99999;//'Member to reset this!
  if (GetGlobalInt(1)==1){toothbg.NormalGraphic=8200; brushesleft=15;}
  if (GetGlobalInt(1)==2){toothbg.NormalGraphic=9335; brushesleft=10;}

  gToothbrushing.Visible=true;
  //Function ends in Global Repex when donebrushing==true; and gToothbrushing.visible=true;
```

The toothbrushing mini-game lives in `repeatedly_execute_always`, alternating left and right arrow presses with the same sprite-number-as-state technique as the barista game. When it finishes, `repeatedly_execute` notices and opens the day's ledger with `BreakItDown()`.

```bash
sed -n '2523,2528p' cartlife/GlobalScript.asc | tr -d '\r' | cut -c1-175
echo ...
sed -n '3297,3298p' cartlife/GlobalScript.asc | tr -d '\r'

```

```output
    //Toothbrushing
    if (gToothbrushing.Visible==true){
      if (brushesleft==15){brushkey=0;}
      if ((brushesleft>=1)&&(brushesleft<50)){
      if ((brushkey==0)&&(IsKeyPressed(eKeyRightArrow)==1)&&(IsKeyPressed(eKeyLeftArrow)==0)){brushkey=1;   toothbg.NormalGraphic++; brushesleft-=1;}//
      else if ((brushkey==1)&&(IsKeyPressed(eKeyLeftArrow)==1)&&(IsKeyPressed(eKeyRightArrow)==0)){brushkey=0;   toothbg.NormalGraphic++; brushesleft-=1;}
...
if ((donebrushing==true)&&(gToothbrushing.Visible==true)&&(toothbg.NormalGraphic==8232)){
    FadeOut(50); gToothbrushing.Visible=false; BreakItDown(); Wait(5); FadeIn(10); Mouse.Visible=true;}
```

`BreakItDown()` fills the `gBreakdown` budget screen from the counters that `checkout()` and every purchase have been accumulating: best-selling and most profitable product, income and expense totals by category, and the itemised `incomelist` / `expenselist`. It finds each "most" by comparing one counter against all the others in a single long condition per product. Closing the screen clears the two ledgers and calls `sleep()`.

```bash
sed -n '5602,5604p' cartlife/CustomFunctions.asc | tr -d '\r' | cut -c1-150
echo ...
sed -n '8760,8763p' cartlife/GlobalScript.asc | tr -d '\r'

```

```output
function BreakItDown(){// Setup the Budget Screen
//The biggest pain is determining the "favorites" or "most bought" etc...
if ((coffeeSoldTotal>AmericanoSold)&&(coffeeSoldTotal>PaperSold)&&(coffeeSoldTotal>Bagel_PlainSold)&&(coffeeSoldTotal>ChaiSold)&&(coffeeSoldTotal>Coco
...
function bd_close_OnClick(GUIControl *control, MouseButton button) {//Breakdown
gRecordExpense.Visible=false; gRecordIncome.Visible=false;
PlaySound(28); gBreakdown.Visible=false; Reveal.NormalGraphic=4570; bd_Anim.NormalGraphic=4570; 
Mouse.Visible=false; incomelist.Clear(); expenselist.Clear(); sleep();}
```

`sleep()` is the overnight state transition. It resets the daily counters, ages perishable inventory (a sandwich becomes a day-old sandwich, then an old one), applies any pending cart delivery, lets the police forget they checked your permit, and advances story meters that are meant to move "overnight". Then it sets `clockspeed` to 2 and sends the player to room 29.

```bash
sed -n '1886,1990p' cartlife/GlobalScript.asc | tr -d '\r' | grep -E '^function sleep|income_today=0|Sandwich Shuffle|LoseInventory\((Dayold|Bigwich)\)|Thales needs to see|\(101\)==10\)|\(409\)==1\)|clockspeed=2;|ChangeRoom\(29' | cut -c1-150

```

```output
function sleep(){
    pizzasbought=0; Teejanger=0; income_today=0.00; caffthresh=80; disaster.NormalGraphic=3212; income_tipstoday=0.00;
    //==Sandwich Shuffle!==//
    if (cSlot2.InventoryQuantity[Dayold.ID]!=0){cSlot2.LoseInventory(Dayold); cSlot2.AddInventory(Oldsand);}
    if (cSlot2.InventoryQuantity[Bigwich.ID]!=0){cSlot2.LoseInventory(Bigwich); cSlot2.AddInventory(Dayold);}
    if (GetGlobalInt(325)==1)SetGlobalInt(325, 0);//Thales needs to see your permit again.
    if (GetGlobalInt(101)==10)SetGlobalInt(101, 11);//Andrus Plot Advance
    if (GetGlobalInt(409)==1)SetGlobalInt(409, 2);//Advance Bigby sign to Graffiti
      clockspeed=2;//RAMMING SPEED!
      player.ChangeRoom(29, 182, 149);
```

Room 29 is a small playable dream. On entry it picks one of two dreamscapes at random, with story-specific overrides (on Melanie's first night her daughter and ex-husband appear in it), and arms room timer 4. When that expires the player wakes: the clock is set to the precomputed wake time, vitality is refilled and nutrition is set to 10, so every morning starts rested and hungry. The player is then placed at home, or beside the cart if Andrus has been evicted from the motel.

```bash
grep -n -E 'int dreamlotto|Dream Zero|Dream One' cartlife/room29.asc | tr -d '\r'
echo ...
sed -n '268,269p;300,303p' cartlife/room29.asc | tr -d '\r'
echo ...
grep -n -E '^if \(kickedout[!=]=1\)\{player.ChangeRoom' cartlife/room29.asc | tr -d '\r' | cut -c1-150

```

```output
39:int dreamlotto = Random(1);
40:if (dreamlotto==0){// Dream Zero: Blank City
44:else if (dreamlotto==1){//Dream One: Big Stand
...
function room_RepExec(){
if (IsTimerExpired(4)==1){//You know, just... Wake up!
//==// Revised method: 8 hours every night, no matter what. //==//
hour=wakehour; minute=wakeminute; 
vitality.Width=110;//full rest
nutrition.Width=10;//Hongry
...
352:if (kickedout!=1){player.ChangeRoom(7, 284, 138); SetGlobalInt(10, 1);}//GLOBAL(10)= 0:Outside  - 1:Inside
353:if (kickedout==1){player.ChangeRoom((Stand.Room), (Stand.x), (Stand.y)); SetGlobalInt(10, 0); }}
```

## 18. Story state, saving and endings

The three stories are layered over this loop as integer state machines. Andrus's is global int 101, Melanie's is the variable `melplot`, and side plots have their own meters (the comment block for global int 411 in section 5 is one). Room scripts and `speakmind` branches test these numbers to decide what is present and what people say, and time-based checks advance them. Melanie's custody thread, for example, turns on a clock test in `repeatedly_execute_always`: was Laura picked up from school by 17:00?

```bash
sed -n '9,13p' cartlife/DialogRequesting.asc | tr -d '\r'
echo ...
sed -n '2945,2950p' cartlife/GlobalScript.asc | tr -d '\r' | cut -c1-175

```

```output
//==// MELPLOT //==//
//0: Tutorial
//1: Tutorial Finished, Sunday night.
//2: First Custody Meeting Missed
//3: Fist Custody Meeting Attended
...
    if ((melplot==1)&&(GetGlobalInt(1)==2)&&(dayofweek>=1)&&(milhour==17)&&(mel_pickeduptoday==false)){//Mel didn't pick up Laura: BAD MOM
    //Display("Declaring Melanie to be a bad mother by changing her global plot variable to a value of ninety nine.");
    melplot=99;}
    if ((melplot==1)&&(GetGlobalInt(1)==2)&&(dayofweek>=1)&&(milhour==17)&&(mel_pickeduptoday==true)){//Mel did pick up Laura: GUT MOM
    //Display("Declaring Melanie to be a good mother and increasing her global plot variable to a value of two.");
    melplot=2;}
```

Andrus's rent works the same way. Room 7 (his motel) sends the landlord George to the door when a week boundary arrives and `rentpaid` has not kept up; paying is a `speakmind` branch.

```bash
sed -n '59,61p' cartlife/room7.asc | tr -d '\r' | cut -c1-150
grep -n 'Paid Rent' cartlife/DialogRequesting.asc | tr -d '\r' | cut -c1-150

```

```output
if ((dayspassed==7)&&(rentpaid==0)){George.ChangeRoom(7, 455, 140); George.UnlockView(); George.Walk(337, 140, eNoBlock, eAnywhere);}
if ((dayspassed==14)&&(rentpaid==1)){George.ChangeRoom(7, 455, 140); George.UnlockView(); George.Walk(337, 140, eNoBlock, eAnywhere);}
if ((dayspassed==21)&&(rentpaid==2)){George.ChangeRoom(7, 455, 140); George.UnlockView(); George.Walk(337, 140, eNoBlock, eAnywhere);}
7756:TopUp("Paid Rent","Lost $119.00"); rentpaid=1; Money-=119.00; expenselist.AddItem("119.00 - Rent"); expense_misc+=119.00;
```

**Saving** is automatic, one slot per character. `speakmind(155)` writes the slot and marks the character "in progress" in `data.dat`; it also restores `clockspeed` to 27 after the night.

```bash
sed -n '1728,1729p;1740,1742p' cartlife/DialogRequesting.asc | tr -d '\r'

```

```output
else if (xvalue == 155) {//Save Game // Saveprompt
clockspeed=27;//Reset Clockspeed to 27 after sleeping
  if (GetGlobalInt(1)==1){SaveGameSlot(1, "Andrus Autosave"); NewScores(0, 1);}
  if (GetGlobalInt(1)==2){SaveGameSlot(2, "Melanie Autosave"); NewScores(1, 1);}
  if (GetGlobalInt(1)==3){SaveGameSlot(3, "Vinny Autosave"); NewScores(2, 1);}
```

**Endings** are reached from story code, not from running out of money. Each one records an outcome with `NewScores(character, state)`, most of them also overwriting the character's autosave, using the play states from section 5 (2 left town, 3 lost, 4 won). Winning with one protagonist while the other is in progress also sets the bonus flag. These are the places an outcome is written from story code:

```bash
cd cartlife && grep -n -E "NewScores\([0-3], *[2-4]\)" room*.asc DialogRequesting.asc | tr -d '\r' | sed 's/: */: /' | cut -c1-120

```

```output
room15.asc: 252:NewScores(0, 3);//Char Select screen set to Northbound
room28.asc: 152:  SaveGameSlot(1, "Andrus Autosave"); NewScores(0, 2);//Left on a Train
room7.asc: 337:           SaveGameSlot(1, "Andrus Autosave"); NewScores(0, 4);//Good Ending
DialogRequesting.asc: 7914:          NewScores(0, 3);//Char Select screen set to Northbound
DialogRequesting.asc: 9414:    SaveGameSlot(2, "Melanie Autosave"); NewScores(1, 2);//Left on a Train
DialogRequesting.asc: 9449:    SaveGameSlot(3, "Vinny Autosave"); NewScores(2, 2);//Left on a Train
DialogRequesting.asc: 11979:  SaveGameSlot(2, "Melanie Autosave"); NewScores(1, 3);//Ooch.
DialogRequesting.asc: 12008:  SaveGameSlot(2, "Melanie Autosave"); NewScores(1, 4);//Yay!
```

After the ending scene, `curtaincall()` sets `gameover` (which both tick functions test first, switching the simulation off), re-reads the scores and shows the `gEnd` summary screen built from the lifetime counters. From then on the select screen in room 1 shows that character's outcome portrait, and `Overwriter()` offers to replay the ending.

```bash
sed -n '9602,9606p' cartlife/CustomFunctions.asc | tr -d '\r'
echo ...
sed -n '2516p;3246,3247p' cartlife/GlobalScript.asc | tr -d '\r'

```

```output
function curtaincall(){
  FadeOut(64);
  CloseAll();
  gameover=true;
  ReadScores();
...
    if (gameover==false){
function repeatedly_execute() {
if (gameover==false){
```

## 19. Debug mode

The header comment of the global script invites readers to explore with the debug keys. There are three ways in: set the `superd` global variable for a "direct flight" at boot (section 5), set `cheatbiking` to true in the editor, or type the password into the in-game text box, which is checked in `TipBox_OnActivate`.

```bash
sed -n '13275,13280p' cartlife/GlobalScript.asc | tr -d '\r'

```

```output
  } else if (input.UpperCase()==("COLLIER")) { //cheatbike: Activate!

    cheatbiking=true;
    gTip.Visible=false;
    TipBox.Text=("0.00");
    TopUp("Debug Keys Enabled.","Debug keys enabled.[Press z for debug menu.");
```

With the flag set, a block near the end of `repeatedly_execute` polls a set of keys by raw key code. The inline comments name most of them.

```bash
sed -n '6080,6330p' cartlife/GlobalScript.asc | tr -d '\r' | grep -E '^ *if \(+IsKeyPressed\([A-Za-z0-9]+\)==1\)[^/]*//' \
  | sed -E 's/^[^/]*IsKeyPressed\(([A-Za-z0-9]+)\)==1\)[^/]*\/\/ *(.*)/\1: \2/' | cut -c1-80

```

```output
78: Trivia Cheater
eKeyF1: F1
66: Char Conveyor
88: Press X for Info
89: The "Y" Key.
85: Press U for dialog escape
75: K for sleep
379: 71=G for ending the game //Is now "END KEY"
eKeyF8: F8 = Buy Game!
eKeyF10: F10 = Mel Truck Scene
90: {Z for Warp / Menu
383: Press DELETE for Fullstock
```

Key 90 (Z) opens the two `Warp` GUIs, whose buttons teleport between rooms, adjust money and time, and step the plot counters forward and back. Key 383 (Delete) hands out every permit and a fully stocked cart for the current character, which is the fastest way to reach the vending loop described in sections 10 to 15.

## 20. Summary: where things live

The game is a state machine spread over GUI controls, numbered global integers and named global variables, driven by two tick functions and a large set of click handlers. In the order the walkthrough followed:

| Stage | Where | Key names |
|---|---|---|
| Declarations of every character, GUI, item, dialog, variable | `Game.agf` | custom properties `limit*`, `pref*` |
| Boot | `CustomFunctions.asc`, `GlobalScript.asc` | `game_start`, `ReadScores`, `superd` |
| Title flow | `room25`, `room1`, `room23`, `room40` | `Overwriter`, `newgame` |
| Clock, meters, movement, spawning | `GlobalScript.asc` `repeatedly_execute` | `clockspeed`, `vitality.Width`, timer 2 |
| Queue, mini-game input, register animation | `GlobalScript.asc` `repeatedly_execute_always` | `que[]`, `hunger[]`, `readytosell` |
| Street scenery and NPC placement | `roomN.asc` | `on_event`, `places` |
| Travel | `room10.asc`, `GlobalScript.asc` | `leavemath`, `taxi_OnClick` |
| Greeting and choice menu | `GlobalScript.asc`, `CustomFunctions.asc` | `salutation`, `BuyorTalk` |
| Talking heads and pacing | `customertalk+listen.asc`, `CustomFunctions.asc` | `customertalk`, `PCTalk`, `newblab` |
| Scripted outcomes | `DialogRequesting.asc` | `speakmind(xvalue)` |
| Topics | `AskOnly.asc` | `askabout`, `boxadd`, `disavow` |
| Pricing and order making | `CustomFunctions.asc`, `GlobalScript.asc` | `sellCoffee_A`, `Make`, `Botch`, `typebox_OnActivate` |
| Payment and bookkeeping | `GlobalScript.asc` | `kaching`, `cr_lock_OnClick`, `checkout` |
| Permits | `CustomFunctions.asc` | `Permitcheck`, `litmus`, `closeupshop` |
| Night | `DialogRequesting.asc`, `GlobalScript.asc`, `room29.asc` | `speakmind(50)`, `BreakItDown`, `sleep` |
| Endings and persistence | `CustomFunctions.asc` | `curtaincall`, `NewScores`, `data.dat` |

Four habits of the code explain most of what looks strange on a first read, and knowing them makes the rest navigable:

1. **GUI controls are the data.** Meters are bar widths, upgrades are button graphics, the menu and the ledgers are list boxes, and mini-game progress is a sprite number.
2. **Identity is a string.** `salebuyer` holds the current customer's name and behaviour is selected by comparing it, one `if` per person, in every module.
3. **Plot is a number.** `GetGlobalInt(1)` selects the protagonist and other indices hold story progress; the only legend is the comment block in `game_start`.
4. **Dialogue is code.** Every line of speech is a statement of the form *animate speaker; set text; wait*, so the writing and the logic are the same text.

To check that the excerpts in this document still match the source, run `uvx showboat verify walkthrough.md` from the repository root.

