# RPP navigation architecture proposal

Draft for review. Updated 2026-11-17.

`rpp_navigation` will provide Nav2-style navigation behavior through RPP plugins,
with Cap'n Proto contracts defined before implementations. The core will work
without ROS. A single optional ROS wrapper will load the navigation script into
its process and translate topics, services, actions, and TF at the boundary.

## Library ownership

| Library | Responsibility |
| --- | --- |
| `rpp_navigation` | Navigation requests, behavior-tree orchestration, global planning, path smoothing, maps and costmaps, navigation behaviors and recovery policy |
| `rpp_localization` | Localization and state estimation, including pose, velocity, uncertainty, and validity information supplied to navigation and control |
| `rpp_control` | Control algorithms and execution, including path tracking and downstream robot control |
| ROS wrapper | ROS communication, message conversion, clock integration, and translation of external lifecycle and task requests |

Navigation will consume localization and control contracts rather than duplicate
their implementations. Its path-following task delegates tracking to
`rpp_control` and observes progress, failures, and completion.

The detailed ownership of goal and progress checkers, velocity smoothing, command
arbitration, collision monitoring, robot kinematics, hardware interfaces, and raw
odometry production remains a review item. Each responsibility must have one
owner even when multiple libraries consume it. Localization owns estimation;
this does not require every source of raw odometry to live in that library.

## Deployment and composition

The default deployment will run the navigation composition and the required
localization and control plugins in one process. One process may contain
multiple threads and execution rates. The composition must not require a ROS
executor or a network connection between its components.

```text
ROS topics, services, actions, TF
                 |
          Single ROS wrapper
                 |
      ROS-independent navigation API <--- Standalone application or simulator
                 |
         RPP navigation script
                 |
       Navigator and behavior tree
          /          |          \
   Planning      Environment    Control contracts
   plugins       plugins           |
                     |         rpp_control plugins
             rpp_localization      |
                contracts      Robot command output
```

Every replaceable component will have a typed RPP plugin contract, including
coordinators, algorithm implementations, costmap layers, and behavior-tree nodes.
Utility functions and data structures do not need artificial plugin boundaries.

The script will declare slots, select implementations through RPP assignments,
and wire execution. Replanning, recovery policy, and goal handling belong in
replaceable navigation plugins. Lifecycle and configuration will use existing
RPP mechanisms.

## Nav2 reference and compatibility

Nav2's separation of navigators, planners, controllers, smoothers, behaviors, and
environmental representations is the architectural reference. Its controller
responsibilities will be fulfilled through `rpp_control`.

Choose a pinned Nav2 release and commit before producing the parity inventory.
ROS 2 Lyrical is the proposed baseline because the existing RPP CI uses it.

Track behavioral parity separately from ROS API compatibility. Nav2-compatible
actions and services do not imply binary compatibility with Nav2 pluginlib
plugins, identical configuration files, or compatibility with every Nav2 client.

References:

