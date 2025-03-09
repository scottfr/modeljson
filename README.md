# ModelJSON

## Objective

Provide a JSON format for defining system models with optional simulation capabilities.

Supported model types include:

- Causal Loop Diagrams  
- Stock and Flow Diagrams (System Dynamics and Differential Equation models)  
- State and Transition Diagrams  

The key goals of the format are:

- Minimal and simple
- Usable for diverse applications
- Reasonably readable and hand-editable
- Suitable for generation by LLMs

## Non-Goals

- **Standardize formula syntax or functions across modeling applications.**  
  Applications have varied capabilities, and aligning those is not an aim of this format.  
- **Serve as the primary save file format for a modeling application.**  
  This format is designed to be minimal. Applications may provide import/export support for ModelJSON, but the format does not seek to cover all the features of any given modeling application. 

## Schema

### Root Object

The root of the schema is a JSON object with the following properties:

- `name` {string} [optional] – A string that provides the name of the model.
- `description` {string} [optional] – A description of the model.
- `file_notes` {string} [optional] - Notes on any issues that were encountered generating the ModelJSON file. This can be used to flag incompatibility issues or things that were stripped during export. It may be displayed to users on import.
- `engine` {"SIMULATION_PACKAGE"} [optional] – A string indicating a specific dialect of ModelJSON that is being used. This determines the syntax and behavior for formulas and units. See the [Engines](#engines) section below for more details. 
- `simulation` {SimulationObject} [optional] – An object specifying the settings and parameters of the simulation.  
- `elements` {ElementObject[]} [required] – An array of objects, each representing a basic element in the simulation (e.g., stocks, flows, variables, or links).  
- `visualizations` {VisualizationObject[]} [optional] – Array of visualizations for simulation results.

### SimulationObject

- `algorithm` {"RK1"|"RK4"} [optional] – Numerical algorithm for the simulation. "RK1" denotes Euler's method, and "RK4" a fourth-order Runge-Kutta.  
- `time_start` {number} [optional] – The start time of the simulation.  
- `time_length` {number} [optional] – The total length of the simulation.  
- `time_step` {number} [optional] – The time step of the simulation.  
- `time_units` {"SECONDS"|"MINUTES"|"HOURS"|"DAYS"|"WEEKS"|"MONTHS"|"YEARS"} [optional] – The time units `time_start`, `time_length` and `time_step` are defined in these units.

### ElementObject

- `type` {"STOCK"|"FLOW"|"VARIABLE"|"LINK"|"CONVERTER"|"STATE"|"TRANSITION"} [required] – The type of element.
- `name` {string} [optional for type `LINK`, required otherwise] – A name for the element. Names are case-insensitive.
- `description` {string} [optional] – Additional descriptive text about the element.  
- `display`
  - `interactive` {boolean} [optional] – If `true`, indicates that it should be simple for a user to adjust this element’s value (e.g., with an interactive slider).  

Depending on the `type`, other properties may be required or optional. See [Element Types](#element-types) below.

### VisualizationObject

- `type` {"TIME_SERIES"|"TABLE"} [required] - The type of the visualization.
- `name` {string} [optional] – A name for the visualization (may be displayed as a graph/table title).
- `elements` {string[]} [required] - An array of elements to include in the visualization.

### Element Types

#### Stock

Stocks are state variables that store a quantity (of water, money, individuals, etc...).

The following additional properties are available:

- `behavior`
  - `initial_value` {number|string} [optional] – The initial value of the stock.  
  - `non_negative` {boolean} [optional] – If `true`, the stock cannot become negative. Defaults to `false`.  
  - `units` {string} [optional] – The units of the stock's value.  

Additionally, the properties from the [node](#node) and [interactive-scalar](#interactive-scalar) mixins are included.

#### Flow

Flows move quantities between stocks.

The following additional properties are available:

- `behavior`
  - `value` {string|number} [optional] – A formula or numeric rate for the flow.  
  - `non_negative` {boolean} [optional] – If `true`, any negative flow value is set to 0. Defaults to `false`.
  - `units` {string} [optional] – The units of the flow's value.  

Note that Flows can only connect to and from stock elements.

Additionally, the properties from the [connector](#connector) and [interactive-scalar](#interactive-scalar) mixins are included.

#### Variable

Variables are constants or formulas that are recalculated. In Causal Loop Diagrams, Variables should generally be used as the nodes.

The following additional properties are available:

- `behavior`
  - `value` {number|string} [optional] – A formula or numeric value for the variable's value.  
  - `units` {string} [optional] – The units for the variable's value.  

Additionally, the properties from the [node](#node) and [interactive-scalar](#interactive-scalar) mixins are included.

#### Link

Links connect elements together. Elements that are not connected in some way (via a link, transition or flow) should not directly influence each other.

The following additional properties are available:

- `behavior`
  - `polarity` {"POSITIVE"|"NEGATIVE"|"NEUTRAL"} [optional] – The polarity of the link.

Additionally, the properties from the [connector](#connector) mixin are included.

#### Converter

A Converter is a table of input/output values. The input to the converter is looked up in the table to determine the output.

The following additional properties are available:

- `behavior`
  - `input` {"TIME"|"ELEMENT"} [optional] – If `TIME`, the input for the converter is the current time (in simulation time units). If `ELEMENT`, the input is the value of another element.  
  - `input_element` {string} [optional] – If `input` is `ELEMENT`, the name of that element.  
  - `interpolation` {"NONE"|"LINEAR"} [optional] – How to interpolate between values.  
  - `data` {[number, number][]} [optional] – An array of `[input, output]` pairs.  
  - `units` {string} [optional] – The units for the converter's output.  

Additionally, the properties from the [node](#node) mixin are included.

#### State

A State is boolean, `true`/`false` state variable.

The following additional properties are available:

- `behavior`
  - `initial_value` {string|boolean} [optional] – A formula indicating whether the state is initially active (`true`) or not (`false`).  

Additionally, the properties from the [node](#node) mixin are included.

#### Transition

Transitions move state elements between `true`/`false` states. They can be triggered stochastically, using an equation which causes the transition to trigger when it evaluates to true, or with a timeout.

The following additional properties are available:

- `behavior`
  - `trigger` {"PROBABILITY"|"CONDITION"|"TIMEOUT"} [optional] – The trigger type for the transition.  
  - `value` {number|string} [optional] – The value or formula associated with the trigger. If the `trigger` is `TIMEOUT`, this is the time in the simulation time units until the trigger. If the `trigger` is `CONDITION`, this is a formula evaluated each time step which will trigger when it evaluates to `true`. If the `trigger` is `"PROBABILITY"`, this is the probability of a trigger happening within a single simulation time unit.

Note that transitions can only connect to and from states.

Additionally, the properties from the [connector](#connector) and [interactive-scalar](#interactive-scalar) mixins are included.

### Mixins

Some element types share common properties. The following mixins may be used with multiple elements. For example, the Flow, Link and Transition elements use the Connector mixin.

#### Connector

These additional properties apply to connector-type elements:

- `from` {string} [required] – The name of the start element. `null` is used to indicate the connector is not connected at the start. 
- `to` {string} [required] – The name of the end element. `null` is used to indicate the connector is not connected at the end. 
- `display`
  - `from_coordinates` {[number, number]} [optional - only specified if `from` is not connected] - [`x`, `y`] position for the end of the connector from the top left corner.
  - `to_coordinates` {[number, number]} [optional - only specified if `to` is not connected] - [`x`, `y`] position for the end of the connector from the top left corner.

#### Node

These additional properties apply to node-type elements:

- `display`
  - `coordinates` {[number, number]} [optional] – [`x`, `y`] position of the top left corner of the node from the top left corner of the canvas.
  - `size` {[number, number]} [optional] – The [`width`, `height`] of the element in pixels.

#### Interactive Scalar

For elements with a slider or numeric interactive UI:

- `display`
  - `interactive_min` {number} [optional] – Minimum user-selectable value.
  - `interactive_max` {number} [optional] – Maximum user-selectable value.

## Engines

ModelJSON does not attempt to standardize model formulas. Instead, an `engine` property is specified, indicating how formulas should be parsed and evaluated. Code to parse and evaluate formulas for each recognized `engine` is provided.

The `engine` governs the parsing and evaluation of the following properties: `value`, `initial_value`, and `units`.

Currently supported engines:

- **SIMULATION_PACKAGE**: [NPM `simulation` package](https://github.com/scottfr/simulation) format  
  - Code is available to parse and evaluate formulas in that repository.

Please submit a PR to add additional engines.


## Extending ModelJSON

In some cases, you may wish to use the ModelJSON format but need a property it does not currently support. Please open an issue to discuss extending the format.

You may also take advantage of the fact that standard properties and constant values will never start with an underscore (`_`). If you wish to add a custom property or constant value to your own ModelJSON objects, you may do so as long as you prepend it with an underscore (e.g., `_my_custom_property` or `"_CONSTANT_VALUE"`).


## Examples

The following examples illustrate the usage of various features of the ModelJSON format.

### Damped Pendulum Oscillations

```json
{
	"engine": "SIMULATION_PACKAGE",
	"name": "Damped pendulum",
	"description": "A simple damped pendulum model.",
	"simulation": {
		"algorithm": "RK4",
		"time_start": 0,
		"time_length": 10,
		"time_step": 0.1,
		"time_units": "SECONDS"
	},
	"elements": [
		{
			"type": "STOCK",
			"name": "Angle",
			"display": {
				"coordinates": [270, 70],
				"size": [120, 40]
			},
			"behavior": {
				"initial_value": 0.2,
				"non_negative": false,
				"units": "radians"
			}
		},
		{
			"type": "STOCK",
			"name": "Angular Velocity",
			"display": {
				"coordinates": [270, 180],
				"size": [120, 40]
			},
			"behavior": {
				"initial_value": 0,
				"non_negative": false,
				"units": "radians/second"
			}
		},
		{
			"type": "VARIABLE",
			"name": "Mass",
			"display": {
				"interactive": true,
				"interactive_min": 0.1,
				"interactive_max": 10,

				"coordinates": [30, 280],
				"size": [80, 40]
			},
			"behavior": {
				"value": 1,
				"units": "kilograms"
			}
		},
		{
			"type": "VARIABLE",
			"name": "Length",
			"display": {
				"interactive": true,
				"interactive_min": 0.1,
				"interactive_max": 10,

				"coordinates": [330, 280],
				"size": [80, 40]
			},
			"behavior": {
				"value": 1,
				"units": "meters"
			}
		},
		{
			"type": "VARIABLE",
			"name": "Gravity",
			"display": {
				"coordinates": [250, 360],
				"size": [80, 40]
			},
			"behavior": {
				"value": 9.81,
				"units": "meters / seconds^2"
			}
		},
		{
			"type": "VARIABLE",
			"name": "Damping Coefficient",
			"display": {
				"interactive": true,
				"interactive_min": 0,
				"interactive_max": 1,

				"coordinates": [70, 380],
				"size": [150, 40]
			},
			"behavior": {
				"value": 0.2,
				"units": "kilograms * meters^2 / seconds"
			}
		},
		{
			"type": "FLOW",
			"name": "Angle Rate",
			"from": null,
			"to": "Angle",
			"display": {
				"from_coordinates": [110, 90]
			},
			"behavior": {
				"value": "[Angular Velocity]",
				"units": "Radians / Second"
			}
		},
		{
			"type": "FLOW",
			"name": "Angular Acceleration",
			"from": null,
			"to": "Angular Velocity",
			"display": {
				"from_coordinates": [110, 200]
			},
			"behavior": {
				"value": "-([Damping Coefficient]/([Mass]*[Length]^2))*[Angular Velocity] - ([Gravity]/[Length])*sin([Angle]) * {1 radian}",
				"units": "Radians / Second^2"
			}
		},
		{
			"type": "LINK",
			"from": "Angular Velocity",
			"to": "Angle Rate"
		},
		{
			"type": "LINK",
			"from": "Angle",
			"to": "Angular Acceleration"
		},
		{
			"type": "LINK",
			"from": "Mass",
			"to": "Angular Acceleration"
		},
		{
			"type": "LINK",
			"from": "Length",
			"to": "Angular Acceleration"
		},
		{
			"type": "LINK",
			"from": "Gravity",
			"to": "Angular Acceleration"
		},
		{
			"type": "LINK",
			"from": "Damping Coefficient",
			"to": "Angular Acceleration"
		}
	],
	"visualizations": [
		{
			"type": "TIME_SERIES",
			"name": "Pendulum States",
			"elements": [
				"Angle",
				"Angular Velocity"
			]
		}
	]
}
```

### Population Growth with Carrying Capacity

```json
{
	"engine": "SIMULATION_PACKAGE",
	"name": "Population Growth",
	"simulation": {
		"algorithm": "RK4",
		"time_start": 0,
		"time_length": 20,
		"time_step": 0.2,
		"time_units": "YEARS"
	},
	"elements": [
		{
			"type": "STOCK",
			"name": "Population",
			"display": {
				"coordinates": [130, 210],
				"size": [100, 40]
			},
			"behavior": {
				"initial_value": 1,
				"non_negative": true
			}
		},
		{
			"type": "CONVERTER",
			"name": "Growth Rate",
			"display": {
				"coordinates": [260, 90],
				"size": [120, 50]
			},
			"behavior": {
				"input": "ELEMENT",
				"interpolation": "LINEAR",
				"data": [
					[0, 2],
					[1500, 1.07],
					[3990, 0.429],
					[6780, 0.125],
					[10000, 0]
				],
				"input_element": "Population"
			}
		},
		{
			"type": "FLOW",
			"name": "Flow",
			"from": null,
			"to": "Population",
			"display": {
				"from_coordinates": [180, 60]
			},
			"behavior": {
				"value": "[Population] * [Growth Rate]",
				"non_negative": true
			}
		},
		{
			"type": "LINK",
			"from": "Growth Rate",
			"to": "Flow"
		},
		{
			"type": "LINK",
			"from": "Population",
			"to": "Growth Rate"
		}
	],
	"visualizations": [
		{
			"type": "TIME_SERIES",
			"name": "Default Display",
			"elements": [
				"Population"
			]
		}
	]
}
```

### Predator/Prey Interactions

```json
{
	"engine": "SIMULATION_PACKAGE",
	"name": "Predator Prey Model",
	"description": "Version of the classic Lotka-Volterra model.",
	"simulation": {
		"algorithm": "RK4",
		"time_start": 2000,
		"time_length": 50,
		"time_step": 0.5,
		"time_units": "YEARS"
	},
	"elements": [
		{
			"type": "STOCK",
			"name": "Prey",
			"behavior": {
				"initial_value": 400,
				"non_negative": true,
				"units": "Prey"
			},
			"display": {
				"interactive": true,
				"interactive_min": 0,
				"interactive_max": 1000,

				"coordinates": [135, 214],
				"size": [100, 40]
			}
		},
		{
			"type": "VARIABLE",
			"name": "Prey Birth Rate",
			"display": {
				"coordinates": [30, 30],
				"size": [120, 50]
			},
			"behavior": {
				"value": 0.25,
				"units": "1 / Years"
			}
		},
		{
			"type": "VARIABLE",
			"name": "Prey Death Rate",
			"display": {
				"coordinates": [285, 371],
				"size": [150, 50]
			},
			"behavior": {
				"value": "{0.005 1/(Predators * Years)} * [Predators]",
				"units": "1 / Years"
			}
		},
		{
			"type": "VARIABLE",
			"name": "Predator Birth Rate",
			"display": {
				"coordinates": [300, 50],
				"size": [120, 50]
			},
			"behavior": {
				"value": "{0.0002 1 / (Prey * Years)} * [Prey]",
				"units": "1 / Years"
			}
		},
		{
			"type": "VARIABLE",
			"name": "Predator Death Rate",
			"display": {
				"coordinates": [637, 340],
				"size": [120, 50]
			},
			"behavior": {
				"value": 0.25,
				"units": "1 / Years"
			}
		},
		{
			"type": "STOCK",
			"name": "Predators",
			"behavior": {
				"initial_value": 20,
				"non_negative": true,
				"units": "Predators"
			},
			"display": {
				"interactive": true,
				"interactive_min": 0,
				"interactive_max": 100,

				"coordinates": [506, 212],
				"size": [100, 40]
			}
		},
		{
			"type": "FLOW",
			"name": "Prey Births",
			"from": null,
			"to": "Prey",
			"display": {
				"from_coordinates": [185, 62]
			},
			"behavior": {
				"value": "[Prey]*[Prey Birth Rate]",
				"non_negative": true,
				"units": "Prey / Years"
			}
		},
		{
			"type": "FLOW",
			"name": "Prey Deaths",
			"from": "Prey",
			"to": null,
			"display": {
				"to_coordinates": [185, 422]
			},
			"behavior": {
				"value": "[Prey]*[Prey Death Rate]",
				"non_negative": true,
				"units": "Prey / Years"
			}
		},
		{
			"type": "FLOW",
			"name": "Predator Births",
			"from": null,
			"to": "Predators",
			"display": {
				"from_coordinates": [556, 60]
			},
			"behavior": {
				"value": "[Predators]*[Predator Birth Rate]",
				"non_negative": true,
				"units": "Predators / Years"
			}
		},
		{
			"type": "FLOW",
			"name": "Predator Deaths",
			"from": "Predators",
			"to": null,
			"display": {
				"to_coordinates": [556, 420]
			},
			"behavior": {
				"value": "[Predator Death Rate]*[Predators]",
				"non_negative": true,
				"units": "Predators / Years"
			}
		},
		{
			"type": "LINK",
			"from": "Prey Birth Rate",
			"to": "Prey Births"
		},
		{
			"type": "LINK",
			"from": "Prey Death Rate",
			"to": "Prey Deaths"
		},
		{
			"type": "LINK",
			"from": "Predators",
			"to": "Prey Death Rate"
		},
		{
			"type": "LINK",
			"from": "Predator Death Rate",
			"to": "Predator Deaths"
		},
		{
			"type": "LINK",
			"from": "Predator Birth Rate",
			"to": "Predator Births"
		},
		{
			"type": "LINK",
			"from": "Prey",
			"to": "Predator Birth Rate"
		}
	],
	"visualizations": [
		{
			"type": "TIME_SERIES",
			"name": "Prey and Predators",
			"elements": [
				"Prey",
				"Predators"
			]
		},
		{
			"type": "TABLE",
			"name": "Details Table",
			"elements": [
				"Predators", "Prey", "Predator Deaths", "Predator Births", "Prey Births", "Prey Deaths"
			]
		}
	]
}
```

### SIR Disease Spread

```json
{
	"engine": "SIMULATION_PACKAGE",

	"simulation": {
		"algorithm": "RK1",
		"time_start": 0,
		"time_length": 20,
		"time_step": 0.2,
		"time_units": "WEEKS"
	},

	"elements": [
		{
			"type": "STOCK",
			"name": "S",
			"description": "Initial number of susceptible individuals.",
			"behavior": {
				"initial_value": 100
			},
			"display": {
                "interactive": true,
				"interactive_min": 0,
				"interactive_max": 100,

				"coordinates": [140, 60],
				"size": [100, 40]
			}
		},
		{
			"type": "STOCK",
			"name": "I",
			"description": "Initial number of infected individuals.",
			"behavior": {
				"initial_value": 3
			},
			"display": {
				"coordinates": [140, 190],
				"size": [100, 40],

				"interactive": true,
				"interactive_min": 0,
				"interactive_max": 100
			}
		},
		{
			"type": "STOCK",
			"name": "R",
			"description": "Initial number of recovered individuals.",
			"behavior": {
				"initial_value": 0
			},
			"display": {
				"coordinates": [140, 310],
				"size": [100, 40],

				"interactive": true,
				"interactive_min": 0,
				"interactive_max": 100
			}
		},
		{
			"type": "VARIABLE",
			"name": "γ",
			"behavior": {
				"value": 0.3
			},
			"display": {
				"coordinates": [50, 185],
				"size": [50, 50],

				"interactive": true,
				"interactive_min": 0,
				"interactive_max": 1
			}
		},
		{
			"type": "VARIABLE",
			"name": "β",
			"behavior": {
				"value": 0.01
			},
			"display": {
				"coordinates": [50, 40],
				"size": [50, 50],

				"interactive": true,
				"interactive_min": 0,
				"interactive_max": 0.05
			}
		},
		{
			"type": "FLOW",
			"name": "Infection",
			"from": "S",
			"to": "I",
			"behavior": {
				"value": "[β] * [S] * [I]"
			}
		},
		{
			"type": "FLOW",
			"name": "Recovery",
			"from": "I",
			"to": "R",
			"behavior": {
				"value": "[γ] * [I]"
			}
		},
		{
			"type": "LINK",
			"from": "β",
			"to": "Infection"
		},
		{
			"type": "LINK",
			"from": "γ",
			"to": "Recovery"
		}
	],
	"visualizations": [
		{
			"type": "TIME_SERIES",
			"name": "Disease Spread",
			"elements": ["I", "R", "S"]
		}
	]
}
```

### Balancing Population Growth Causal Loop Diagram

```json
{
	"elements": [
		{
			"type": "VARIABLE",
			"name": "Population",
			"display": {
				"coordinates": [140, 100],
				"size": [120, 50]
			}
		},
		{
			"type": "VARIABLE",
			"name": "Births",
			"display": {
				"coordinates": [90, 280],
				"size": [120, 50]
			}
		},
		{
			"type": "VARIABLE",
			"name": "Carrying Capacity",
			"display": {
				"coordinates": [480, 60],
				"size": [130, 50]
			}
		},
		{
			"type": "VARIABLE",
			"name": "Deaths",
			"display": {
				"coordinates": [430, 180],
				"size": [120, 50]
			}
		},
		{
			"type": "LINK",
			"from": "Population",
			"to": "Deaths",
			"behavior": {
				"polarity": "POSITIVE"
			}
		},
		{
			"type": "LINK",
			"from": "Population",
			"to": "Births",
			"behavior": {
				"polarity": "POSITIVE"
			}
		},
		{
			"type": "VARIABLE",
			"name": "Net Growth",
			"display": {
				"coordinates": [320, 320],
				"size": [120, 50]
			}
		},
		{
			"type": "LINK",
			"from": "Deaths",
			"to": "Net Growth",
			"behavior": {
				"polarity": "NEGATIVE"
			}
		},
		{
			"type": "LINK",
			"from": "Carrying Capacity",
			"to": "Deaths",
			"behavior": {
				"polarity": "NEGATIVE"
			}
		},
		{
			"type": "LINK",
			"from": "Births",
			"to": "Net Growth",
			"behavior": {
				"polarity": "POSITIVE"
			}
		},
		{
			"type": "LINK",
			"from": "Net Growth",
			"to": "Population",
			"behavior": {
				"polarity": "POSITIVE"
			}
		}
	]
}
```

### Bathtub Model

```json
{
	"engine": "SIMULATION_PACKAGE",
	"name": "Bathtub",
	"simulation": {
		"algorithm": "RK1",
		"time_start": 0,
		"time_length": 20,
		"time_step": 1,
		"time_units": "MINUTES"
	},
	"elements": [
		{
			"type": "STATE",
			"name": "Is Filling",
			"display": {
				"coordinates": [630, 110],
				"size": [100, 40]
			},
			"behavior": {
				"initial_value": true
			}
		},
		{
			"type": "STATE",
			"name": "Is Bathing",
			"display": {
				"coordinates": [630, 230],
				"size": [100, 40]
			},
			"behavior": {
				"initial_value": false
			}
		},
		{
			"type": "STATE",
			"name": "Is Draining",
			"display": {
				"coordinates": [630, 350],
				"size": [100, 40]
			},
			"behavior": {
				"initial_value": false
			}
		},
		{
			"type": "TRANSITION",
			"name": "Done Filling",
			"from": "Is Filling",
			"to": "Is Bathing",
			"behavior": {
				"trigger": "TIMEOUT",
				"value": 5
			}
		},
		{
			"type": "TRANSITION",
			"name": "Bath Over",
			"from": "Is Bathing",
			"to": "Is Draining",
			"behavior": {
				"trigger": "TIMEOUT",
				"value": 5
			}
		},
		{
			"type": "STOCK",
			"name": "Bathtub",
			"description": "The bathtub starts empty.",
			"display": {
				"coordinates": [390, 220],
				"size": [100, 40]
			},
			"behavior": {
				"initial_value": 0,
				"non_negative": true,
				"units": "Liters"
			}
		},
		{
			"type": "FLOW",
			"name": "Filling",
			"description": "Water fills at a rate of 10 liters per minute.",
			"from": null,
			"to": "Bathtub",
			"display": {
				"from_coordinates": [440, 90]
			},
			"behavior": {
				"value": "if [Is Filling] then\n  10\nend if",
				"non_negative": true,
				"units": "Liters/Minute"
			}
		},
		{
			"type": "FLOW",
			"name": "Draining",
			"description": "Water drains at a rate of 20% per minute.",
			"from": "Bathtub",
			"to": null,
			"display": {
				"to_coordinates": [440, 400]
			},
			"behavior": {
				"value": "if [Is Draining] then\n  {0.2 1/Minute} * [Bathtub]\nend if",
				"non_negative": true,
				"units": "Liters/Minute"
			}
		},
		{
			"type": "LINK",
			"from": "Is Filling",
			"to": "Filling"
		},
		{
			"type": "LINK",
			"from": "Is Draining",
			"to": "Draining"
		}
	],
	"visualizations": [
		{
			"type": "TIME_SERIES",
			"name": "Bathtub Volume",
			"elements": [
				"Bathtub"
			]
		},
		{
			"type": "TIME_SERIES",
			"name": "States",
			"elements": [
				"Is Filling",
				"Is Bathing",
				"Is Draining"
			]
		}
	]
}
```
