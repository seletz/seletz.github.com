---
layout:     post
title:      "Hot-Reloading game code in C"
date:       2024-01-08
comments:   true
tags:       meta programming raylib
---

## Problem 

When doing graphics programming for games or "recreational" programming, one
wants to have a fast feedback loop.

Additionally, when testing / debugging a game, we often need to reload the
**logic** of the game and test it with a given game state.  Often enough we
can't fully serialise and reload game state from/to disk.

Usually this is done by using a scripting language, which is often embedded in
games for speed reasons.

Using shared libraries and POSIX `dlopen(3)` and `dlsym(3)` we can archive the
same effect without using a scripting language in C.  I believe, that in fact
any language which compiles down to binary code and supports the C calling
convention could be used[^zig].

## The Goal

- Create a minimal example of hot-code-reloading using raylib.

- The goal is to be able to **hot-reload** the _game code_ shared object while the
  game is running, whenever we hit the `r` key.
  
- Document the solution.

## The Plan

I want to create some graphics window on the screen, and I want to hot-reload
changes in the graphics code.

To do this, I'll use [raylib](https://www.raylib.com) which has a very simple API
and is easy to use, and has plenty of examples.

I'll create a main executable which will:

- Load a shared library containing the _game code_ and find and bind symbols
  form that shared library to global function pointes
   
- Initialise a graphics window

- Initialise global _game state_ using a function from the _game code_

- Start a game loop, doing:

  - Check if the user hits the `r` key and if so, reload and rebind the _game code_ shared
    library
    
  - Update the _game state_
  
  - Update (draw) the next frame
  
  
## Implementation

This is what it looks like in the end -- nothing spectacular:

![](/assets/images/raylib-hotreload.png)


### Initialization

Initialization is pretty straight forward:

```c
int main(void) {
    const int screenWidth = 800;
    const int screenHeight = 450;

    if (do_reload() != 0) {
        return 1;
    };

    state = init_gamestate();

    SetConfigFlags(FLAG_WINDOW_RESIZABLE);
    InitWindow(screenWidth, screenHeight, "Hot Reloading example.");

    SetTargetFPS(60);
    
    // Game Loop Here

    CloseWindow();

    free_gamestate(state);
    state = NULL;

    return 0;
    
}
```

- The `init_gamestate()` is a function of the _game code_ shared library which allocates and initializes global
  game state.  The `free_gamestate()` frees that memory again.
  
- The `do_reload()` function handles the loading of the shared library and the
  binding of the function pointers, more on that later.
  
- The rest of the code is how a window of a given size with flags and title is opened and closed in
  raylib.
  
### The game loop

The is pretty easy, too:

```c
    while (!WindowShouldClose())
    {

        if (IsKeyPressed(KEY_R))
            do_reload();

        update_gamestate(state);

        update_frame(state);
    }
```

- The game loop executes until we get a signal that we should close the window

- for every frame we:

  - We check if key `r` is hit and call our `do_reload()` to reload all game code

  - We update the game state by calling the `update_gamestate()` function
  
  - We call the `update_frame()` function to redraw the frame.
  
  
The `update_gametate()`, `update_frame()`, `init_gamestate()` and `free_gamestate()`
functions are globally defined likes so:

```c

#incude "game.h"

...

init_gamestate_t init_gamestate = NULL;
free_gamestate_t free_gamestate = NULL;
update_gamestate_t update_gamestate = NULL;
update_frame_t update_frame = NULL;
  
...

```

These type aliases and the _game state_ are defined in `game.h` as function pointers:

```c
#ifndef _GAME_H
#define _GAME_H 1

#include "raylib.h"

typedef struct {
    unsigned long long tick;
    Camera3D camera;
    Vector3 cubePosition;
} gamestate_t;


typedef gamestate_t * (*init_gamestate_t)();
typedef int (*free_gamestate_t)(gamestate_t *state);
typedef int (*update_gamestate_t)(gamestate_t *state);
typedef int (*update_frame_t)(gamestate_t *state);

#endif
```
  
### Loading and Reloading 

To load a shared library into the current process memory, the `dlopen(3)`
call is used.  That call takes a *path* to a shared object and returns a
*handle* to the shared object.  We than can use `dlsym(3)` to search that
library for symbols to obtain their *address*.

We use this to find the _game code_ function addresses and then set the
global function pointers to these addresses.

All this is done in `reload.c`.

To have it easier for the caller, a `symtab_t` type alias is defined in `reload.h`:

```c
#ifndef RELOAD_H_
#define RELOAD_H_

typedef struct {
  char *symbol;  // name of the symbol
  void **ptr;    // address to be updated
} symtab_t;

/**
 * Load given dll and rebind symbols found in supplied
 * symbol table.
 */
int reload(char *path, symtab_t *symtab);

#endif // RELOAD_H_
```

