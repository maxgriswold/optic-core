# OPTIC Core

## Installing Package

#### Clone

Clone the repository to your machine. Once you have the project on your local machine, the first thing you want to do is build and install the package. Open the project in RStudio - easiest to just open the optic.Rproj file. Make sure you have `devtools` installed.

```R
# fresh build of the package documentation and source code
devtools::document()
devtools::build()

# install optic package and dependencies
devtools::install()
```

## Running Simulation

Example below dependent on access to `optic_sim_data_exp.Rdata` data object.

#### Configure Simulation

The first step is to congifure the simulation or set of simulations you wish to run. There four main components to the configuration object:

1.  **Basic Parameters**: this includes straighforward arguments like the data, number of iterations, and verbosity
1.  **Models**: models are provided to the configuration method in a nested list where each model you want to test/use is a list element with the necessary named values based on the simulation being run (e.g., concurrent or selection bias)
1.  **Methods**: the configuration object requires at least three methods - `method_sample`, `method_model`, and `method_results`. These "slots", if you will, are intended for the user to provide their sampling, treatment identification, and common data transformation code; modeling code; and code to extract results, fit information, summary statistics, etc. respectively. There are also two optional slots - `method_premodel` and `method_postmodel` that if provided will be applied to the simulation run immediately before and immediately after the `method_model`. See the section on "How an Iteration is Run" below.
1.  **Iterable and Global Parameters**: The `params` argument is where you provide the named values that will be expanded into a full set of all unique combination of all named elements in the provided list. The `globals` argument should also be a list of named elements that are statically stored for access by any of the methods and will not be expanded with the `params` values.

Here we will walk through an example of setting up the configuration object,  exploring the configuration object prior to running the job, and looking at how a single simulation is defined. For our example, we will look at evaluating a linear 2-way fixed effects model
compared to a linear autoregressive model under a context of co-occurring policies. Let's assume we have some data and some setup:

```R
x <- readRDS("our_data.rds")

# we will define two scenarios for different effect magnitudes using
# 5, 10, and 15 percent changes in the outcome
linear5 <- 690 / ((sum(x$population) / length(unique(x$year))) / 100000)
linear10 <- 1380 / ((sum(x$population) / length(unique(x$year))) / 100000)
linear15 <- 2070 / ((sum(x$population) / length(unique(x$year))) / 100000)

scenario1 <- c(linear10, linear10)
scenario2 <- c(linear5, linear15)
```

Let's first setup our models object. Note: the methods applied are already defined in `R/concurrent-methods.R`. For each element in the list, we will define the same named attributes:
-  name: a name for the model to idetify it in output
-  type: this is used in some of our methods to transform the data correctly for the type of model being run
-  model_call: the R function name
-  model_formula: the formula to run with the specified model_call
-  model_args: will be passed as additional arguments when running model
-  se_adjust: in this case our post-processing can perform some standard error adjustments and we only want two here

```R
my_models <- list(
  list(
    name="fixedeff_linear",
    type="reg",
    model_call="lm",
    model_formula=crude.rate ~ unemploymentrate + as.factor(year) + as.factor(state) + treatment1_level + treatment2_level,
    model_args=list(weights=as.name('population')),
    se_adjust=c("none", "cluster")
  ),
  list(
    name="autoreg_linear",
    type="autoreg",
    model_call="lm",
    model_formula=deaths ~ unemploymentrate + as.factor(year) + treatment1_change + treatment2_change,
    model_args=list(weights=as.name('population')),
    se_adjust=c("none", "cluster")
  )
)
```

Now we will setup the configuration object that will be used to dispatch each of the simulations. In this case we will tell the configuration function our data, the models we specified above, and the numnber of iterations we want for each simulation. In a production or real simulation we probably want 5,000 iterations, but we are using 10 here for illustrative purposes. We pass each of the models defined in `R/concurrent-methods.R` to their respective arguments, then define the `params` that will be expanded and we will compare with this simulation. This simulation does not have any globals.

```R
sim_config <- configure_simulation(
  x=x,
  models=my_models,
  iters=10,
  
  method_sample=concurrent_sample,
  method_pre_model=concurrent_premodel,
  method_model=concurrent_model,
  method_post_model=concurrent_postmodel,
  method_results=concurrent_results,
  
  params=list(
    unit_var="state",
    time_var="year",
    effect_magnitude=list(scenario1, scenario2),
    n_units=c(30),
    effect_direction=c("null", "neg"),
    policy_speed=c("instant", "slow"),
    n_implementation_periods=c(3),
    rhos=c(0, 0.25, 0.5, 0.75, 0.9)
  )
)
```

