# articles you shouldn't care about

this is a repository where i keep all the nonsensical and technical stuff that goes on in my mind. welcome to my brain dump and hope you like it here.

## Dependencies

Requires a brain.

## Build Instructions

```
brain mastercontrol load_unsafe ./voxell.brain -v --no-shutdown > ./temp.brain
brain mastercontrol flush
brain nextwakeup run "set-current-brain $(cat ./temp.brain | grep -oP '(?<=brain: ).*')"
brain mastercontrol givecontrol
``` 