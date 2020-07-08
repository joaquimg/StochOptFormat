# StochOptFormat: a novel data structure for multistage stochastic programming

This repository describes a data structure and file-format for stochastic
optimization problems called _StochOptFormat_, with the file extension
`.sof.json`.

For convenience, we often refer to StochOptFormat as _SOF_.

**Authors**

- [Oscar Dowson](http://github.com/odow) (Northwestern)
- [Joaquim Garcia](http://github.com/joaquimg) (PSR-Inc, PUC-Rio)

_Note: StochOptFormat is in development. Things may change! If you have
suggestions or comments, please open an issue._

## Motivation and design principles

Libraries of benchmark instances have been instrumental in driving progress in
many areas of optimization. In stochastic programming, some effort has been made
on this front (see, e.g., https://www.stoprog.org/resources). However, the
predominant file format for these problems, SMPS, does not scale to the types of
problems we want to solve (large multistage stochastic programs), nor does it
permit the variety of problem classes we want to study (e.g., stochastic conic
programs).

Over the last 4 years, we (and the broader JuMP team) have set about reimagining
how we formulate single period deterministic optimziation problems. The result
is a novel data structure for optimization called MathOptInterface [2]. In
addition, we have also set about standardizing how we formulate multistage
stochastic programming problems. The result is a natural decomposition of the
problem into the _policy graph_ [1].

**We highly recommend that you do not read further without reading (at minimum)
sections 1, 2, and 3 of [1] and sections 1, 2, and 5 of [2].**

These two workstreams are synergistic with each other. We can use the policy
graph to describe the high-level structure of a stochastic program, and we can
use MathOptInterface to describe the low-level optimization problem faced by the
agent within each stage (really, node of the policy graph). Putting these two
concepts together leads to a natural data structure for multistage stochastic
programs. StochOptFormat is a serialization of this data structure into the JSON
file format.

StochOptFormat is inspired by our work on [JuMP](https://jump.dev) and
[SDDP.jl](https://odow.github.io/SDDP.jl/latext). However, it is not exclusive
to Julia or stochastic dual dynamic programming. For example, this format makes
it possible to read in multistage stochastic programming problems into Python
and solve them with progressive hedging. We have not done so yet because this is
not our area of expertise.

In creating StochOptFormat, we wanted to achieve the following:

- We wanted a format that is able to scale to problems with hundreds of
  decision periods, state variables, and control variables.
- We wanted a format that was rigidly defined by a schema, so that files could
  be validated for syntactic correctness.
- We wanted a format that was not restricted to linear programming. We want to
  add cones and integrality.
- We wanted a format based on the policy graph. Doing so allows us to represent
  a very large class of problem structures, including finite and infinite
  horizon problems, problems with linear stagewise independence, problems with
  Markovian structure, and problems represented by an arbitrary scenario tree.
- We wanted a well-defined notion of what a solution to a stochastic program is.
  (Spoiler alert: it is not the first stage decision. See
  [Evaluating the policy](#evaluating-the-policy).)

Equally important as the things that we set out to do, is the things that we did
_not_ set out to do.

  - We did not try to incorporate chance constraints.
  - We did not try to incorporate continuous random variables.

## Example

There are alot of concepts to unpack in SOF. We present a simple example first,
and then explain each section in detail.

Consider a two-stage newsvendor problem. In the first stage, the agent chooses
`x`, the number of newspapers to buy at a cost \$1/newspaper. In the second
stage, the uncertain demand of `d` newspapers is realized, and the agent sells
`u` newspapers at a price of \$1.50/newspaper, with the constraint that
`u = min{x, d}`. The demand is a either 10 units with probability 0.4, or 14
units with probability 0.6.

### Vocabulary

StochOptFormat relies of a specific set of vocabulary. We summarize the
important terms here, and we direct the reader to [1] for more information.

1. Nodes

  A policy graph is made up of nodes. Each node correspond to a point in time
  at which the agent makes a decision.

  In our example, there are two nodes: `first_stage` and `second_stage`.

  For convenience, we also introduce a `root` node, which is not a node at which
  the agent makes a decision, but is just used as a marker for where the agent
  first enters the sequential decision making process. The root node stores the
  initial value for each state variable (see below).

2. Edges

  Edges connect two nodes in the policy graph. Each edge has a `from` node, a
  `to` node, and a `probability` of transitioning along the edge. Note that the
  sum of probabilities along outgoing edges of a node does not have to sum to 1.
  This allows, for example, discount factors in cyclic graphs and nodes with no
  children. See [1] for more details.

  In our example, there are two edges.
    1. `root` to `first_stage` with probability 1
    2. `first_stage` to `second_stage` with probability 1

3. State variables

  State variables are the information in the problem that flows between two
  nodes along an edge. In each node, each stage variable is associated with an
  _incoming_ state variable and an _outgoing_ state variable.

  In our example, there is one state variable, `x`, the number of newspapers
  purchased in the first stage. The _incoming_ state variable of `first_stage`
  represents the initial stock on hand, the _outgoing_ state variable of the
  `first_stage` is equivalent to the _incoming_ state variable of the
  `second_stage`, and the outgoing state variable of the `second_stage` is the
  quantity of unsold newspapers at the end of the second stage.

4. Random variables

  Random variables are local to a node, and are assumed to be independent of the
  incoming state variable and the realization of any prior random variables.
  Some nodes may not have any random variables. If so, we say that the node is
  deterministic.

  In our example, the `first_stage` is deterministic, and the `second_stage`
  node has one random variable, `d`, the demand.

5. Control variables

  Control variables are decisions taken by the agent within a node. They remain
  local to the node, and their values are not need by the agent in future nodes
  to make a decision.

5. Subproblem

The first-stage subproblem is:
```
  V₀(x) = max: -1 * x′
          s.t.      x′ >= 0,
```
and the second-stage is:
```
  V₁(x, d) = max: 1.5 * u
             s.t. u - x     <= 0
                  u     - d <= 0
                  u         >= 0.
```

### Problem in StochOptFormat

Encoded in StochOptFormat, the newsvendor problem becomes:
```json
{
  "author": "Oscar Dowson",
  "name": "newsvendor",
  "description": "An SOF implementation of the classical two-stage newsvendor problem.",
  "version": {"major": 0, "minor": 1},
  "root": {
    "name": "root",
    "state_variables": {
      "x": {"initial_value": 0.0}
    }
  },
  "nodes": {
    "first_stage": {
      "state_variables": {
        "x": {"in": "x_in", "out": "x_out"}
      },
      "random_variables": [],
      "subproblem": {
        "version": {"major": 0, "minor": 4},
        "variables": [{"name": "x_in"}, {"name": "x_out"}],
        "objective": {
          "sense": "max",
          "function": {
            "head": "ScalarAffineFunction",
            "terms": [{"variable": "x_out", "coefficient": -1.0}],
            "constant": 0.0
          }
        },
        "constraints": [{
          "function": {"head": "SingleVariable", "variable": "x_out"},
          "set": {"head": "GreaterThan", "lower": 0.0}
        }]
      },
      "noise_terms": []
    },
    "second_stage": {
      "state_variables": {
        "x": {"in": "x_in", "out": "x_out"}
      },
      "random_variables": ["d"],
      "subproblem": {
        "version": {"major": 0, "minor": 4},
        "variables": [
          {"name": "x_in"}, {"name": "x_out"}, {"name": "u"}, {"name": "d"}
        ],
        "objective": {
          "sense": "max",
          "function": {
            "head": "ScalarAffineFunction",
            "terms": [{"variable": "u", "coefficient": 1.5}],
            "constant": 0.0
          }
        },
        "constraints": [{
          "function": {
            "head": "ScalarAffineFunction",
            "terms": [
              {"variable": "u", "coefficient": 1.0},
              {"variable": "x_in", "coefficient": -1.0}
            ],
            "constant": 0.0
          },
          "set": {"head": "LessThan", "upper": 0.0}
        }, {
          "function": {
            "head": "ScalarAffineFunction",
            "terms": [
              {"variable": "u", "coefficient": 1.0},
              {"variable": "d", "coefficient": -1.0}
            ],
            "constant": 0.0
          },
          "set": {"head": "LessThan", "upper": 0.0}
        }, {
          "function": {"head": "SingleVariable", "variable": "u"},
          "set": {"head": "GreaterThan", "lower": 0.0}
        }]
      },
      "noise_terms": [
        {"probability": 0.4, "support": {"d": 10.0}},
        {"probability": 0.6, "support": {"d": 14.0}}
      ]
    }
  },
  "edges": [
    {"from": "root", "to": "first_stage", "probability": 1.0},
    {"from": "first_stage", "to": "second_stage", "probability": 1.0}
  ],
  "test_scenarios": [
    [
      {"node": "first_stage", "support": {}},
      {"node": "second_stage", "support": {"d": 10.0}}
    ], [
      {"node": "first_stage", "support": {}},
      {"node": "second_stage", "support": {"d": 14.0}}
    ], [
      {"node": "first_stage", "support": {}},
      {"node": "second_stage", "support": {"d": 9.0}}
    ]
  ]
}
```

### Explanation

SOF is a JSON document. The problem is stored as a single JSON object. JSON
objects are key-value mappings enclused by curly braces.

The file begins with three self-explanatory optional metadata fields:
`name::String`, `author::String`, and `description::String`.

Note: In the following, `name::String` means that the key of an object is `name`
and the value should be of type `String`. `::List{Object}` means that the type
is a `List`, and elements of the list are `Object`s.

Then, there are four required keys:

- `version::Object`

  An object describing the minimum version of MathOptFormat needed to parse
  the file. This is included to safeguard against later revisions. It contains
  two keys: `major` and `minor`. These keys should be interpreted using
  [SemVer](https://semver.org).

- `root::Object`

  An object describing the root node of the policy graph. It has two required
  keys:

  - `name::String`

    A unique string name for the root node to distinguish it from other nodes.

  - `state_variables::Object`

    An object describing the state variables in the problem. Each key is the
    unique name of a state variable. The value is an object with one required
    key:

    - `initial_value::Number`

      The value of the state variable at the root node.

- `nodes::Object`

  An object mapping the name of each node of the policy graph (excluding the
  root node) to an object describing the node. Each object has four required
  keys:

  - `state_variables::Object`

    An object that maps the name of each state variable (as defined in the root
    node) to an object describing the incoming and outgoing state variables in
    the subproblem. Each object has two required keys:

    - `in::String`

      The name of the variable representing the incoming state variable in the
      subproblem.

    - `out::String`

      The name of the variable representing the outgoing state variable in the
      subproblem.

  - `random_variables::List{String}`

    A list of strings describing the name of each random variable in the
    subproblem.

  - `subproblem::Object`

    The subproblem corresponding to the node as a MathOptFormat object.

  - `noise_terms::List{Object}`

    A list of objects describing the finite discrete realizations of the stagewise-independent noise term in each node. Each object has two required keys:

    - `probability::Number`

      The nominal probability of each realization.

    - `support::Object`

      An object describing the support corresponding to the realization. The
      keys of the object are the random variables declared in
      `random_variables`, and the values are the value of the random variable in
      that realization.

- `edges::List{Object}`

  A list of objects with one element for each edge in the policy graph. Each
  object has three required keys:

  - `from::String`

    The name of the node that the edge exits.

  - `to::String`

    The name of the node that the edge enters. This cannot be the root node.

  - `probability::Number`

    The nominal probability of transitioning from node `from` to node `to`.

In addition to the required fields, there is an additional field,
`test_scenarios::List{List{Object}}`, that is used for evaluating the
performance of a policy.

`test_scenarios` is a list, containing one element for each scenario in the test
set. Each scenario is a list of objects. Each object has two required nodes:
`node::String` and `support::Object`. `node` is the name of the node to visit,
and `support` is the realization of the random variable at that node. Note that
`support` may be an _out-of-sample_ realization, that is, one which is not
contained in the corresponding `noise_terms` field of the node. Testing a policy
is a larger topic, so we expand on it in the section
[Evaluating the policy](#evaluating-the-policy).

## Evaluating the policy

TODO

## FAQ

- Q: The policy graph is too complicated. I just want a format for linear
  T-stage stochastic programs.

  A: The policy graph does take some getting used to. But for a T-stage problem,
  our format requires T subproblems, a list of the state variables, and a
  sequence of edges. Of those things, only the list of edges would be
  superfluous in a purely T-stage format. So, for the sake of a list of objects
  like `{"from": "1", "to": "2", "probability": 1}`, we get a format that
  trivially extends to infinite horizon problems and problems with a stochastic
  process that is not stagewise independent.

- Q: MathOptFormat is too complicated. Why can't we use LP or MPS files?

  A: Please read Section 2 of [2].

- Q: You don't expect me to write these by hand do you?

  A: No. We expect high-level libraries like [SDDP.jl](https://github.com/odow/SDDP.jl)
  to do the reading and writing for you.

- Q: What happened to SMPS?

  A: SMPS is too limiting for multistage problems. We hope to implement a
  converter between SMPS and StochOptFormat at some point... Want to help?

- Q: This seems catered to SDDP; I just have some scenarios.

  A: The policy graph can represent any scenario tree. Go read [1].

- Q: I want continuous random variables.

  A: In this initial version of the format, we only consider finite discrete
  random variables. We might consider adding continuous onces in the future.

- Q: My stochastic process is not stagewise-independent.

  A: Two options: expand the state-space, or create a scenario tree. For more
  information, read Sections 1, 2, and 3 of [1].

- Q: I don't like JSON.

  A: We're open to better ideas. JSON is universal support in every major
  programming language, and is human-readable(-ish).

- Q: JSON seems too verbose.

  A: JSON files compress well. For example, for problems in the MIPLIB 2017
  benchmark set, compressed MathOptFormat files are only 37% larger than their
  compressed MPS equivalents.

- Q: I want the uncertainty to be an objective/constraint coefficient.

  A: Formulate the objective/constraint as a `ScalarQuadraticFunction`. It's up
  to the reader to infer from the list of the parameters if this is a
  parameterized `ScalarAffineFunction`, or a `ScalarQuadraticFunction` without
  parameters.

- Q: Follow up to the previous. I want to have `parameter * x * y`.

  A: Changing the quadratic coefficient matrices in solvers is slow, and doing
  so could easily make the problem non-convex. If you really want to, you could
  add a slack variable (and equality constraint) `z == parameter * x`, and then
  use `z * y`.

- Q: Why haven't you written an interface to ⟨INSERT LANGUAGE HERE⟩ yet?

  A: Does ⟨INSERT LANGUAGE HERE⟩ have a JSON reader? Just import the file!

## References

[1] Dowson, O. (2020). The policy graph decomposition of multistage stochastic
  programming problems. Networks, 71(1), 3-23.
  doi: https://onlinelibrary.wiley.com/doi/full/10.1002/net.21932
  [[preprint]](http://www.optimization-online.org/DB_HTML/2018/11/6914.html)

[2] Legat, B., Dowson, O., Garcia, J.D., Lubin, M. (2020). MathOptInterface: a
  data structure for mathematical optimization problems.
  [[preprint]](http://www.optimization-online.org/DB_HTML/2020/02/7609.html)
  [[repository]](https://github.com/jump-dev/MathOptFormat)