Once the configuration object is created, you can take a look at a couple of key things:
1.  Did the `params` expand as you expected?
1.  Setup a single simulation to get a sense for how it works.

```R
# if we look at object, it is an R6 class object and has a number of named elements
> class(sim_config)
[1] "SimConfig" "R6"

> names(sim_config)
 [1] ".__enclos_env__"         "method_results"          "method_post_model"      
 [4] "method_model"            "method_pre_model"        "method_sample"          
 [7] "simulation_params"       "globals"                 "params"                 
[10] "iters"                   "models"                  "data"                   
[13] "clone"                   "print"                   "setup_single_simulation"
[16] "initialize"

# the "simulation_params" element is the expanded data frame
> head(sim_config$simulation_params)
  unit_var time_var effect_magnitude1 effect_magnitude2 n_units effect_direction policy_speed n_implementation_periods rhos
1    state     year         0.4579364         0.4579364      30             null      instant                        3    0
2    state     year         0.2289682         0.6869046      30             null      instant                        3    0
3    state     year         0.4579364         0.4579364      30              neg      instant                        3    0
4    state     year         0.2289682         0.6869046      30              neg      instant                        3    0
5    state     year         0.4579364         0.4579364      30             null         slow                        3    0
6    state     year         0.2289682         0.6869046      30             null         slow                        3    0

# the "setup_single_simulation" element is actually an internal method for this class
# what it does is take an integer, 1:nrow(simulation_params) table, pulls the values
# from that row in the table and all the static values in the object and creates
# a single simulation object:
example_single_sim <- sim_config$setup_single_simulation(1)

> names(example_single_sim)
 [1] "unit_var"                 "time_var"                 "effect_magnitude1"       
 [4] "effect_magnitude2"        "n_units"                  "effect_direction"        
 [7] "policy_speed"             "n_implementation_periods" "rhos"                    
[10] "data"                     "models"                   "iters"                   
[13] "method_sample"            "method_pre_model"         "method_model"            
[16] "method_post_model"        "method_results"
```

#### Dispatch Simulation

When you run a simulation you use the `dispatch_simulation` method, which will iterate over the rows in the `"simulation_params"` element and run the iterations for each simulation either in a loop or in parallel depending on how you choose to dispatch the job.

```R
# use_future will run the iterations in parallel, in this case we are also showing
# how you may need to specify certain environment inclusions like a specific method
# or package to ensure it is used in each of the threads the future library will
# create
# you can also set a seed, which will work for both single-threaded and multi-
# threaded runs to ensure replication is possible.
lm_results <- dispatch_simulations(
    sim_config,
    use_future=TRUE,
    seed=9782,
    verbose=2,
    future.globals=c("cluster_adjust_se"),
    future.packages=c("dplyr", "optic")
)

# we get back a list with a data.frame of results for each of the 40 unqiue
# combinations of parameters we provided in the configuration object
> length(lm_results)
[1] 40

# take a look at one of the data.frames if you would like
head(lm_results[[1]])
```

#### How an Iteration is Run

When constructing methods, it is important to understand how they are going to be implemented. It's relatively straightforward and you can check out the `R/run-iteration.R` program for a quick look. Essentially, each method you write should take a single list object and return a single list object that has all the information needed for subsequent methods. Only the `method_results` function should return the "final product" - in the above case a data.frame. Here is the logic:

A `single_simulation` object (list) is created and provided to the `run_iteration` function. The `run_iteration` function goes through the methods provided by the user in the following order:

1.  The `method_sample` function is applied to the list and the returned object overwrites `single_simulation`. This is the sampling and common data transformation step. Everything after this step will be looped anf performed once per model element provided in the models argument - in our example above there are two models elements.
1.  Then, the `run_iteration` method enters a loop - looping over each of the models provided, creates a new `model_simulation` object that is a copy of the `single_simulation` object. This isolates each model to start from the same intitial sampling and data transformation so any further data altering or changes to the object by other methods are not retained/impacting subsequent models' runs. The rest of the methods are then run in order:
    1.  If provided, runs `method_pre_model` passing `model_simulation` as the sole argument, the return object is stored in `model_simulation`
    1.  Runs the `method_model` function passing `model_simulation` as the sole argument and overwriting the object with the returned object
    1.  If provided, runs the `method_post_model`, the return object is assigned to a list whose named element is the model name
    1.  If no post model method is provided, the return object from the `method_model` function is assigned to the list directly
1.  The list of independent model results is provided as the argument to `method_results` and the return object from this final function is returned by the `run_iteration` function.

