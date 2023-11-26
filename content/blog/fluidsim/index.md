---
title: Fluid Simulation
description: Another implementation of Joe Stam's famous algorithm
date: 2023-11-24
tags:
  - golang
  - wasm
layout: layouts/post.njk
---

{% image "./fluidsim.png", "gray scale fluid simulation" %}

Showing off a Golang implementation of Joe Stam's fluid simulation algorithm.

### Joe Stam

I've had an interest in simulating fluid dynamics ever since my boy and I played "Where's My Water" on the iPad back in the meaty part of the last decade.
It's a fun game, totally built around the fluids.

Fast-forward to June of this year.
I'm sat in a lovely public [library](https://www.openstreetmap.org/#map=18/60.23118/24.93314) of suburban Helsinki, digging up ideas for a gamey sort of project.
The now teenage boy, is still sleeping in our nearby room :)

And up pops, [Real-Time Fluid Dynamics for Games](http://graphics.cs.cmu.edu/nsp/course/15-464/Fall09/papers/StamFluidforGames.pdf).
Looks workable!  I've seen Stam's name mentioned quite a lot in relation to fluid simulation and the paper has a more or less complete implementation.

Here is the `diffuse` function from the paper:

```c
void diffuse ( int N, int b, float * x, float * x0, float diff, float dt )
{
  int i, j, k;
  float a=dt*diff*N*N;

  for ( k=0 ; k<20 ; k++ ) {
    for ( i=1 ; i<=N ; i++ ) {
      for ( j=1 ; j<=N ; j++ ) {
        x[IX(i,j)] = (x0[IX(i,j)] + a*(x[IX(i-1,j)]+x[IX(i+1,j)]+
                    x[IX(i,j-1)]+x[IX(i,j+1)]))/(1+4*a);
      }
    }
    set_bnd ( N, b, x );
  }
}
```

Translating to Golang was kind of fun, got my head into the algorithm, and only resulted in one bug!

I did scratch my head quite a bit around the concept of an incompressible fluid while the simulation clearly models a varying density.
Turns out, the density being modeled is that of a dye in an incompressible fluid such as water, aha!

Also a significant exercise left for the reader of the paper is how one is meant to interface with the sim code from the larger application.
It's a very "C" kind of a thing, by way of shared data structures that represent different aspects of the simulation depending on which step is being executed.

Still, all very approachable and in short order I had density diffusion apparently working and built from there.

## Ebiten

Or [Ebitengine](https://ebitengine.org/).  I love playing around with simple 2D game engines and this one's a gem!

Also, "games" built with it can be compiled to WASM (foreshadowing). 

## The Golang

As per usual, the code is out there on [GitHub](https://github.com/clarktrimble/stam).


Here is the Golang rendition of the `diffuse` function from above:

```go
// stam.go
func (fl *Fluid) diffuse(bnd int, diff float64, xx, x0 ifc.Gridder) {

  a := fl.dt * diff * float64(fl.size*fl.size)

  for k := 0; k < 20; k++ {
    for i := 1; i <= fl.size; i++ {
      for j := 1; j <= fl.size; j++ {

        num := x0.Get(i, j) + a*(xx.Get(i-1, j)+xx.Get(i+1, j)+xx.Get(i, j-1)+xx.Get(i, j+1))
        result := num / (1 + 4*a)

        xx.Set(i, j, result)
      }
    }
    fl.setBnd(bnd, xx)
  }
  return
}
```

Totally punted on unit testing this baby.
I'd just as totally do this on a rewrite, he says.

A significant feature of the Stam C code is swapping vectors (arrays) as the algorithm moves from step to step.

Here's how it turned out:

```go
// vektor.go
type Vektor struct {
  dim  int
  vals *[]float64
}

func (vk *Vektor) Swap(other ifc.Gridder) {

  ov, ok := other.(*Vektor)
  if !ok {
    panic(fmt.Sprintf("somehow asked to swap a non-vector: %#v", other))
  }
  if vk.dim != ov.dim {
    panic("will not swap vectors of differing dimentions")
  }

  tmp := vk.vals
  vk.vals = ov.vals
  ov.vals = tmp
}
```

Where `Vektor` implements `Gridder` allowing it to be injected into simulation code.
Not sure I'd go with this again, but was an interesting exercise :)

I did like the way simulation and game code factored relative to each other, as hinted at in `game.New`:

```go
// game.go
func New(gridSize, scale int) (game *Game, err error) {

  fnt, err := fonts.Font(fontSize, fontDpi)
  if err != nil {
    return
  }

  game = &Game{
    font:      fnt,
    help:      true,
    gridImage: ebiten.NewImage(gridSize, gridSize),
    gridSize:  gridSize,
    gridMid:   gridSize / 2,
    scale:     scale,
    fluid:     stam.NewFluid(gridSize, visc, diff, dt, factory),
  }
  return
}
```


## WASM

Coming back to this project the day after Thanksgiving, it's been super straightforward to publish to the web via WebAssembly.

### Compile

```bash
~/proj/stam$ env GOOS=js GOARCH=wasm go build -ldflags "-w -s" -o bin/fluidsim.wasm cmd/fluidsim/main.go
```

### Javascript

First, copy `bin/fluidsim.wasm` and `.../go/misc/wasm/wasm_exec.js` into _ba blog_.

Then sprinkle a little js boilerplate into an html page:

```html
<!DOCTYPE html>
<script src="/public/wasm_exec.js"></script>
<script>
const go = new Go();
WebAssembly.instantiateStreaming(fetch("/public/fluidsim.wasm"), go.importObject).then(result => {
    go.run(result.instance);
});
</script>
```

### Give it a click!

It's ........................ [fluidsim browser-edition](/pdf/main.html)

Kind of a cool toy, yeah?

### Caveats

Already lackluster performance took a hit with WASM.
I cranked the grid size down from 80 to 40 to keep things moving along.

Almost certainly won't work on a phone, as I've only handled mouse events.

## Postscript

It works! And it's another project in a passably finished state :)

The next time I'm summering in the Baltic maybe I'll take a crack at something along the lines of Sebastian Lague's [particle-based fluid simulation](https://github.com/SebLague/Fluid-Sim) which is a lot more like the one I recall from "Where's My Water" and super awesome.

Thanks for reading and I hope your holiday season is off to a great start.
