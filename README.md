# SVLR Memory Tests

This document describes three memory-based tasks used to validate the SVLR world memory system with the Franka Panda robot.

The current design keeps the robot action interface simple. The LLM receives the current environment, the world memory state, and the list of available actions. It should use this information to generate memory-aware `pick_and_place` actions.

The main high-level action is:

```json
[
  {
    "action": "pick_and_place",
    "parameters": ["object_to_pick", "destination_object"]
  }
]
```

The LLM is expected to use the memory state to reason about hidden objects, stored positions, and previously manipulated objects. The Python layer then resolves symbolic targets such as `previous position` or `initial position` into robot-frame targets before execution.

---

## Test 1 — Hidden Object / Occlusion Memory

### Goal

Validate that SVLR can use memory to manipulate an object that is currently hidden under another object.

### Objects in the scene

- red cube
- yellow cube
- purple cube
- green cube

### Setup

All four cubes are initially visible on the table.

### Step 1 — Create the occlusion relation

User prompt:

```text
pick the purple cube and place it on green cube
```

Expected LLM output:

```json
[
  {
    "action": "pick_and_place",
    "parameters": ["purple cube", "green cube"]
  }
]
```

Expected system final command:

```text
Action 1: pick_and_place with params: [purple cube, green cube]
```

Expected memory update:

```text
purple cube on green cube
green cube under purple cube
green cube occluded_by purple cube
```

### Step 2 — Ask to move the hidden object

User prompt:

```text
pick the green cube and place it on red cube
```

At this point, the green cube is no longer detected by the VLM because it is hidden under the purple cube.

Detected objects:

```text
purple cube
red cube
yellow cube
```

Expected memory reasoning:

```text
The LLM reads the memory state and sees that the green cube is occluded by the purple cube.
It should not try to pick the hidden green cube directly.
It should first move the occluding purple cube back to its previous position.
Then it should complete the original request by moving the green cube to the red cube.
```

Expected LLM output:

```json
[
  {
    "action": "pick_and_place",
    "parameters": ["purple cube", "previous position"]
  },
  {
    "action": "pick_and_place",
    "parameters": ["green cube", "red cube"]
  }
]
```

Expected system final command:

```text
Action 1: pick_and_place with params: [purple cube, previous position]
Action 2: pick_and_place with params: [green cube, red cube]
```

### Result

The test succeeded.

The system correctly generated two actions for the second prompt:

```text
1. Move the occluding purple cube back to its previous position.
2. Pick the hidden green cube and place it on the red cube.
```

### Success condition

The test is successful if the LLM uses memory to generate two pick-and-place actions in the same user request:

```text
pick_and_place(purple cube, previous position)
pick_and_place(green cube, red cube)
```

The robot must first move the purple cube away, then move the green cube to the red cube.

### Possible failure conditions

- The robot tries to pick the hidden green cube directly.
- The robot only moves the purple cube and stops.
- The system generates only one action for the second prompt.
- The robot moves the wrong object.
- The green cube is not correctly remembered as being occluded by the purple cube.

---

## Test 2 — Return to Initial Position

### Goal

Validate that SVLR can remember the initial position of an object and use this memory to place the object back where it was first observed.

### Objects in the scene

- red cube
- yellow cube
- purple cube
- green cube

### Setup

All four cubes are initially visible on the table.

For this test, the object to move is the **red cube**, and the destination object is the **purple cube**.

### Step 1 — Move the object

User prompt:

```text
pick the red cube and place it on the purple cube
```

Expected LLM output:

```json
[
  {
    "action": "pick_and_place",
    "parameters": ["red cube", "purple cube"]
  }
]
```

Expected system final command:

```text
Action 1: pick_and_place with params: [red cube, purple cube]
```

Expected memory state after Step 1:

```text
Known objects:
- green cube: visible; initial position available
- purple cube: not visible; initial position available; under red cube; occluded_by red cube
- red cube: visible; initial position available; previous position available; on purple cube
- yellow cube: visible; initial position available

Relations:
- purple cube occluded_by red cube
- red cube on purple cube

Last successful pick_and_place:
- red cube -> purple cube
```

This shows that memory stores the relation created by the robot action: the red cube is now on the purple cube, and the purple cube is considered occluded by the red cube.

### Step 2 — Return to initial position

User prompt:

```text
pick the red cube and place it back to its initial position
```

Expected memory reasoning:

```text
The LLM reads the memory state and sees that the red cube has an initial position stored.
It should use the symbolic destination "initial position" instead of inventing coordinates.
The Python layer will resolve this symbolic destination into the stored robot-frame target.
```

Expected LLM output:

```json
[
  {
    "action": "pick_and_place",
    "parameters": ["red cube", "initial position"]
  }
]
```

Expected system final command:

```text
Action 1: pick_and_place with params: [red cube, initial position]
```

or internally:

```text
Action 1: pick_and_place with params: [red cube, red cube initial position target]
```

Expected memory behavior:

```text
SVLR resolves "initial position" using the stored initial_position_robot of the red cube.
The LLM does not output coordinates.
The Python memory layer converts the symbolic target into a robot-frame target.
```

Expected memory state after Step 2:

```text
Known objects:
- green cube: visible; initial position available
- purple cube: visible; initial position available
- red cube: visible; initial position available; previous position available
- yellow cube: visible; initial position available

Last successful pick_and_place:
- red cube -> initial position
```

The important point is that the red cube still has an initial position stored, and the system was able to use it as a symbolic destination.

### Result

The test succeeded if the robot picked the red cube from the purple cube and placed it back close to its original position.

### Success condition

The red cube returns close to the position where it was first observed before being moved.

