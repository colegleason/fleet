# Scheduling Services

## Making Scheduling Decisions

The current method of making service placement decisions is incredibly simple. 
When a user requests a given service be started in the system, a JobOffer is created.
Agents react to this JobOffer by deciding if they are able to run the referenced Job, and if so, submitting a JobBid back to the Engine.
The Engine simply accepts the first bid that is submitted for a given offer and commits the schedule change.

**NOTE:** The current approach of accepting the first bid is only temporary - the Engine will make an effort to fairly schedule across the entire schedule in the near future.

Read more about [fleet's architecture and data model](https://github.com/coreos/fleet/blob/master/Documentation/architecture.md).

## User-Defined Requirements

##### Schedule unit to specific machine

The `X-ConditionMachineID` option of a unit file causes the system to schedule a unit to a machine identified by the option's value.

The ID of each machine is currently published in the `MACHINE` column in the output of `fleetctl list-machines -l`.
One must use the entire ID when setting `X-ConditionMachineID` - the shortened ID returned by `fleetctl list-machines` without the `-l` flag is not acceptable.

fleet depends on its host to generate an identifier at `/etc/machine-id`, which is handled today by systemd.
Read more about machine IDs in the [official systemd documentation][machine-id].

[machine-id]: http://www.freedesktop.org/software/systemd/man/machine-id.html

##### Schedule unit to machine with specific metadata

The `X-ConditionMachineMetadata` option of a unit file allows you to set conditional metadata required for a machine to be elegible.

```
[X-Fleet]
X-ConditionMachineMetadata="region=us-east-1" "diskType=SSD"
```

This requires an eligible machine to have at least the `region` and `diskType` keys set accordingly. A single key may also be defined multiple timess, in which case only one of the conditions needs to be met:

```
[X-Fleet]
X-ConditionMachineMetadata=region=us-east-1
X-ConditionMachineMetadata=region=us-west-1
```

This would allow a machine to match just one of the provided values to be considered eligible to run.

A machine is not automatically configured with metadata.
A deployer may define machine metadata using the `metadata` [config option](https://github.com/coreos/fleet/blob/master/Documentation/configuration.md).

##### Schedule unit next to another unit

In order for a unit to be scheduled to the same machine as another unit, a unit file can define `X-ConditionMachineOf`.
The value of this option is the exact name of another unit in the system, which we'll call the target unit.

If the target unit is not found in the system, the follower unit will be considered unschedulable. 
Once the target unit is scheduled somewhere, the follower unit will be scheduled there as well.

Follower units will reschedule themselves around the cluster to ensure their `X-ConditionMachineOf` options are always fulfilled.

##### Schedule unit away from other unit(s)

The value of the `X-Conflicts` option is a [glob pattern](http://golang.org/pkg/path/#Match) defining which other units next to which a given unit must not be scheduled.

If a unit is scheduled to the system without an `X-Conflicts` option, other units' conflicts still take effect and prevent the new unit from being scheduled to machines where conflicts exist.

##### Dynamic requirements

fleet supports several systemd specifiers to allow requirements to be dynamically determined based on a Job's name. This means that the same unit can be used for multiple Jobs and the requirements are dynamically substituted when the Job is scheduled.

For example, a Job by the name `foo.service`, whose unit contains the following snippet:

```
[X-Fleet]
X-ConditionMachineOf=%p.socket
```

would result in an effective `X-ConditionMachineOf` of `foo.socket`. Using the same unit snippet with a Job called `bar.service`, on the other hand, would result in an effective `X-ConditionMachineOf` of `bar.socket`.

For more information on the available specifiers, see the [unit file configuration](Documentation/unit-files.md#systemd-specifiers) documentation.
