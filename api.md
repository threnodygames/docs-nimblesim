Actions are atomic behaviour units that can be composed into sequences. They provide optional lifecycle hooks like:

`OnStart` - Runs when the action first becomes active in a sequence
`OnEnd` - Runs when the action completes & just before the sequence moves to the next step
`Update` - Runs every frame
`IsComplete` - Assessed after every update & tells the sequence when to move to the next step

`Get`

Get is the only method that you are required to implement. This is called whenever an action is reloaded (like in a looping sequence).

This method is important because it gives you the ability to decide how you want an action to work on reload. If you return a new instance, then all of the state you stored in the action will be reset. If you return the current reference, then state will be retained.