### Possible failure conditions

- The robot places the red cube on the purple cube again.
- The robot picks the wrong object.
- The system cannot resolve `initial position`.
- The stored initial position is overwritten after the object moves.
- The robot uses the current position instead of the initial position.

---

## Test 3 — Matching Previous Object

### Goal

Validate that SVLR can use the last successful pick-and-place action as a reference and later select a new visible object that best matches the previously manipulated object.

### Objects in the first scene

- yellow cylinder
- red cube
- green cube
- purple cube

### Step 1 — Store the reference object through action history

User prompt:

```text
pick the yellow cylinder and place it on the red cube
```

Expected LLM output:

```json
[
  {
    "action": "pick_and_place",
    "parameters": ["yellow cylinder", "red cube"]
  }
]
```

Expected system final command:

```text
Action 1: pick_and_place with params: [yellow cylinder, red cube]
```

Expected memory update:

```text
World Memory State:
Known objects:
- yellow cylinder: visible; initial position available; previous position available; on red cube
- red cube: visible; initial position available; under yellow cylinder; occluded_by yellow cylinder
- green cube: visible; initial position available
- purple cube: visible; initial position available

Relations:
- yellow cylinder on red cube
- red cube under yellow cylinder
- red cube occluded_by yellow cylinder

Last successful pick_and_place:
- yellow cylinder -> red cube (type=cylinder; color=yellow)
```

---

### Objects in the second scene

The yellow cylinder is removed or hidden. A new cylinder is placed on the table.

- blue cylinder
- red cube
- green cube
- purple cube

### Step 2 — Ask for the matching object

User prompt:

```text
pick the object that matches the previous one and place it on the red cube
```

Expected memory reasoning:

```text
The LLM reads the memory state and sees that the previous manipulated object was the yellow cylinder.
It compares this previous object with the currently visible objects.
The blue cylinder is selected because it has the same object type: cylinder.
```

Expected LLM output:

```json
[
  {
    "action": "pick_and_place",
    "parameters": ["blue cylinder", "red cube"]
  }
]
```

Expected system final command:

```text
Action 1: pick_and_place with params: [blue cylinder, red cube]
```

Expected memory state:

```text
World Memory State:
Known objects:
- blue cylinder: visible; initial position available
- red cube: visible; initial position available
- green cube: visible; initial position available
- purple cube: visible; initial position available
- yellow cylinder: not visible; initial position available

Last successful pick_and_place:
- yellow cylinder -> red cube (type=cylinder; color=yellow)

Selected matching object:
- blue cylinder
```

### Result

The test succeeds if the robot selects the blue cylinder as the object matching the previously manipulated yellow cylinder, and places it on the red cube.

### Success condition

```text
The system resolves the task to:
pick_and_place(blue cylinder, red cube)
```

The robot selects the blue cylinder as the object matching the previously manipulated yellow cylinder, and places it on the red cube.

### Possible failure conditions

- The robot tries to pick the old yellow cylinder even though it is no longer visible.
- The robot selects a cube instead of the visible cylinder.
- The system cannot resolve the matching object.
- The robot forgets the previous reference object.
- The robot forgets the destination.

---
---

## Example World Memory JSON

The world memory stores the information needed by the LLM to reason about previous observations and actions.

Example after the robot placed the purple cube on the green cube:

```json
{
  "known_objects": {
    "red cube": {
      "visible": true,
      "status": "visible",
      "initial_position_robot": [0.52, -0.10, 0.17],
      "previous_position_robot": null,
      "current_position_robot": [0.52, -0.10, 0.17],
      "attributes": {
        "type": "cube",
        "color": "red"
      }
    },
    "yellow cube": {
      "visible": true,
      "status": "visible",
      "initial_position_robot": [0.51, 0.11, 0.17],
      "previous_position_robot": null,
      "current_position_robot": [0.51, 0.11, 0.17],
      "attributes": {
        "type": "cube",
        "color": "yellow"
      }
    },
    "purple cube": {
      "visible": true,
      "status": "visible",
      "initial_position_robot": [0.64, 0.10, 0.17],
      "previous_position_robot": [0.64, 0.10, 0.17],
      "current_position_robot": [0.64, -0.05, 0.22],
      "attributes": {
        "type": "cube",
        "color": "purple"
      },
      "supported_by": "green cube"
    },
    "green cube": {
      "visible": false,
      "status": "occluded",
      "initial_position_robot": [0.64, -0.05, 0.15],
      "previous_position_robot": null,
      "current_position_robot": [0.64, -0.05, 0.15],
      "attributes": {
        "type": "cube",
        "color": "green"
      },
      "occluded_by": "purple cube"
    }
  },
  "relations": [
    {
      "subject": "purple cube",
      "relation": "on",
      "object": "green cube"
    },
    {
      "subject": "green cube",
      "relation": "under",
      "object": "purple cube"
    },
    {
      "subject": "green cube",
      "relation": "occluded_by",
      "object": "purple cube"
    }
  ],
  "last_successful_pick_and_place": {
    "object": "purple cube",
    "destination": "green cube",
    "object_attributes": {
      "type": "cube",
      "color": "purple"
    }
  }
}
```

This memory contains the key information needed for the three tests:

```text
1. Hidden object:
   The green cube is not visible and is occluded_by the purple cube.

2. Return position:
   The purple cube has a previous_position_robot and an initial_position_robot.

3. Matching previous object:
   The last_successful_pick_and_place stores the last manipulated object and its attributes.
```

The LLM should not invent robot coordinates. It should only use object names and symbolic targets such as:

```text
previous position
initial position
matching object
```


