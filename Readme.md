[![codebeat badge](https://codebeat.co/badges/f72db301-fd66-4c05-b1ca-9b8c8196f06e)](https://codebeat.co/projects/github-com-csxeba-evolute-master)
# Evolute
Evolutionary algorithm toolbox

## Documentation

**Evolute** is a simple tool for quick experimentation with evolutionary
algorithms for numerical optimization.

It defines a **population** of individuals, represented as floating point
vectors, and applies a configurable set of evolutionary **operators** to
them in a predefined order.

The order is the following:
1. **Selection**: a subset of individuals are discarded, depending on
their fitness value
2. **Reproduction**: the discarded individuals are replaced by new
individuals, somehow generated from the survivors
3. **Mutation**: mutate some of the individuals
4. **Update**: update the fitnesses of the individuals 

Some nomenclature:
- **individual**: a single member of the population.
- **genotype**: refers to an individual in the encoded state.
*Evolute* encodes information in floating point vectors.
- **locus**: a member of a genotype (a single scalar in the vector)
- **phenotype**: refers to an individual in the decoded state. In the
case of neuroevolution, this would be a neural network object, which
is encoded as a genotype. Evolute is not doing any *genotype - phenotype*
conversion, this has to be implemented by the user in the fitness
calculation.


### Module: *population*

Defines containers for individuals and aggregates the operators
applied to them. Currently the following population types are defined:
- **GeneticPopulation**: can be used for genetic algorithms, simply
evaluates the fitnesses and applies the operators in the above specified
order. Its constructor accepts the following arguments:
  - **loci**: number of elements in an individual's chromosome
  - **fitness_wrapper**: instance of a class in evolute.evaluation
  - **limit**: maximum number of individuals, defaults to 100
  - **operators**: an instance of *evolute.operators.Operators*, see operators later
  - **initializer**: instance of a class defined in evolute.initialization, optional

Will add support for MemeticPopulation, which allows for explicit modification
of the individuals during fitness calculation

An evolutionary optimization can be run using the population.run() method, which
accepts the following arguments:
- **epochs**: how many iterations should be done
- **survival_rate**: reset survival rate
- **mutation_rate**: reset mutation rate
- **force_update_at_every**: at every *k*th iteration, force a complete
fitness-recalculation
- **verbosity**: 0 is silent, > 0 is verbose.
- **history**: evolute.utils.history.History object, in which runtime metrics
can be recorded. If omitted, one is instantiated implicitly.

The run() method returns a history object with generation, best_grade, mean_grade
and grade_stdev recorded by default. 

A population can be saved and loaded with the save() and load() methods, which
produces a gzip-compressed pickle of the Population object.

Single epoch can be also run with the epoch() method and an update can be forced
with the update() method.

For convenience, these methods are defined:
- **epoch()**: use this to evaluate a single epoch
- **update()**: forces a fitness update
- **simple_fitness()**: class factory method, which takes a nude fitness function
and wraps it implicitly with SimpleFitness (see below).
- **get_individual(index)**
- **set_indivudual(index, individual)**
- **get_best()**: the current best individual with the lowest fitness/grade
- **get_champion**: the all-time best individual is alway stored implicitly,
and can be accessed here.
 

### Module: *evaluation*

Defines different wrappers for fitness functions.
**Fitness** is a scalar value assigned to every individual.
Currently in Evolute, the lower fitness is the better.

Since a fitness function can be any kind of logic, fitness wrappers
expect a function pointer or some kind of callable. Some advanced
wrappers are there to support possibly multiple fitness functions
or functions with multiple return values. In the end, the fitness
has to be reduced to a single scalar.

The process of combining multiple fitnesses to a single scalar is
termed **grading** in Evolute.

#### Fitness wrappers

Currently the following fitness wappers are defined in
*evolute.evaluation*:

- **SimpleFitness**: wraps a simple function with a single scalar
return value. Its constructor expects the following arguments:
  - **fitness_function**: a function reference or callable object
  - **constants**: optional dictionary of constants to be passed by name

Variables may be passed to the fitness function during the running of
the algorithm.

- **MultipleFitnesses**: wraps multiple fitness functions. The
constructor expects the following arguments:
  - **functions_by_name**: dictionary of fitness function references
or callable objects. They need to be identified by a unique name or ID
  - **contants_by_function_name**: optional dictionary of arguments (as dicts as well).
They have to be referenced by the same name or ID.
  - **order_by_name**: optional, an ordered iterable of the IDs if call order
matters.
  - **grader**: grader object or any callable which takes the array of
fitnesses and produces a single scalar grade from them. Defaults to
simple summation.

- **MultiReturnFitness**: wrapper for a single fitness function which
returns multiple fitness values. The constructor expects the following
arguments:
  - **number_of_return_values**
  - **constants**: optional
  - **grader**: optional, defaults to simple summation
  
#### Graders

More advanced fitness wrappers expect a Grader instance defined in
*evolute.evaluation.grade* or any callable which accepts a NumPy
array and returns a single scalar.

Some graders are available in *evolute.evaluation*:

- **SumGrader**: simply sums the fitness values
- **WeightedSumGrader**: accepts a set of weights in its constructor
and produces a weighted sum (dot product) with the fitness values.

### Module: *initialization*

This module defines initializers: objects for random population
initialization strategies. Currently the following distributions
are available in *evolute.initialization*:

- **NormalRandom**: with customizable **mean** and **stdev**
- **UniformRandom**: with customizable **low** and **high**
- **OrthogonalNormal**: produces a diagonal random matrix 
 
### Module: *operators*
 
Evolutionary operators are defined here. This module defines an
**Operators** class, which aggregates all the operators needed to
run an evolutionary optimization. Operators' constructor takes the
following arguments:
- **selection_op**
- **mutate_op**
- **mate_op**

Submodules define the particular operator types, which are the following:

#### Selection

During selection, a subset of individuals are discarded. The logic by
which these are specified depends on the actual implementation:

- **Elitism**: selects the top m% of individuals. Its constructor
accepts the following arguments:
  - **selection_rate**: scalar between 0 and 1 to specify the rate of
discarded individuals
  - **mate_op**: a mate operator, which is used to fill up the slots
where individuals are missing (applies reproduction). Details on these
later.
  - **exclude_self_mating**: bool, whether to disallow mating an individual
with itself

If no selection operator is defined, every selection-using class defaults
to *Elitism*.

#### Mate

Mating is defined as some kind of combination of two individuals to
reproduce a new individual. Currently the following mate operators
are defined:

- **LambdaMate**: wraps a user-defined function, which takes two
individuals and produces a single individual
- **RandomPickMate**: randomly pics loci from the two genotypes
- **SmoothMate**: takes the mean of the two genotypes
- **ScatterMateWrapper**: wrapper for any mate operator. When applied,
it adds gaussian noise to the new individual. Its constructor expects
the following parameters:
  - **base**: reference to the base mate operator object
  - **stdev**: standard deviation of the additive gaussian noise
  
If no mate operator is defined, every mate-using class defaults to
*RandomPickMate*.

#### Mutate

This operator takes the whole population and mutates its individuals
by perturbing them with some kind of noise. The following mutation
operators are available:

- **UniformLocuswiseMutation**: adds uniform distributed noise to
individuals. Mutants are determined on the locus level, and mutation
rate (see below) is adjusted for this. Its constructor takes the
following arguments:
  - **rate**: rate by which mutations occur in the population. It has
  to be given as a rate of individuals but it is implicitly corrected
  for loci.
  - **low** and **high**: parameters of the uniform distribution
- **NormalIndividualwiseMutation**: adds normal random noise to
individuals. Mutants are determined on the individuals' level. Its
constructor takes the following arguments:
  - **rate**
  - **stdev**
 
If no mutate operator is defined, every mutate-using class defaults
to *UniformLocuswiseMutation*.

### Module: utility

Some useful stuff here. Maybe the most interesting is the
*evolute.keras_utility* module, which defines some helpers to interface
with Keras and do some Neuroevolution on the weights of a network.