Using this, the `do_reload()` function of the main executable calls the `reload()`
function with a "symbol table" containing the global function pointers and
the name of the shared object to load:

```c
int do_reload() {
    static symtab_t symtab[] = {
        {"init_gamestate", (void **)&init_gamestate},
        {"free_gamestate", (void **)&free_gamestate},
        {"update_gamestate", (void **)&update_gamestate},
        {"update_frame", (void **)&update_frame},
        {0}
    };
    return reload("libgamecode.so", symtab);
}
```

The `reload()` function is implemented like this:

```c
int reload(char *dll_name, symtab_t *symtab) {
    static void *game_dll = NULL;

    fprintf(stderr, "DLL_NAME: %s\n", dll_name);

    if (game_dll) {
        fprintf(stderr, "CLOSING DLL Handle %p\n", game_dll);
        dlclose(game_dll);
        game_dll = NULL;
    }

    game_dll = dlopen(dll_name, RTLD_LAZY | RTLD_LOCAL);
    if (!game_dll) {
        fprintf(stderr, "ERROR: Could not open shared library %s: %s\n",
                dll_name, dlerror());

        return 1;
    }
    fprintf(stderr, "NEW DLL Handle: %p\n", game_dll);

    while (symtab->symbol) {
        void *sym = dlsym(game_dll, symtab->symbol);
        if (!sym) {
            fprintf(stderr, "ERROR: Caould not open symbol: %s: %s\n",
                    symtab->symbol, dlerror());

            return 2;
        }

        fprintf(stderr, "RELOAD: found symbol %s -> %p\n", symtab->symbol, symtab->ptr);

        *symtab->ptr = sym;
        symtab++;
    }

    return 0;
}
```


### Graphics and Game Code

The `update_gamestate()` and `update_game()` functions are defined in a separate
source file which will be built into a shared object.  It's basically the
`core_3d_camera.c` example from raylib:

```c
#include <stdio.h>
#include <stdlib.h>
#include <assert.h>

#include "debug.h"

#include "raylib.h"
#include "game.h"

gamestate_t * init_gamestate() {
    gamestate_t *state = malloc(sizeof(gamestate_t));

    state->tick = 0LL;
    state->camera.position = (Vector3){ 0.0f, 10.0f, 10.0f };  // Camera position
    state->camera.target = (Vector3){ 0.0f, 0.0f, 0.0f };      // Camera looking at point
    state->camera.up = (Vector3){ 0.0f, 1.0f, 0.0f };          // Camera up vector (rotation towards target)
    state->camera.fovy = 45.0f;                                // Camera field-of-view Y
    state->camera.projection = CAMERA_PERSPECTIVE;             // Camera mode type

    state->cubePosition = (Vector3){ 0.0f, 0.0f, 0.0f };

    return state;
}

int free_gamestate(gamestate_t *state) {
    assert(state);
    free(state);
    return 0;
}

int update_gamestate(gamestate_t *state) {
    assert(state);

    state->tick += 1;

    return 0;
}


int update_frame(gamestate_t *state) {
    assert(state);

    BeginDrawing();

        ClearBackground(RAYWHITE);

        BeginMode3D(state->camera);

            DrawCube(state->cubePosition, 2.0f, 2.0f, 2.0f, GREEN);
            DrawCubeWires(state->cubePosition, 2.0f, 2.0f, 2.0f, MAROON);

            DrawGrid(100, 1.0f);

        EndMode3D();

        DrawText("Welcome to the third dimension!", 10, 40, 20, DARKGRAY);
        char buf[80];
        snprintf(buf, 80, "TICK %lld", state->tick);
        DrawText(buf, 10, 60, 20, DARKGRAY);

        DrawFPS(10, 10);

    EndDrawing();

    return 0;
}
```

## Conclusion

I was very surprised to see how *easy* and straight-forward hot-reloading can be.  Of course,
I'm aware that this implementation is *not* thread-safe.  I have been using _python_ and other
programming languages, where hot code reloading is supported.  I see that C has an advantage here,
because we need to build our own data structures, and have no support for OOP, polymorphism etc,
which complicates things greatly (how is reloading a class going to affect objects of that class?).

I'm also very pleased how easy it is to get things done in [raylib](https://www.raylib.com).

## Links

- The [code is
  available](https://github.com/seletz/raylib-hot-code-reload-c-example) on
  GitHub.

- I got inspired to try this by this [blog
post](https://medium.com/@TheElkantor/how-to-add-hot-reload-to-your-raylib-proj-in-c-698caa33eb74),
which does something similar but for Windows.

- I also found [this video](https://youtu.be/Y57ruDOwH1g) by
[tsoding](https://www.youtube.com/tsoding) helpful.

---

[^zig]: Maybe I'll explore that idea in a future post, using [zig](https://ziglang.org) or [rust](https://www.rust-lang.org).

