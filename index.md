# Agent Based Modelling for Cell Population Dynamics

To jump straight into our code, visit our org page [github.com/cyton-dynamics](https://github.com/cyton-dynamics)

## Overview
This is a framework for implementing agent based models of stochastic cell population dynamics. It is based on the ideas in [this review](https://pubmed.ncbi.nlm.nih.gov/30129201/). Cells have number of fates: division, death, division destiny, differentiation, etc. These fates are determined by competition between internal modules.

The framework provides abstractions for these modules and other important concepts but leaves implementation to the modeller. The framework provides the machinery to evolve a population cells in time and enables events generated by timers to alter cell fate.

The next sections cover the key concepts provided by the package.

Detailed documentation on the API is here [cyton-dynamics.github.io/cyton.toolkit/](https://cyton-dynamics.github.io/cyton.toolkit/)

## Cell Population
This is a collection of cells that is evolved through time by the model stepper. Cells can die, divide or can be otherwise inserted into the simulation. The population can instrumented with observers to gather measurements as the population evolves. A cell population is created by calling the `createPopulation` function and passing a factory method to create cells.

## Cells
Cells are primarily a collection of fate timers. In addition, they carry the cell creation time and the generation number. Cells can respond to events (see below). The basic cell class is a parameterised type, `AbstractCell{T}`, the type parameter, `T` can be used by the modeller, for example to carry a genotype, phenotype or treatment. By default, a new cell will be of type `GenericCell`.

## Fate Timers
Fate timers are where cell dynamics are implemented. There is abstract base class in the framework and modellers are required to implement their own timers. Timers are implemented by creating:

- a concrete class to hold the state of the timer
- a step function that updates the state of the timer at each time step. The stepper can return an event triggered by the timer.
- an inherit function that creates new timers from the original timer for daughter cells on division

Cells have observers that listen for events triggered by other timers in the cell which can in turn modify the state of other timers.

## Cell Events.
Events are the mechanism by which timers trigger change in fate or behaviour. The abstract base class in `CellEvent`. The framework provides two concrete classes:

- `Death` which causes the cell to be removed from the population and hence the simulation. 
- `Division` which creates a new cell and calls `inherit` on all the timers.

Events can be used to implement interaction between timers, for example a timer can silence another timer. Events can carry data. 

## Building models
At a minimum, the modeller needs to provide a cell factory, a function that takes the current time and returns a cell. The cell factory will create timers and add them to the cell. For example:
```
# This function takes the birth time of the cell and parameters for this run
function myCellFactory(birth::Time, myParameters)
 cell = Cell(birth) # Create a new cell

 myTimer = MyTimer(parms) # Create timers
 addTimer(cell, myTimer) # add the timers to the cell
 ...
 return cell
end

# This will typically be a structure carrying the parameters for this run
myParameters = MyParameter(this, that, the, other)

# We create an anonymous function to close the parameters
population = createPopulation(nCells, (birth)->myCellFactory(birth, myParameters))
```

The behaviour of the timers is defined by a step function. The timer type needs to be precisely specified in the function signature because this how Julia finds the correct stepper.
```
function step(timer::MyTimer, time::Time, ??t::Duration)::Union{CellEvent, Nothing}
  # update the state of the timer
  if <some condition>
    return MyEvent() # Trigger an event
  else
    return nothing # Or just return nothing 
  end
end
```

If cells will divide, the modeller must implement a mechanism by which timers are inherited by the daughter cells. The method is called twice on each `FateTimer` to create timers for each daughter.
```
function inherit(timer::MyTimer, time::Time)::MyTimer
 ...
end
```

**Note:** It may be tempting to simple return the parent timer if the cell inherits the timers but be aware the stepper will called for that timer for each cell that references it.

Once the initial population is created, the modeller needs to step the population in time.
```
function run(model::CellPopulation, runDuration::Time)
  counts = DataFrame(time=Float64[], count=Int[]) # A nice data frame for output
  ??t = modelTimeStep(model) # Get the time step

  # Caller the stepper and gather data
  for tm in 1:??t:runDuration
    step(model)
    push!(counts, (tm, length(model.cells)))
  end

  # Do something with the output
  h = plot(counts, x=:time, y=:count, Geom.line)
  display(h)
end
```

The model can now run for several parameters
```
for parms in allParameters
  population = createPopulation(nCells, (birth)->myCellFactory(birth, parms))
  run(model, runDuration)
end
```

## Sample models
We have implemented some models here and more will added over time. Visit our org page: [github.com/cyton-dynamics](https://github.com/cyton-dynamics)

### A Cyton like model
This model implements a [Cyton like model](https://www.frontiersin.org/articles/10.3389/fbinf.2021.723337/full) with timers for death and division. Timers are drawn from log normal distributions. Death timers are inherited and divisions timers are reset. The repo is here: [github.com/cyton-dynamics/simple-cyton](https://github.com/cyton-dynamics/simple-cyton).

### A stretched cell cycle model 
This a model of cell cycle. There is a single timer that determines whether the cell is in G1, S or G2/M. The model simulates the results of BrdU pulse staining and 7AAD point in time staining and writes Flow Cytometry Standard (FCS) files. The output reproduces fig 4A of this paper: [pubmed.ncbi.nlm.nih.gov/24733943/](https://pubmed.ncbi.nlm.nih.gov/24733943/).

The repo is here: [github.com/cyton-dynamics/cell-cycle](https://github.com/cyton-dynamics/cell-cycle)

### Homeostasis model
Population homeostasis in the presence of a division and signals driving Myc production. This is a work in progress.

The repo is here: [github.com/cyton-dynamics/homeostasis](https://github.com/cyton-dynamics/homeostasis)

## Getting started with Julia
The framework is written in Julia. The Julia type system is a bit unusual in that
- abstract types hold no state
- methods are not owned by types.
Once you get used to, it encourages the developer to separate implementation from types and behaviours. A core idea of the Cyton framework is that there are basic types and expected behaviours but details will differ from cell type to cell type and question to question.

These are the basic steps to install the Cyton package and run one of models. It is clearly not a Julia tutorial.

1. Install Julia from here [https://julialang.org/downloads/](https://julialang.org/downloads/). The software works with 1.7 and 1.8-beta. Installation is very lightweight, just unpack the tarball and add the bin directory to your path.
2. Clone one of the sample repos.
```
git clone git@github.com:cyton-dynamics/homeostasis.git
cd homeostasis
```
3. Start julia in that directory and in the package manager (hit `]`) 
```
dev https://github.com/cyton-dynamics/cyton.abm
add .
activate .
```
The `dev` command means that julia will make a local clone of the package and always load that. You can pull in the local directory to get updates.
4. Load the sample into the REPL
```
using homeostasis
```
5. Run the sample
```
include("run.jl")
```
6. In subsequent runs,
```
julia --project=<path to homeostasis>
```

It is highly recommended to use Visual Studio Code as the development environment: [https://www.julia-vscode.org/](https://www.julia-vscode.org/)
