
<!-- README.md is generated from README.Rmd. Please edit that file -->
nlrx <img src="man/figures/logo.png" align="right" width="150" />
=================================================================

[![AppVeyor build status](https://ci.appveyor.com/api/projects/status/github/nldoc/nlrx?branch=master&svg=true)](https://ci.appveyor.com/project/nldoc/nlrx) [![Travis build status](https://travis-ci.org/nldoc/nlrx.svg?branch=master)](https://travis-ci.org/nldoc/nlrx) [![lifecycle](https://img.shields.io/badge/lifecycle-experimental-orange.svg)](https://www.tidyverse.org/lifecycle/#experimental)

The nlrx package provides tools to setup NetLogo simulations in R. It uses a similar structure as NetLogos Behavior Space but offers more flexibility and additional tools for running sensitivity analyses.

Installation
------------

You can install the released version of nlrx from [CRAN](https://CRAN.R-project.org) with:

``` r
install.packages("nlrx")
```

And the development version from [GitHub](https://github.com/) with:

``` r
# install.packages("devtools")
devtools::install_github("nldoc/nlrx")
```

Example
-------

The nlrx package uses S4 classes to store basic information on NetLogo, the model experiment and the simulation design. Experiment and simulation design class objects are stored within the nl class object. This allows to have a complete simulation setup within one single R object.

The following steps guide you trough the process on how to setup, run and analyze NetLogo model simulations with nlrx

#### Step 1: Create a nl object:

The nl object holds all information on the NetLogo version, a path to the NetLogo directory with the defined version, a path to the model file, and the desired memory for the java virtual machine.

``` r
nl <- nl(nlversion = "6.0.3",
         nlpath = "C:/Program Files/NetLogo 6.0.3/",
         modelpath = "C:/Program Files/NetLogo 6.0.3/app/models/Sample Models/Biology/Wolf Sheep Predation.nlogo",
         jvmmem = 1024)
```

#### Step 2: Attach an experiment

The experiment object is organized in a similar fashion as NetLogo BehaviorSpace experiments. It holds information on model variables, constants, metrics, runtime, ...

``` r
nl@experiment <- experiment(expname="wolf-sheep",
                            outpath="C:/out/",
                            repetition=1,
                            tickmetrics="true",
                            idsetup="setup",
                            idgo="go",
                            idfinal=NA_character_,
                            idrunnum=NA_character_,
                            runtime=50,
                            evalticks=seq(40,50),
                            metrics=c("count sheep", "count wolves", "count patches with [pcolor = green]"),
                            variables = list('initial-number-sheep' = list(min=50, max=150, step=10, qfun="qunif"),
                                             'initial-number-wolves' = list(min=50, max=150, step=10, qfun="qunif")),
                            constants = list("model-version" = "\"sheep-wolves-grass\"",
                                             "grass-regrowth-time" = 30,
                                             "sheep-gain-from-food" = 4,
                                             "wolf-gain-from-food" = 20,
                                             "sheep-reproduce" = 4,
                                             "wolf-reproduce" = 5,
                                             "show-energy?" = "false"))
```

#### Step 3: Attach a simulation design

While the experiment defines the variables and specifications of the model, the simulation design creates a parameter input table based on these model specifications and the chosen simulation design method. nlrx provides a bunch of different simulation designs, such as full-factorial, latin-hypercube, sobol, morris and eFast. A simulation design is attached to a nl object by using one of these simdesign functions:

``` r
nl@simdesign <- simdesign_lhs(nl=nl,
                               samples=100,
                               nseeds=3,
                               precision=3)
```

#### Step 4: Run simulations

All information that is needed to run the simulations is now stored within the nl object. The run\_nl\_one() function allows to run one specific simulation from the siminput parameter table. The run\_nl\_all() function runs a loop over all simseeds and rows of the parameter input table siminput. The loops are created by calling furr::future\_map\_dfr which allows running the function either locally or on remote HPC machines.

``` r
future::plan(multisession)

results <- run_nl_all(nl = nl, cleanup = "all")
```

#### Step 5: Attach results to nl and run analysis

nlrx provides method specific analysis functions for each simulation design. Depending on the chosen design, the function reports a tibble with aggregated results or sensitivity indices. In order to run the analyze\_nl function, the simulation output has to be attached to the nl object first. After attaching the simulation results, these can also be written to the defined outpath of the experiment object.

``` r
# Attach results to nl object:
setsim(nl, "simoutput") <- results

# Write output to outpath of experiment within nl
write_simoutput(nl)

# Do further analysis:
analyze_nl(nl)
```

Further notes
-------------

#### Comments on simdesigns and variable definitions

Correctly defining variables within the experiment class object is crucial for creating simdesigns. The implemented simdesigns have different requirements for variable definitions:

| Simdesign              | Variable requirements                             | data type |
|------------------------|---------------------------------------------------|-----------|
| simdesign\_simple      | only constants are used                           | any       |
| simdesign\_distinct    | values (need to have equal length)                | any       |
| simdesign\_ff          | values, or min, max, step (values is prioritized) | any       |
| simdesign\_lhs         | min, max, qfun                                    | numeric   |
| simdesign\_sobol       | min, max, qfun                                    | numeric   |
| simdesign\_sobol2007   | min, max, qfun                                    | numeric   |
| simdesign\_soboljansen | min, max, qfun                                    | numeric   |
| simdesign\_morris      | min, max, qfun                                    | numeric   |
| simdesign\_eFast       | min, max, qfun                                    | numeric   |
| simdesign\_genSA       | min, max                                          | numeric   |
| simdesign\_genAlg      | min, max                                          | numeric   |

Categorical variable values are currently only allowed for simdesign\_simple, simdesign\_distinct and simdesign\_ff. Variable values that should be recognized by NetLogo as strings need to be nested inside escaped quotes (e.g. "\\"string\\""). Variable values that should be recognized by NetLogo as logical need to be entered as strings (e.g. "false").

#### Comments on self-written output

The experiment provides a slot called "idrunnum". This slot can be used to transfer the current nlrx runnumber (siminputrow) to NetLogo. To use this functionality, a slider or numeric input field widget needs to be created on the GUI of your NetLogo model. The name of this widget can be entered into the "idrunnum" field of the experiment. During simulations, the value of this widget is automatically set to the current siminputrow number. For self-written output In NetLogo, we suggest to include this global vairable and the current random-seed, which allows referecing this output to the collected output of the nlrx simulations.

#### Comments on seed management

The experiment provides a slot called "repetition" which allows to run multiple simulations of one parameterization. This is only useful if you manually generate a new random-seed during the setup of your model. By default, the NetLogo random-seed is set by the simdesign that is attached to your nl object. If your model does not reset the random seed manually, the seed will always be the same for each repetition.

However, the concept of nlrx is based on sensitivity analyses. Here, you may want to exclude stoachsticity from your output and instead do multiple sensitivity analyses with the same parameter matrix but different random seeds. You can then observe the effect of stochasticity on the level of your final output, the sensitivity indices. Thus we suggest to set the experiment repetition to 1 and instead use the nseeds variable of the desired simdesign to run multiple simulations with different random seeds.

In summary, if you set the random-seed of your NetLogo model manually, you can increase the repitition of the experiment to run several simulations with equal parameterization and different random seeds. Otherwise, set the experiment repetition to 1 and increase the nseeds variable of your desired simdesign.

#### Comments on measurements

Three slots of the experiment class define how measurements are taken:

-   tickmetrics, defines if measurements are taken at the end of the simulation or on each tick

-   evalticks, if tickmetrics = "true" evalticks can be used to filter the results for defined ticks

-   metrics, definition of valid NetLogo reporters that are used to collect data

Due to the evalticks definition, it might happen, that a simulation stops before any output has been collected. In such cases, output is still reported but all metrics that could not be collected for any defined evalticks will be filled up with NA.

Further Wolf Sheep examples for all implemented simdesigns
----------------------------------------------------------

The following section provides valid experiment setups for all supported simdesigns using the Wolf Sheep Model from the NetLogo models library.

First we set up a nl object. We use this nl object for all simdesigns:

``` r
nl <- nl(nlversion = "6.0.3",
         nlpath = "C:/Program Files/NetLogo 6.0.3/",
         modelpath = "C:/Program Files/NetLogo 6.0.3/app/models/Sample Models/Biology/Wolf Sheep Predation.nlogo",
         jvmmem = 1024)
```

##### simdesign\_simple

To setup a simple simdesign, no variables have to be defined. The simple simdesign only uses defined constants and reports a parameter matrix with only one parameterization.

``` r
nl@experiment <- experiment(expname="wolf-sheep",
                            outpath="C:/out/",
                            repetition=1,
                            tickmetrics="true",
                            idsetup="setup",
                            idgo="go",
                            idfinal=NA_character_,
                            idrunnum=NA_character_,
                            runtime=50,
                            evalticks=seq(40,50),
                            metrics=c("count sheep", "count wolves", "count patches with [pcolor = green]"),
                            variables = list(),
                            constants = list("initial-number-sheep" = 20,
                                             "initial-number-wolves" = 20,
                                             "model-version" = "\"sheep-wolves-grass\"",
                                             "grass-regrowth-time" = 30,
                                             "sheep-gain-from-food" = 4,
                                             "wolf-gain-from-food" = 20,
                                             "sheep-reproduce" = 4,
                                             "wolf-reproduce" = 5,
                                             "show-energy?" = "false"))

nl@simdesign <- simdesign_simple(nl=nl,
                                 nseeds=3)
```

##### simdesign\_distinct

To setup a distinct simdesign, vectors of values need to be defined for each variable. These vectors must have the same number of elemtents across all variables.

``` r
nl@experiment <- experiment(expname="wolf-sheep",
                            outpath="C:/out/",
                            repetition=1,
                            tickmetrics="true",
                            idsetup="setup",
                            idgo="go",
                            idfinal=NA_character_,
                            idrunnum=NA_character_,
                            runtime=50,
                            evalticks=seq(40,50),
                            metrics=c("count sheep", "count wolves", "count patches with [pcolor = green]"),
                            variables = list('initial-number-sheep' = list(values=c(10, 20, 30, 40)),
                                             'initial-number-wolves' = list(values=c(30, 40, 50, 60))),
                            constants = list("model-version" = "\"sheep-wolves-grass\"",
                                             "grass-regrowth-time" = 30,
                                             "sheep-gain-from-food" = 4,
                                             "wolf-gain-from-food" = 20,
                                             "sheep-reproduce" = 4,
                                             "wolf-reproduce" = 5,
                                             "show-energy?" = "false"))

nl@simdesign <- simdesign_distinct(nl=nl,
                                   nseeds=3)
```

##### simdesign\_ff

To setup a full-factorial simdesign, vectors of values need to be defined for each variables. Alternatively, a sequence can be defined by setting min, max and step. However, if both (values and min, max, step) are defined, the values vector is prioritized.

``` r
nl@experiment <- experiment(expname="wolf-sheep",
                            outpath="C:/out/",
                            repetition=1,
                            tickmetrics="true",
                            idsetup="setup",
                            idgo="go",
                            idfinal=NA_character_,
                            idrunnum=NA_character_,
                            runtime=50,
                            evalticks=seq(40,50),
                            metrics=c("count sheep", "count wolves", "count patches with [pcolor = green]"),
                            variables = list('initial-number-sheep' = list(values=c(10, 20, 30, 40)),
                                             'initial-number-wolves' = list(min=0, max=50, step=10)),
                            constants = list("model-version" = "\"sheep-wolves-grass\"",
                                             "grass-regrowth-time" = 30,
                                             "sheep-gain-from-food" = 4,
                                             "wolf-gain-from-food" = 20,
                                             "sheep-reproduce" = 4,
                                             "wolf-reproduce" = 5,
                                             "show-energy?" = "false"))

nl@simdesign <- simdesign_ff(nl=nl,
                             nseeds=3)
```

##### simdesign\_lhs, \_sobol, \_sobol2007, \_soboljansen, \_morris, \_eFast

To setup sensitivity analysis simdesigns, variable distributions (min, max, qfun) need to be defined.

``` r
nl@experiment <- experiment(expname="wolf-sheep",
                            outpath="C:/out/",
                            repetition=1,
                            tickmetrics="true",
                            idsetup="setup",
                            idgo="go",
                            idfinal=NA_character_,
                            idrunnum=NA_character_,
                            runtime=50,
                            evalticks=seq(40,50),
                            metrics=c("count sheep", "count wolves", "count patches with [pcolor = green]"),
                            variables = list('initial-number-sheep' = list(min=50, max=150, qfun="qunif"),
                                             'initial-number-wolves' = list(min=50, max=150, qfun="qunif")),
                            constants = list("model-version" = "\"sheep-wolves-grass\"",
                                             "grass-regrowth-time" = 30,
                                             "sheep-gain-from-food" = 4,
                                             "wolf-gain-from-food" = 20,
                                             "sheep-reproduce" = 4,
                                             "wolf-reproduce" = 5,
                                             "show-energy?" = "false"))

nl@simdesign <- simdesign_lhs(nl=nl,
                               samples=100,
                               nseeds=3,
                               precision=3)


nl@simdesign <- simdesign_sobol(nl=nl,
                                 samples=200,
                                 sobolorder=2,
                                 sobolnboot=20,
                                 sobolconf=0.95,
                                 nseeds=3,
                                 precision=3)

nl@simdesign <- simdesign_sobol2007(nl=nl,
                                     samples=200,
                                     sobolnboot=20,
                                     sobolconf=0.95,
                                     nseeds=3,
                                     precision=3)

nl@simdesign <- simdesign_soboljansen(nl=nl,
                                       samples=200,
                                       sobolnboot=20,
                                       sobolconf=0.95,
                                       nseeds=3,
                                       precision=3)


nl@simdesign <- simdesign_morris(nl=nl,
                                  morristype="oat",
                                  morrislevels=4,
                                  morrisr=100,
                                  morrisgridjump=2,
                                  nseeds=3)

nl@simdesign <- simdesign_eFast(nl=nl,
                                 samples=100,
                                 nseeds=3)
```

##### simdesign\_GenSA, \_GenAlg

To setup optimization simdesigns, variable ranges (min, max) need to be defined.

``` r
nl@experiment <- experiment(expname="wolf-sheep",
                            outpath="C:/out/",
                            repetition=1,
                            tickmetrics="true",
                            idsetup="setup",
                            idgo="go",
                            idfinal=NA_character_,
                            idrunnum=NA_character_,
                            runtime=50,
                            evalticks=seq(40,50),
                            metrics=c("count sheep", "count wolves", "count patches with [pcolor = green]"),
                            variables = list('initial-number-sheep' = list(min=50, max=150),
                                             'initial-number-wolves' = list(min=50, max=150)),
                            constants = list("model-version" = "\"sheep-wolves-grass\"",
                                             "grass-regrowth-time" = 30,
                                             "sheep-gain-from-food" = 4,
                                             "wolf-gain-from-food" = 20,
                                             "sheep-reproduce" = 4,
                                             "wolf-reproduce" = 5,
                                             "show-energy?" = "false"))

nl@simdesign <- simdesign_GenAlg(nl=nl, 
                                 popSize = 200, 
                                 iters = 100, 
                                 evalcrit = 1,
                                 elitism = NA, 
                                 mutationChance = NA, 
                                 nseeds = 1)

nl@simdesign <- simdesign_GenSA(nl=nl,
                                par=NULL,
                                evalcrit=1,
                                control=list(max.time = 600),
                                nseeds=1)
```
