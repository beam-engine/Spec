# Beam Specification
-------------------------

A **State Machine** is represented  by a JSON object.

## Example:
```json
{
  "Name": "SampleWorkflow",
  "Description": "A simple workflow",
  "InitState": "AppFormPost",
  "Async": false,
  "ResultVariable": "output",
  "States": {
    "AppFormPost": {
      "Type": "Task",
      "End": false
    }
  }
}
```
When this state machine is launched, the engine begins execution by identifying the "InitState". It executes that state, and then checks to see if the state is marked as an End State. If it is, then state machine terminates and returns a result. If the state is not an End State, the engine looks for a "Next" field to determine what state to run next; it repeats this process until it reaches an End State or a runtime error occurs.

Every state machine should contain *Name*, *Description*, *InitState* and *Async* flag. When *Async* is false then we have a *ResultVariable*

|    Object    |                                                                                                                                                                                         Description                                                                                                                                                                                          |
|:------------:|:--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------:|
|    Name	     |                                                                                                                                                                            Defines the name of the state machine                                                                                                                                                                             |
| Description	 |                                                                                                                                                                       Human-readable description of the state machine                                                                                                                                                                        |
|  InitState	  |                                                                                                                       First state to execute, whose value must exactly match one of names of the "States" fields. The state machine starts running at the named state                                                                                                                        |
|    Async	    |                                                                                                                   Type of execution of the state machine. When async is enabled the caller will not wait for the workflow to complete. The response is shared via callback                                                                                                                   |
|    ResultVariable	    |                                                                                                                             This is only applicable for sync state machine. On successful execution we return the value of this variable mentioned to the caller                                                                                                                             |
|    Type	     |                                                                                                                                                  Represents the state type, currently we have 4 states *Task, Condition, Wait and Parallel*                                                                                                                                                  |
|    Next	     | Transitions link states together, defining the control flow for the state machine. After executing a non-terminal state, the state machine follows a transition to the next state which is specified through the state's "Next" field. All non-terminal states MUST have a "Next" field. |
| Conditions	  |                                                                                                                              Defines array of rules to match, When the corresponding rule is matched it takes the "Next" field value to move to the next state                                                                                                                               |
|   Simple	    |                                                                                                                                                      One of the expression type, which contains only one expression or one rule match.                                                                                                                                                       |
|     And	     |                                                                                                                                                                       With And expression we can match multiple rules                                                                                                                                                                        |
|     Or	      |                                                                                                                        With Or expression also we can match multiple rules. This is similar to "Simple" rule only but we can have multiple rules in "Or" expression.                                                                                                                         |
|  Variable	   |                                                                                                                                                                              Variable to run the matching logic                                                                                                                                                                              |
|  MatchType	  |                                                                                                                                  With Represents what kind of matching logic to execute, currently supports two types "StringEquals" and "StringNotEquals"                                                                                                                                   |
| MatchValue	  |                                                                                                                                                                                    Result value to match                                                                                                                                                                                     |

## Concepts

### States
States are represented as fields of the top-level "States" object

- All states must have a "Type" field.
- All states must have a field named "End" whose value must be boolean. The term "Terminal State" means a state with { "End": true }.

#### Task State
The task state is identified by Type("Type": "Task") is responsible for executing the business logic of the workflow.
```json
{
  "Name": "SampleWorkflow",
  "Description": "A simple workflow",
  "InitState": "AppFormPost",
  "Async": false,
  "ResultVariable": "output",
  "States": {
    "AppFormPost": {
      "Type": "Task",
      "Next": "CreditPolicyCheck",
      "End": false
    }
  }
}
```


#### Condition State
A Condition State ("Type":"Condition") adds branching logic to a state machine. A Condition State must have a "Conditions" field whose value is a non-empty array. Each element of the array must be a JSON object and is called a Condition Rule. Each JSON object should contain "Variable", "MatchType" and "MatchValue".

Currently, the "MatchType" should be either "StringEquals" or "StringNotEquals". Boolean is not support at this point of time. Also it support multiple types of conditions such as "And", "Or", "Simple".

