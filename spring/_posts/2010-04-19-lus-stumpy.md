---
title: Lua unit script of a stumpy
author: Tobi Vollebregt
layout: post
---

* This list will contain the toc (it doesn't matter what you write here)
{:toc}

# Lua unit script: Stumpy

## Introduction

This tutorial will guide you through:

* setting up a mutator of an existing game
* enabling Lua unit scripting in this mutator
* writing a first unit script in Lua

It is expected neither Lua nor Spring are completely new for you: this isn't a Lua or Spring tutorial.


## Set up a mutator

Create a new directory *stumpy.sdd* next to the other Spring games you have installed. Create a file *modinfo.lua* in this new directory and put the following inside:

{% highlight lua %}
return {
	name = 'Lua unit script: Stumpy example',
	modType = '1',
	depend = {
		'Balanced Annihilation V7.12'
	},
}
{% endhighlight %}

If you now press 'Reload maps/mods' in your lobby client the mutator should show up. To test it you can start a game with it: it should be as if you start Balanced Annihilation itself, apart from the changed name.


## Enable Lua unit scripts

Create a directory *stumpy.sdd/LuaRules/Gadgets* and put a file *unit_script.lua* inside, containing:

{% highlight lua %}
-- Enable Lua unit scripts by including the gadget from springcontent.sdz
return include('LuaGadgets/Gadgets/unit_script.lua')
{% endhighlight %}

Continue by creating the directory *stumpy.sdd/scripts* and put an empty file *armstump.lua* inside. Now launch Spring and look at the infolog. It should contain something like this:

	[      0] Using mod Lua unit script: Stumpy example
	[      0] Using mod archive stumpy.sdd
	...
	[      0] Loading gadget: Lua unit script framework  <unit_script.lua>
	[      0]   Loading unit script: scripts/armstump.lua
	[      0] Loaded gadget:  Lua unit script framework  <unit_script.lua>

Try to attack something with the stumpy. It will not work, this empty Lua script takes precedence over BA's *armstump.bos*.

You may want to keep both Spring (set it to run in a window) and your editor open when you move to the next section: Lua unit scripts can be reloaded without restarting Spring. They are reloaded when the entire LuaRules state is reloaded, like this:

	/cheat
	/luarules reload

## The Stumpy

### Hello, world

Keep *armstump.lua* open, we'll put something in it. Let's start by examining the pieces of the model. This isn't needed in any way but it serves as a nice 'Hello world' example and it may even be a useful snippet if you can't remember your piece names ;-)

{% highlight lua %}
for k, v in pairs(Spring.GetUnitPieceMap(unitID)) do
	Spring.Echo('piece ' .. k)
end
{% endhighlight %}

Now reload LuaRules (remember: **/luarules reload**), and the infolog should contain a few lines like this:

	[   6941] piece flare
	[   6941] piece barrel
	[   6941] piece turret
	[   6941] piece base

This are the pieces of which the Stumpy model exists. Clear the file and let's start on something more useful.

### Arming the Stumpy

This example makes the weapon of the stumpy fully functional. Here is the complete code:

{% highlight lua %}
local base, flare, turret, barrel = piece('base', 'flare', 'turret', 'barrel')
local SIG_AIM = 1

function script.AimWeapon1(heading, pitch)
	Signal(SIG_AIM)
	SetSignalMask(SIG_AIM)
	Turn(turret, y_axis, heading, 0.5)
	Turn(barrel, x_axis, -pitch, 0.25)
	WaitForTurn(turret, y_axis)
	WaitForTurn(barrel, x_axis)
	return true
end

function script.FireWeapon1()
	Show(flare)
	Move(barrel, z_axis, -2.4)
	Sleep(150)
	Hide(flare)
	Move(barrel, z_axis, 0, 3)
end

function script.AimFromWeapon1()
	return turret
end

function script.QueryWeapon1()
	return flare
end
{% endhighlight %}

Let's examine some patterns we see here in detail in the next sections.

### Piece definitions

{% highlight lua %}
local base, flare, turret, barrel = piece('base', 'flare', 'turret', 'barrel')
{% endhighlight %}

Here piece numbers are retrieved (using the **piece** function) and assigned to the variables base, flare, turret and barrel.

### Aiming

{% highlight lua %}
local SIG_AIM = 1

function script.AimWeapon1(heading, pitch)
	Signal(SIG_AIM)          -- Kill all aim threads for this unit
	SetSignalMask(SIG_AIM)   -- Mark this thread as an aim thread,
	                         -- so a subsequent Signal(SIG_AIM) call kills it.

	Turn(turret, y_axis, heading, 0.5)
	Turn(barrel, x_axis, -pitch, 0.25)
	WaitForTurn(turret, y_axis)
	WaitForTurn(barrel, x_axis)
	return true
end
{% endhighlight %}

This is a common pattern present in (nearly) all unit scripts you'll read and write. **AimWeapon** is the call-in that is started by Spring when an enemy is nearby and the unit needs to aim at this enemy. Spring pre-calculates the heading and pitch and passes these to this function. Spring calls this function very often though, which is why **Signal** and **SetSignalMask** are used to ensure only one aim thread (per unit) is alive at any one time.

The next statements define the actual animation: the turret piece is turned towards the target at a horizontal speed of 0.5 rad/sec and a vertical speed of 0.25 rad/sec. The thread is then suspended until the animation has finished before it shouts 'FIRE!' to Spring by means of the statement **return true**.

### Firing

{% highlight lua %}
function script.FireWeapon1()
	Show(flare)
	Move(barrel, z_axis, -2.4)
	Sleep(150)
	Hide(flare)
	Move(barrel, z_axis, 0, 3)
end
{% endhighlight %}

Once the weapon is fired (even if **AimWeapon** returns true it's possible the unit does not fire because e.g. friendly units or trees block its line of fire) Spring calls **FireWeapon** for that weapon. Typically, this displays a flare and plays a recoil animation. That is exactly what is done in this example. First, the flare piece is shown and the barrel is positioned -2.4 elmos backward. Then the call to **Sleep** suspends the thread for 150 milliseconds before the flare piece is hidden again. Immediately the barrel starts moving back to it's initial position at a speed of 3 elmos/sec.

### What's left?

{% highlight lua %}
function script.AimFromWeapon1()
	return turret
end

function script.QueryWeapon1()
	return flare
end
{% endhighlight %}

Last but not least these two call-ins define which pieces Spring shall use as 'origin' in all the aiming calculations. If you are new to unit scripting, just follow the pattern presented here: **AimFromWeapon** should return the turret piece and **QueryWeapon** should return a piece at the end of the barrel. Later on you may use these to implement weapons that fire from multiple barrels on the same turret.


## Conclusion

In this tutorial I showed how to script a simple unit using a Lua unit script. You've (hopefully) learned something about the following call-ins:

* **AimWeapon**: aims the unit's weapon at the target selected by Spring
* **FireWeapon**: called when the weapon fires
* **AimFromWeapon**, **QueryWeapon**: defines the origin pieces for the aiming calculations

And the following call-outs:

* **Show**: make a piece visible
* **Hide**: make a piece invisible
* **Signal**: kill all threads matching a certain 'signal mask'
* **SetSignalMask**: set the 'signal mask' of the current thread
* **Turn**: rotate a piece around an axis with a certain speed
* **Move**: move a piece along an axis with a certain speed
* **WaitFor{Turn,Move}**: suspend the thread until an animation is finished
* **Sleep**: suspend the thread for some milliseconds

And got some idea now how to put basic scripts on the units in your branch new game. Good luck!
