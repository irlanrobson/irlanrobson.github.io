---
layout: post
title: Keyboard Input for Fixed-Time Step Games
tags:
- Game Programming
- Input
---

Suppose we are using a fixed-time step to update the game and we're at the begin of a new frame. Then, the keyboard input events that were fired are requested and handled via callbacks or similar, making a character to jump or shoot in an enemy. What is the problem with that?

The problem is that inputs handled directly (e.g. callbacks) without some kind of pre-processing step approaches the game simulation into an event-based application. We all know that games aren't event based applications even though there are game events to be handled in a game. That's particularly important when we're using a fixed time step simulation.

In this post I'll show you one approach that handles keyboard input on Windows assuming that the game is been updated using a fixed-time step. However, the concept extends to basically any other input device. In order to not go futher into the subject (i.e. describing high-level game input logging/mapping/contexts) I'll only describe a solution for the previous problem and after that provide an Windows implementation which uses the concept of buffered inputs and multithreading.

## Acknowledgment

Thanks to 
[L. Spiro](http://www.gamedev.net/user/51611-l-spiro/) for sharing his way of handling inputs at 
[GameDev.net](http://www.gamedev.net/page/index.html), and for the excelent posts at 
[L. Spiro Engine](http://lspiroengine.com/)'s website about general engine architecture. His discussions inspired me to write this post.

## Introduction

Updating the game simulation each frame by the elapsed frame time since the last frame update (that is, a variable time step), means that the game behaviour varies with time. That's the main reason why currently game programmers tend to update their games each frame by fixed time intervals. It is well know that this is efficiently achieved by the famouse fixed-time step. However, its implementation details are off-topic. I'm assuming that the reader have implemented fixed-time step games or understands how they work. If not, I suggest him to read 
[this post](http://www.gamedev.net/page/index.html) which explains various methods of stepping a simulation and ultimately chooses the fixed-time-step approach used here as the best for good reasons. Thus, a single fixed-time step update eventually goes by the name game logical update.

A single game logical update updates the current game logical time by a fixed time interval which is usually 16.6 ms (60 Hz) or 33.3 ms (30 Hz), and it updates our game by 
n times each frame depending of the current real frame time. The game loop of a fixed-time step game is written below to freshen our minds.

{% highlight cpp %}

// Game.h

static const u64 s_dt = 33333; 

// Game.cpp

void Game::Run() 
{
     // Input handling and processing go below.
     m_window.PoolEvents(&m_keyboard);
     m_renderTime.Update();
     u64 curTime = m_renderTime.CurTime();
     while (curTime - m_logicTime.CurTime() > s_dt) 
     {    
             m_logicTime.UpdateBy(s_dt);
             UpdateGameState();
     }
} 

{% endhighlight %}

In the previous pseudo-code, m_renderTime.Update() updates the application time by the real elapsed time since the last call, and m_logicTime.UpdateBy(s_dt) updates the game logic time by s_dt microseconds.

If one presses a button in the beginning of the current frame, set this button state as pressed, and suddenly release the button during a game logical update, then the button will be seen by the next updates (if there are) as if it got pressed during the entire frame. This is not a problem when the elapsed frame time is small because it'll jump to the next frame quickly, but if the current frame time is considerably larger than the time step, then that can turn in something undesirable to some players. A better approach that pottentially solves our problem is to pool input events right before a game logic update, as written bellow.

{% highlight cpp %}

void Game::Run() 
{
     m_renderTime.Update();
     u64 curTime = m_renderTime.CurTime();
     while (curTime - m_logicTime.CurTime() > s_dt) 
     {    
             m_logicTime.UpdateBy(s_dt);
             m_window.PoolEvents(&m_keyboard);             
             UpdateGameState();
     }
}

{% endhighlight %}

However, if we're concerned with input timing for game logic, then we need a way of retrieving those inputs precisely, which means time-stamping a button when it gets pressed or released, so its duration can be correctly measured later, and more importantly, we can synchronize it with the game simulation. 

We can attempt the increase input latency by handling keyboard inputs in a separate thread and processing them before a game logic update (the UpdateGameState function previously), much similar to the consumer and producer model, in this case the producer being the window and the consumer the game. But how am I going to get the correct set of inputs once these things are available? Well, since the game logic time depends on the current real time, we must 
**consume on each game logical update only the input events that occurred up to the current game logic time** in order to keep inputs synchronized with the game simulation. The following example scenario will illustrate this idea.

### Frame Update

| Fixed-Time Step | Render Time | Game Time | Total Game Logical Updates
|---------|-------------|---------------|-------------
| 100 ms | 1000 ms | 0 ms | 1000 / 100 = 10|

### Input Buffer

| Button | Event Type | Time-Stamp 
|---------|-------------|---------------
| X | Down | 700 ms | 
| X | Up | 850 ms |
| Y | Down | 860 ms |
| Y | Up | 900 ms |
| Z | Down | 1050 ms |

### Input Buffer Pre-Processing

Now a couple of comments on the data we've seen in the table.

1st update. Eats 100 ms of input: There are no inputs up to there to be consumed, then go to the next logical update.

1-7th update. There are no inputs up to the current logical game time (600 ms). Therefore go to the next logical update.

8th update. The game time is: 800 ms. Then the time-stamped X-down event must be consumed. The current duration of the X button is the current game time subtracted by the time stamp, that is, 800 ms - 700 ms = 100 ms. Now, the game can check if a button is being held for a certain amount of time, which is an usable information. Momentarily, we know that a mechanism could be fired here because is the first time the user presses the X button (in the example, of course, because there was no X-down before). Another thing we could do on this example would be mapping the X button to a game-engine understandable input key, logging it into the input system, and then remapping it into the game.

9th update. Game time = 900 ms. X-up, and Y-down along with its time-stamps can be consumed. The X button was released, then its total duration since it was pressed is the current game time subtracted by its first tap time-stamp, that is, 900 ms - 700 ms = 300 ms. You may want to log this change somewhere in the game-side. Y was pressed, then we repeat for it the same thing we did to X in the last update.

10th update. The current game time is 1000 ms. We repeat the same thing we did to X in the last update for Y.

And of course, since the input gathering runs asynchronously (separated from input pre-processing), a game update can take too long to be executed and therefore the Z-Down event at 1050 ms will be processed in a next frame.

## Implementation

### The Time Class

At this point you should already know how the computer time works and how to create an appropriate timer class. The timer class stores microseconds as the standard time units in order to avoid numerical drifts. Small intervals can be converted to seconds or miliseconds and stored in doubles or floats but they're not accumulated in our timer class.

{% highlight cpp %}

// Time.h

#pragma once

class Time 
{
public :
	Time();

	void Update();
	void UpdateBy(u64 ammount);

	u64 CurTime() const { return m_curTime; }
	
	void SetFrequency(u64 freq)
	{
		m_frequency = freq;
	}

	void Sync(const Time& other) 
	{
		m_lastRealTime = other.m_lastRealTime;
	}

	static u64 RealTime();
private :
	u64 m_frequency;
	u64 m_curTime;
	u64 m_lastTime;
	u64 m_lastRealTime;
	u64 m_curMicros;
};

{% endhighlight %}

{% highlight cpp %}

// Time.cpp

#include "Time.h"
#include <Windows.h>

Time::Time() : 
m_frequency(0), 
m_curTime(0), 
m_lastTime(0), 
m_lastRealTime(0),
m_curMicros(0) 
{
	QueryPerformanceFrequency((LARGE_INTEGER*)&m_frequency));
	m_lastRealTime = RealTime();
}

u64 Time::RealTime() 
{
	u64 count = 0;
	QueryPerformanceCounter((LARGE_INTEGER*)&count));
	return count;
}

void Time::Update() 
{
	u64 t = RealTime();
	u64 dt = t - m_lastRealTime;
	m_lastRealTime = t;
	UpdateBy(dt);
}

void Time::UpdateBy(u64 dt) 
{
	m_lastTime = m_curTime;
	m_curTime += dt;
	u64 lastMicros = m_curMicros;
	m_curMicros = m_curTime * 1000000 / m_frequency;
}

{% endhighlight %}

### The Key Buffer

If you're running Windows, you probably know that the pre-processed input events are pooled automatically by the O.S. into the Win32 API message queue. It's mandatory to keep the event pooling in the same thread the window was created, but is not to keep the game running on another. In order to separate input processing from the game logic, we can let the message queue running on the main thread while the game simulation and rendering is on another. Here, for simplicity, we'll assume the rendering runs on the game-thread, so we won't need syncronization between game and rendering.

Below there is an window procedure just to let us remember how does look like processing keyboard events using the Win32 API.

{% highlight cpp %}

// Win32Window.cpp

LRESULT CALLBACK Win32Window::WindowProc(HWND handle, UINT msg, WPARAM wParam, LPARAM lParam) 
{ 

switch (msg) 
{
	case WM_KEYDOWN :
	{
		BufferKeyDown(Translate(wParam), Time::RealTime());
		return 0;
	}
	case WM_KEYUP :
	{
		BufferKeyUp(Translate(wParam), Time::RealTime());
		return 0;
	}
	// ...
};

} 

{% endhighlight %}

Because the SRP (Single Responsability Principle), we'll create a thread-safe keyboard buffer class whose unique responsability is store keyboard events that will be consumed later on the game thread. Of course we'll must give to the window class above an instance of it.

{% highlight cpp %}

// KeyBuffer.h

enum Keys
{
	// Enumerate the keyboard buttons here.
	MaxKeys = 300
};

enum ButtonState
{
	Released,
	Pressed
};

struct ButtonEvent
{
	ButtonState id;
	u64 time;
};
 
class KeyBuffer 
{
public :
	enum { MaxEvents = 256 };	
	
	void Press(int button);
	void Release(int button);
	void Consume(Array<ButtonEvent, MaxEvents>* events, u64 maxTime);
private :
	Array<ButtonEvent, MaxEvents> m_events[MaxKeys]; //This is a fixed-capacity array of capacity equals to MaxEvents. 	
	Time m_time;
	CriticalSection m_mutex;	
}; 

{% endhighlight %}

{% highlight cpp %}

// KeyBuffer.cpp

void KeyBuffer::Press(int button) 
{ 
	ScopeLock lock(m_mutex); 
	m_time.Update(); 
	ButtonEvent e; 
	e.id = Pressed; 
	e.time = m_time.CurTime(); 
	m_events[button].Push(e); 
} 

void KeyBuffer::Release(int button) 
{ 
	ScopeLock lock(m_mutex); 
	m_time.Update(); 
	ButtonEvent e; 
	e.id = Released; 
	e.time = m_time.CurTime(); 
	m_events[button].Push(e); 
}

{% endhighlight %}

The keyboard buffer class holds an array of events for each possible keyboard key and now can be used by the Win32Window class to store time-stamped events coming from the native event queue. The window procedure should look like below after the changes that have been made.

{% highlight cpp %}

// Win32Window.h

LRESULT CALLBACK Win32Window::WindowProc(HWND handle, UINT msg, WPARAM wParam, LPARAM lParam) 
{ 

switch (msg) 
{
	case WM_KEYDOWN :
	{
		m_keyBuffer.Press(Translate(wParam));
		return 0;
	}
	case WM_KEYUP :
	{
		m_keyBuffer.Release(Translate(wParam));
		return 0;
	}
	// ...
}; 

{% endhighlight %}

### Time Synchronization

Importantly, in order for the game thread to request inputs from the keyboard buffer up to some specific time, its current timer must be synchronized with the window thread's so its time-stamps are relative to the same timer and the set of inputs can be read correctly. This is the key concept for all our asynchronous scheme work correctly! Hence, all it needs to be done is to have a simple function in the time class that will internally copy one timer's last real time into another's effectively sincronizing both timers.

{% highlight cpp %}

void Time::Sync(const Time& other) 
{
	m_lastRealTime = other.m_lastRealTime;
}

{% endhighlight %}

Now it's possible to synchronize the game timer with the window's. The following shows how this is done assumming it is done before the game runs.

{% highlight cpp %}

void Game:Game(Engine& engine) 
{
	KeyBuffer& buffer = engine.GetKeyBuffer();
	m_renderTime.Sync(buffer.m_time);
	m_logicTime.SetFrequency(1000000);
} 

{% endhighlight %}

Notice that the game logic timer's frequency must be one microsecond in seconds since it gets updated by a fixed time interval, and therefore we don't want the interval to be divided by the real frequency. All that needs to be done is set the game logic timer frequency to 1000000 (1 us = 1 / 1000000 s) as written above. Now that time-stamped events are being buffered correctly, the main thread is listening for input events on the background and shouldn't interfere the game directly. But we still do need to consume and process events in order to actually use them in the game.

{% highlight cpp %}

void Win32Window::PoolEvents() 
{
	MSG msg;
	while (m_IsOpen)
	{
		WaitMessage(); //This will avoid wasting cycles in this thread when there are no messages to be processed.
		while (PeekMessage(&msg, m_handle, 0, 0, PM_REMOVE))
		{
			DispatchMessage(&msg);
		};
	};
}

{% endhighlight %}

### The Keyboard

The responsibility of the keyboard buffer is to buffer keyboard presses and releases at low-level so it should be possible to read them in the game-thread. In the game-thread before a game tick we will only consume the input events up to the current game logical time and then generate another set of input events that will be consumed in a linear fashion. That is needed since events that can't be consumed in the current tick need to be kept in this temporary buffer so it can be consumed later in a next update. (Ideally, this temporary buffer should use n stack allocator as it only contains temporary and POD data.)

{% highlight cpp %}

// KeyBuffer.cpp

void KeyBuffer::Consume(Array<ButtonEvent, MaxEvents>* outEvents, u64 maxTime) 
{
	ScopeLock lock(m_mutex);

	for (uint i = 0; i < MaxKeys; ++i)
	{
		Array<ButtonEvent, MaxEvents>& events = m_events[i];

		for (uint j = 0; j < events.Count();)
		{
			ButtonEvent e = events[j];
			if (e.time <= maxTime)
			{
				// Peel an event from the buffer.
				outEvents[i].Push(e);
				j = events.Remove(j);
			}
			else
			{
				++j;
			}
		}
	}
}

{% endhighlight %}

Now we can use these sets of events to update the keyboard which contains keyboard key states and their durations in order to be acessed by the game. This class should be responsible of answering questions such as 

* "For how long this key was pressed?" or 
* "What is the current duration of this key?"

{% highlight cpp %}

// Keyboard.h

#pragma once

#include "KeyBuffer.h"

class Keyboard 
{
public :
	Keyboard();

	bool KeyIsPressed(int button) const
	{
		return m_curState[button].pressed;
	}
        
	u64 KeyDuration(int button) const
	{
		return m_curState[button].duration;
	}
private :
	struct KeyState
	{
		// The key is pressed.
		bool pressed;
		// The time the key was pressed. This is needed to calculate its duration.
		u64 timePressed;
		// This should be logged but is here for simplicity
		u64 duration;
	};

	KeyState m_curState[KeyBuffer::MaxEvents];
	KeyState m_lastState[KeyBuffer::MaxEvents];
}; 

{% endhighlight %}

Now the keyboard is able to be used as our final keyboard on the game, and we still do need to fed the data coming from the temporary keyboard buffer into it. The following function shows how that is done.

{% highlight cpp %}

// KeyBuffer.cpp

void UpdateKeyState(Array<ButtonEvent, MaxEvents>* buffer, Keyboard& keyboard, u64 maxTime) 
{
	for (uint i = 0; i < MaxKeys; ++i)
	{
		Array<ButtonEvent, MaxEvents>& events = buffer[i];
		Keyboard::KeyState lastState = keyboard.m_lastState[i];
		Keyboard::KeyState curState = keyboard.m_curState[i];
		
		for (uint j = 0; j < events.Count(); ++j)
		{
			ButtonEvent e = events[j];

			if (e.state == Pressed)
			{
				if (lastState.pressed)
				{
					// The key is being held.
				}
				else
				{
					// Compute the time that the key was pressed.
					curState.pressed = true;
					curState.timePressed = e.time;
				}
			}
			else
			{
				// Compute the total duration of the key event.
				curState.pressed = false;
				curState.duration = e.time - curState.timePressed;
			}

			lastState = curState;
		}

		if (curState.pressed)
		{
			// The key is being held. Update its duration.
			curState.duration = maxTime - curState.timePressed;
		}
		
		lastState = curState;		
	}
} 

{% endhighlight %}

Now that we have readable inputs, we can use them in the game logical update.

### The Final Game Loop

I'm aware how much a game must be agnostic about input management. But we won't get into architecture details here, and for simplicity we'll give the game an instance of the keyboard class.

{% highlight cpp %}

// Game.h

class Game 
{
public :
	Game(Engine& engine);
	void Run();
protected :
	void UpdateGameState();

	Engine& m_engine;
	StackAllocator m_stackAlloc;
	Keyboard m_keyboard;
}; 

{% endhighlight %}

With that the final game-thread loop with can be written.

{% highlight cpp %}

// Game.cpp

void Game::Run() 
{
	KeyBuffer& buffer = m_engine.GetWindow().GetKeyBuffer();

	m_renderTime.Update();
	u64 curTime = m_renderTime.CurTime();
	while (curTime - m_logicTime.CurTime() > s_dt)
	{
		m_logicTime.UpdateBy(s_dt);
		u64 maxTime = m_logicTime.CurTime();

		u8* marker = m_stackAlloc.Marker();		
		Array<ButtonEvent, MaxEvents>* events = m_stackAlloc.Alloc< Array<ButtonEvent, MaxEvents> >(MaxKeys);
		buffer.Consume(events, maxTime);		
		UpdateKeyState(events, m_keyboard, maxTime);
		m_stackAlloc.Unwind(marker);

		// Now we can use the keyboard at any time in a game-state.
		UpdateGameState();
	}
} 

{% endhighlight %}

## Conclusion

That's all. What we did in this article was create a small input system that is synchronized with the game simulation and it runs asynchronously. After the game has all the input information then it can start mapping and logging it. There are open doors for optimization.

Note that this way of managing inputs can add some complexity to your current code base. For small testbeds or demos or simple applications, using the old polling it still can be an advantage in favor of simplicity. Thanks for reading.
