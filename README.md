
<!-- README.md is generated from README.Rmd. Please edit that file -->
nlrx <img src="man/figures/logo.png" align="right" width="150" />
=================================================================

[![AppVeyor build status](https://ci.appveyor.com/api/projects/status/github/nldoc/nlrx?branch=master&svg=true)](https://ci.appveyor.com/project/nldoc/nlrx) [![Travis build status](https://travis-ci.org/nldoc/nlrx.svg?branch=master)](https://travis-ci.org/nldoc/nlrx) [![Codecov test coverage](https://codecov.io/gh/nldoc/nlrx/branch/master/graph/badge.svg)](https://codecov.io/gh/nldoc/nlrx?branch=master) [![lifecycle](https://img.shields.io/badge/lifecycle-experimental-orange.svg)](https://www.tidyverse.org/lifecycle/#experimental)

The nlrx package provides tools to setup NetLogo simulations in R. It uses a similar structure as NetLogos Behavior Space but offers more flexibility and additional tools for running sensitivity analyses.

Installation
------------

~~You can install the released version of nlrx from~~ ~~[CRAN](https://CRAN.R-project.org) with:~~ ~~install.packages("nlrx")~~

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

results %<-% run_nl_all(nl = nl, cleanup = "all")
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

Meta
----

-   Please [report any issues or bugs](https://github.com/nldoc/nlrx/issues/new/).
-   License: GPL3
-   Get citation information for `nlrx` in R doing `citation(package = 'nlrx')`
-   We are very open to contributions - if you are interested check [Contributing](CONTRIBUTING.md).
    -   Please note that this project is released with a [Contributor Code of Conduct](CODE_OF_CONDUCT.md). By participating in this project you agree to abide by its terms.