```json
{
  "Name": "SampleWorkflow",
  "Description": "A simple workflow",
  "InitState": "AppFormPost",
  "Async": false,
  "ResultVariable": "output",
  "States": {
    "AppFormPost": {
      "Type": "Task",
      "Next": "AppFormCondition",
      "End": false
    },
    "AppFormCondition": {
      "Type": "Condition",
      "End": false,
      "Conditions": [
        {
          "Simple": [
            {
              "Variable": "appFormPostStatus",
              "MatchType": "StringEquals",
              "MatchValue": "success"
            }
          ],
          "Next": "CreditPolicyCheck"
        }
      ]
    }
  }
}
```

```json
{
  "Name": "SampleWorkflow",
  "Description": "A simple workflow",
  "InitState": "AppFormPost",
  "Async": false,
  "ResultVariable": "output",
  "States": {
    "AppFormPost": {
      "Type": "Task",
      "Next": "AppFormCondition",
      "End": false
    },
    "AppFormCondition": {
      "Type": "Condition",
      "End": false,
      "Conditions": [
        {
          "And": [
            {
              "Variable": "validationResult",
              "MatchType": "StringEquals",
              "MatchValue": "passed"
            },
            {
              "Variable": "appFormPost",
              "MatchType": "StringEquals",
              "MatchValue": "success"
            }
          ],
          "Next": "CreditPolicyCheck"
        }
      ]
    }
  }
}
```
#### Wait State
A Wait State ("Type":"Wait") causes the engine to suspend the state machine from continuing until it resumes.
<p>
The Wait state is only supported in async workflow(i.e. Async = true)
</p> 

#### Parallel State
The Parallel State ("Type":"Parallel") causes parallel execution of "Branches". A Parallel State uses the engine to execute each branch starting with the state named in its "InitState" field, as concurrently as possible, and wait until each branch terminates before processing the Parallel State's "Next" field.


A Parallel State must contain a field named "Branches" which is an array whose elements must be objects. Each object must contain fields named "States" and "InitState" whose meanings are exactly like those in the top level of a State Machine.

A state in a Parallel State branch "States" field must not have a "Next" field that targets a field outside that "States" field. A state must not have a "Next" field which matches a state name inside a Parallel State branch’s "States" field unless it is also inside the same "States" field.

Put another way, states in a branch’s "States" field can transition only to each other, and no state outside that "States" field can transition into it.

If any branch fails, due to an unhandled error or by transitioning to a Fail State, the entire Parallel State is considered to have failed and all the branches are terminated. If the error is not handled by the Parallel State, the engine should terminate the execution with an error.

```json
{
  "Name": "ConditionWorkflow",
  "Description": "A simple workflow",
  "InitState": "T1",
  "Async": false,
  "ResultVariable": "output",
  "States": {
    "T1": {
      "Type": "Task",
      "Next": "T1Condition",
      "End": false
    },
    "T1Condition": {
      "Type": "Condition",
      "End": false,
      "Conditions": [
        {
          "Simple": [
            {
              "Variable": "T1Status",
              "MatchType": "StringEquals",
              "MatchValue": "success"
            }
          ],
          "Next": "TParallel"
        }
      ]
    },
    "TParallel": {
      "Type": "Parallel",
      "Branches": [
        {
          "InitState": "T2",
          "Name": "Credit Policy",
          "States": {
            "T2": {
              "Type": "Task",
              "Next": "",
              "End": true
            }
          }
        },
        {
          "InitState": "T3",
          "Name": "GC Algo",
          "States": {
            "T3": {
              "Type": "Task",
              "Next": "",
              "End": true
            }
          }
        }
      ],
      "Next": "T3Condition",
      "End": false
    },
    "T3Condition": {
      "Type": "Condition",
      "End": false,
      "Conditions": [
        {
          "And": [
            {
              "Variable": "T2Status",
              "MatchType": "StringEquals",
              "MatchValue": "success"
            },
            {
              "Variable": "T3Status",
              "MatchType": "StringEquals",
              "MatchValue": "success"
            }
          ],
          "Next": "T4"
          }
      ]
    },
    "T4": {
      "Type": "Task",
      "Next": "T4Condition",
      "End": false
    },
    "T4Condition": {
      "Type": "Condition",
      "End": false,
      "Conditions": [
        {
          "Simple": [
            {
              "Variable": "T4Status",
              "MatchType": "StringEquals",
              "MatchValue": "success"
            }
          ],
          "Next": "T5"
        }
      ]
    },
    "T5": {
      "Type": "Task",
      "Next": null,
      "End": true
    }
  }
}
```