- [Nav2 navigation servers](https://docs.nav2.org/rolling/getting_started/navigation_concepts/navigation_servers/)
- [Nav2 behavior-tree navigator](https://docs.nav2.org/rolling/configuration_and_development/configuration_guide/core_servers/configuring_bt_navigator/)

## Capn Proto contracts first

Define common data and interface semantics before implementing algorithms.
Cross-library types must have one canonical schema owner and be imported by
consumers. Review whether foundational geometry and time types belong in
`rpp_common`; navigation-specific data belongs in `rpp_navigation`.

| Contract group | Initial scope |
| --- | --- |
| Common data | Time, duration, clock identity, frames, poses, transforms, velocities, covariance, paths, footprints |
| Localization boundary | State estimates, uncertainty, timestamps, validity, frame relationships |
| Sensor and map inputs | Observations, occupancy maps, freshness, coordinate conventions |
| Environment | Transform access, map provider, global and local costmaps, layers and filters |
| Navigation algorithms | Planner and path smoother |
| Control boundary | Path or trajectory references, constraints, tracking feedback, cancellation, completion, faults |
| Navigation execution | Navigator, behavior-tree engine and nodes, recovery behaviors |
| Public API | Requests, task identities, feedback, results, cancellation, status, diagnostics |
| Later capabilities | Waypoint tasks, graph routes, docking |

Settle these semantics during contract review:

- SI units, coordinate conventions, frame identity, and transform timestamps.
- Monotonic deadlines versus simulation or observation timestamps, including
  simulation clock pauses and resets.
- Unknown cells, obstacle costs, footprints, and stale observations.
- Task acceptance, cancellation, preemption, deadlines, and terminal results.
- Error codes, partial results, and recovery eligibility.
- Buffer ownership, concurrency, snapshot revisions, and data lifetime.
- Schema evolution, stable ordinals, and lifecycle state transfer.
- Configuration ownership and validation across library boundaries.

Use distinct typed interfaces for planners, costmaps, navigators, and control
integration rather than a generic component operation interface.

## Task execution

Long-running navigation operations need an explicit task model independent of
ROS actions:

```text
submit -> accepted or rejected
accepted -> running -> succeeded, failed, or canceled
running -> feedback
cancel(taskId) -> cancellation acknowledged -> terminal result
```

Cancellation acknowledgement must be distinguished from completed stopping.
Specify how replacement goals supersede old tasks and reject stale feedback or
commands from an earlier task. The wrapper maps this model to ROS actions.

## Scheduling and shared data

Planning must not block control. ROS callbacks should enqueue inputs and
requests. Costmap readers should consume consistent, versioned snapshots while
updates proceed independently. Control and collision monitoring need bounded
work and explicit input freshness requirements.

Cap'n Proto defines interfaces and data; it does not require network RPC inside
the process. Prefer RPP's native in-process execution where supported. Evaluate
copying and binding overhead before choosing how maps and sensor buffers cross
interfaces; zero-copy behavior is not assumed.

Navigation, recoveries, and docking must share a command ownership policy.
Define stopping behavior for cancellation, deactivation, stale required inputs,
control faults, and shutdown. The placement and ordering of command arbitration,
velocity smoothing, and collision monitoring require review with `rpp_control`.
The final obstacle constraint must not be undone by downstream smoothing.

## Single ROS wrapper

The wrapper will load the same navigation composition used by standalone
applications. It will provide:

- Sensor, odometry, map, and TF input conversion where required by the composition.
- Navigation actions with feedback, cancellation, preemption, and results.
- Services for supported operations such as map loading and costmap clearing.
- Velocity, path, costmap, and diagnostic outputs.
- ROS clock integration and lifecycle translation.

ROS nodes, messages, TF2, and ROS action servers stay in this integration layer.
Navigation decisions and control algorithms remain in their respective core
libraries. The wrapper will not launch a second navigation process.

## Implementation milestones

1. **Architecture and parity inventory.** Pin the Nav2 reference, list its
   components and behaviors, and settle library ownership and compatibility goals.
2. **Contract review.** Define the shared types and initial navigation,
   localization, and control interfaces, including tasks, errors, timing, and
   ownership. Review signatures before adding navigation algorithms.
3. **ROS-free composition.** Load mock plugins through an RPP script. Establish
   lifecycle order, scheduling, cancellation, and command ownership. Make the
   current mandatory ament build optional and audit dependency ROS coupling.
4. **First navigation slice.** Integrate a static map, localization supplied by
   `rpp_localization`, a basic costmap, one planner, and path tracking supplied by
   `rpp_control`, with goal/progress handling and collision stopping.
5. **Nav2-style execution.** Add behavior-tree orchestration, replanning,
   recoveries, smoothing, and waypoint navigation.
6. **ROS wrapper.** Run the same composition against ROS inputs and expose the
   selected Nav2-compatible APIs.
7. **Broader parity.** Add algorithms, layers, filters, routing, and docking;
   extend localization and control integration through their owning libraries.

The proposed first navigation scope is 2D ground navigation. Localization is
supplied through `rpp_localization`; control is supplied through `rpp_control`.

## Validation plan

Future CI must cover a genuinely ROS-free environment and ROS integration with
colcon. A ROS container alone cannot demonstrate ROS independence.

Planned coverage includes schema compatibility, plugin substitution, task state
transitions, cancellation and preemption, lifecycle failures, clock behavior,
stale inputs, costmap consistency, command ownership, stopping behavior, and
deterministic navigation scenarios. Timing measurements must precede real-time
performance claims.

## Decisions for review

- Exact `rpp_control` scope, including path tracking and hardware integration.
- Ownership of checkers, smoothing, arbitration, and collision monitoring.
- Canonical shared schemas and dependency direction between libraries.
- Nav2 baseline, initial algorithm selection, and ROS API compatibility targets.
- BehaviorTree.CPP behind an RPP plugin versus another execution engine.
- Threading, update rates, buffer ownership, and task preemption policy.
- Optional ros2_control interoperability and its effect on deployment topology.
