[Reactor-Controller-FSM.md](https://github.com/user-attachments/files/34496340/Copy.of.Exploiting.Barotrauma.s.Electrical.System.By.Building.a.Custom.FSM.md)
# Exploiting Barotrauma's Electrical System By Building a Custom FSM
Wooooooo\! What a weird start to this project. 
I’ll have to disappoint but this breaks that cliche of “Man builds poorly optimised CPU out of 5000 small LEGOS”

This project features a **programmable FSM** (Finite State Machine) implemented using 6 components representing up to 3 states. This is a **walkthrough of its development** and overall application as an advanced **reactor controller** for overclocking, underclocking and normal operation made up of 13 components as shown in the finalised diagram below.

<img width="1024" height="950" alt="circuitsimage" src="https://github.com/user-attachments/assets/869e3e58-1e92-4133-9009-dde641a6b820" />

**Feel free to skip pages:** [past the intro:](#Limitations-with-regular-controllers),** or [to the FSM logic](#Onto-the-drawing-board) 

Context. I felt motivated to outdo on my existing overclocker in barotrauma from 2024\. Fundamentally it felt incomplete as it relied on the in-built autoreactor to stabilise the fission rate during overclocking. 

I’ll first go over this old system, it abuses several mechanics in the barotrauma power system in which it quickly becomes evident why standard reactor controller designs fall short \-and why we had to build our own.


### Wiring mechanics and power distribution

Normally the reactor is limited to a single **power output**. There is a way around it, I dub it **“wire shorting”**. When two wires are connected to the same powered component such as the engine, a lamp, or the navigation terminal. They share power between each other indirectly through said “short” ; the wires need only stem from \*some\* distributor e.g a junction box, ship batteries or as you’ll see, **relay power out** slots. I’m sure plenty of stealth submarine **designers have pulled hair** unknowingly looking for these “shorts” mysteriously powering the sub.  


### The old system  
   
The original overclocker uses a switch into a signal check outputting 100 to the reactor’s turbine output with the **autoreactor enabled**, allowing for **overvoltage** equal to the **maximum power output** of a given reactor once the fission rate stabilised.   
Which normally is unusable as such insane overvoltage would instantaneously fry every junction across the submarine. Which brings us to the second exploit.. the.. \***docking hatch**\* 

It’s unique in the fact that it is an indestructible **power distributor** which is immune to **electrical damage** since it’s essentially a door. But like every component in the game, it has a limited wire output of 5\. You totally could wire the reactor straight to the docking hatch but now you’d only have 4 outputs while overvolting.. n-nothing? My design incorporates a short on the engine, bridging the reactor to the docking hatch, which **saves** us **one output** slot. 

The remainder of the design incorporates **three relays** rated for 1000W each, for two they’re “powered in” to **step down power** for the rest of the sub. The third we abuse the “power out” slot and create a **“short” here** creating more connections between the docking hatch and additional machines of interest, such as ballast pumps or deconstructors e.t.c. for overvolting, plus the navigation terminal. You wouldn’t want to be overclocking the engine while blindfolded without steering  \-the step down relays become unstable under overvoltage causing power drops 33\~55% of their rated outputs.

If you had the luxury of battery power you could shut off the relays completely, freeing up more power to overvolt other components while depleting battery power for everything else. Although at the time of writing **overvoltage benefits** have been **capped to 200%** so there is little to be gained by doing this unless your submarine has an awful reactor. 

The downsides of this design stems from the **autoreactor**, when adding more than two fuel rods, the reactor must first be **manually calibrated** otherwise calibrations will take too long and the reactor is likely to explode before it can stabilise. Plus the minor downside of the engine now being liable for connecting the reactor and the rest of the ship. ( If the engine fully breaks at 0% durability power is no longer transferred through it ).


### Limitations with regular controllers

Reactor controllers calculate their outputs based on two formulas. One for Turbine output based on load\_out and max power out. The other for Fission rate based on turbine output and an existing constant. 

**You see the problem right?** The formulas are based on “load\_out” which means they are strictly tied to the exact power demands of the submarine. There is a workaround which existing designs use, you multiply load\_out by some overvoltage coefficient 1.0x to 2.0x. 

This comes with a hidden drawback, the coefficient must stay safely under the bounds of 1.7x to avoid the risk of mismatching the turbine and fission rate, if the **turbine output** is ever calculated to be **higher than the max output** of the reactor, it will be **capped at 100**. But the **fission rate** formula which is derived from this won’t know any better, it will **continue climbing** relative to this mismatch potentially leading to fires or a meltdown swiftly followed by spontaneous combustion of the reactor.

### Past Optimisation Problems

Before we move on. A bit of history, which later becomes integral to the design of my controller. My friend was working on their **light controller** with smooth transitions and streaming over voice chat. Each light in barotrauma accepts four inputs, RGB and A. 

They installed 5 preset colours. Given the limitation of one value per memory component, that’s **20 entire memory components** centralised around **four multiplexors** for routing each set. Limiting this design to the sandbox. (A legacy version of their project prior to the introduction of multiplexors and demultiplexors can be found here [light smoother 2.0](https://steamcommunity.com/sharedfiles/filedetails/?id=3211183061))

Then.. I had an idea\! What if we could store everything on one single memory component? We may be limited to one value but it doesn’t limit how long it can be\! We could use a regex component to filter out the values but I had no idea how you would choose your values since on the surface regex is a **static expression** with no control/input besides data in.

After some experimentation I figured out the “Set\_Output” of the regex could be used to select for **named capture groups**. The rest is history. The values on the single memory component are held as one long string of numerical values identified with ```255R1, 255G1, 255B1 .. 123A5``` .. matched using ?(\\d+)**ID** on the regex. Once combined with our new selector and name tags  **\<?1 to 5\>**.  The regex is essentially acting like an indexer.

Notice that while we have found a way to index the string, how do we separate out the different categories of values? Regex only has one output and all of them would **share the same output** channel so contamination seems inevitable.

I tried to devise a workaround by adding a “1 to 9” demultiplexor. It has an **inbuilt counter** that determines the current output and can be read, set, and incremented. By forwarding the selected output back to the regex selector and with a clock incrementing the DMX we can sort the output into **9 buckets**.

I would later recognise this pair to be the fundamental core of my reactor controller, but for the **current application** it was impractical to differentiate each colour set when all outputs of the DMX were active. Not to mention with only 9 values means we can only have 2 colour sets, far from the original 5\. 

So, the demultiplexor and clock were scrapped. It was replaced by four regex components configured to only search for their respective sets for Rx,Gx,Bx,Ax. By selecting 1-5 on the regexes we get a set of 4 outputs, a set for each colour. That compresses 24 components into 5, one memory \+ four regexes.

Also I was lazy and didn’t want to handle strict string formatting, so in front of each capture group was a lookahead **(?=.\*)** Which is dubiously resource intensive , especially with four parallel regexes. But as long as the number of colours remains low it was a worthwhile tradeoff. 

### Onto the drawing board

So continuing on, since we can’t rely on normal reactor controllers we need to design our own custom controller while keeping it seaworthy; a nasty optimisation problem, bringing up that regex & demultiplexer pair.

I had actually found myself playing with the idea that we could repurpose the DMX to update itself through “set output” which would enable the idea of a “jump” when indexes are placed in memory and forwarded back.

Think of it like this, there is an **inspector checking 9 buckets** {1,2,3,4,5,6,7,8,9} one by one on a loop. We can skip over {3,4} given **2** has a note that says skip **to bucket 5**, let's call this jump A. Then if we add another jump B, conditional since we don’t want an infinite loop. We can go from **9 to 3**, skipping over {1,**2**} flushing it down the toilet with its respective jump (You\! AND your jump\!) until the condition is no longer fulfilled, in which case it loops back. 

We can call these alternating paths “states” in our FSM:  

Default state → {1,**2**} → {5,6,7,8,**9**} →  
Conditional state → {3,4} → {5,6,7,8,**9**} → 

Here is a physical demo featuring {1,2,3,**4**} with 3 values and one jump **4** → 2\. With 6 text displays showing all values at 1 Hz.
<img width="800" height="160" alt="JumpDemo" src="https://github.com/user-attachments/assets/28e8aab8-3b9c-47ee-ae9d-437445b13d84" />


Let’s try adding a third state → {5,6} → {7,8,**9**} With a jump **8** → 5\. Notice the problem?  
The mistake I had made was not adding an extra jump to state B to avoid C causing the conditional states B and C to merge together. Running state B would also unconditionally run state C.

Fixing this issue and moving the jumps to make space for C leaves us with the following states:
```
Default state → {1,2} → {7,8,9} →
(The jump 2 has been moved to 7)

Conditional State B → {3,4} → {7,8,9} →
(A jump 4 → 7 has been added)

Conditional State C  → {5,6} → {7,8,9} →
(Jump 8 occurs before Jump 9. Higher priority on state C)
(If both conditions are true state C is prioritised)

Features the following jumps:
2 → 7 , 4 → 7 , 8 → 5 , 9 → 3
```

### Putting logic into practice

So how is this even set up? Where do you even begin?  
Let’s think about functionality starting with underclocking, let's look at the formulas.

$$ 
Turbine_{Out} = \frac{Power\ Load_{Out}}{Max\ Power\ Output}
$$

               (Do note that Max Power is set 100x smaller such that Turbine Out is 0-100 and not 0-1.0)

If we want to underclock. We increase the constant “Max Power Output” by some factor, in this case, 5/3. Such that the turbine output is 40% lower.   
But first, assuming our **default max power** is 3 kw, we set bucket1 of the default state as 30\. Then we can place our calculated value, 50, in bucket 3 of state B. Meanwhile Turbine Out can be **forwarded** to the formula below:  
$$Fission\ Rate = Turbine_{Out} \cdot \frac{75}{Fuel_{Out}}$$

On the other hand overclocking is done differently. You might remember the limitations connected to the formulas using “Load out”, hence requiring custom input.. 

This is done by forwarding a preset turbine output, 100, to the fission rate formula and turbine value directly to the reactor. By **avoiding a constant** for “Max Power” the turbine output calculation **becomes null**, and **nothing is forwarded on**. We can place 100 in bucket 5 of state C.

(I do want to mention that 75 as a constant, it’s been placed in bucket 7 where all states converge. Such that it is always available for fission calculations.)

With the **values added** our states should now look something like this:  
```
Default state → {30,2} → {75,8,9} →

Conditional State B → {50,4} → {75,8,9} →

Conditional State C  → {100,null} → {75,8,9} →
```
(Since **bucket 6** is unused, another constant 2001W is placed here. The jumps are shifted accordingly)
After we add in the jumps “**2** → 6 , **4** → 6 , **8** → 5 , **9** → 3”:
```
Default state → {30,6} → {2001,75,5,3} →

Conditional State B → {50,6} → {2001,75,5,3} →

Conditional State C  → {100} → {2001,75,5,3} →
```
With values and jumps added, all 9 buckets lined up will look like this:  
```
{30,6,50,6,100,2001,75,5,3}
```

### Connecting everything together  
You might have realised by now, the “inspector” is actually the regex and demultiplexor pair. The buckets represent values in memory. Every time the clock increases the counter, the “inspector” matches the value with the corresponding number tag then sends it to the corresponding output 1-9.

Upon landing on a jump condition, the output is forwarded through a relay, then back. Updating the counter and moving the “inspector” along. The relays physically enable or disable jumps and are configured by signals from sensors.

Meanwhile each value in memory needs an identifier such that it can be matched by the regex. Here. I've labelled each value with a letter.  
The string will in memory look something like this:  
```
30a 6b 50c 6d 100e 2001f 75g 5h 3i
```

#### Optimising Regex Expressions  
The original regex expression used in the light controller looked like this (?=.\*?(?\<1\>\\d+)**a**) for each value. It was then copy pasted as follows:  
```
(?=.*?(?<1>\d+)a)(?=.*?(?<2>\d+)b) until we get to (?=.*?(?<9>\d+)i)
```
The major advantage and drawback of this expression are the lookaheads. On one hand, it makes the expression extremely modular and flexible. Specific values can be matched regardless of order or spacing. 

\-With the massive downside of being inefficient “O(n^2)”.    
So I've gotta say “*That's alotta of lookaheads..*” 

Although those strengths were particularly useful when it came to dropping the entire catalog of RGBA light values into that light controller; Varying light parameters didn’t need to be strictly ordered and can easily be added to the search pattern.

Luckily, the string we are working with has no need to be dynamic or modular. We can just **strictly match** the whole string itself by including the spaces **\\s** with each block like this (?\<1\>\\d+)**a**\\s

So an optimised expression “O(n)” would look something like this:   
```
(?<1>\d+)a\s(?<2>\d+)b\s(?<3>\d+)c\s(?<4>\d+)d\s(?<5>\d+)e\s(?<6>\d+)f\s(?<7>\d+)g\s(?<8>\d+)h\s(?<9>\d+)i
```

#### Physical Wiring  
With all the theory and design **wrapped up**, it’s time to actually wire this up.   
Feel free to configure the relays however you see fit. Personally I use a manual switch for overclocking. Underclocking is set automatically by targeting a baseline power load on a “greater than” component so the reactor “goes to sleep” when no subsystems are drawing power.

If you do intend to build this, do remember to change the power values “30**a**” and “50**c**” to ones which match your specific submarine. (reminder to ensure all math components have a **non-zero timeframe** and for regex components don't forget to enable ‘use capture groups’ and leave ‘false output’ empty) 

<img width="1306" height="663" alt="image" src="https://github.com/user-attachments/assets/f998b864-b22a-41ad-bc89-c6a291588d24" />

This circuit box diagram is for reference only; contains additional connection details as an extension to the original diagram. Try to **avoid using circuit** boxes because they are **incompatible** due to external relays even when forwarding through the given outputs.
(Also while you technically could use an existing junction box to forward signals back to save components, failure of the junction box could jeopardise the reactor controller so I wouldn't.)

(The second and third relays shown on the right are placeholders for external relays)  
The third relay is an existing power down relay and handles unconditional jumps. (This relay is always connected - does not change state)
The second relay is an existing “shorted” relay on “power out” and is controlled by a switch. Both are external. It handles jumps on output 8\. ("Shorts" bypass the relay entirely and as such the state of the relay does not affect power.)

With that finished, I imagine that concludes our time together. Thanks for reading, I really enjoyed it. Oh- And before you go. If you wanted to test it I’ve put up a [demo](https://steamcommunity.com/sharedfiles/filedetails/?source=Facepunch.Steamworks&id=3834851710) on the vanilla submarine “the Dugong”, that’s what the current values are configured for. Have fun. If you have any further questions do feel free to reach out.

   
