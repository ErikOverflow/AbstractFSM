# What is a Finite State Machine?

Think of an FSM (Finite State Machine) like a **big if statement** structure. "If this, then do that." "If this other thing, then do something else." It contains a finite list of states that each perform their own logic.

Each **State** contains:
* **Actions**
    * Unique logic performed while in the state.
* **Transitions**
    * How this state _transitions_ to other states.

Each **Transition** contains:
* **Decision**
    * Conditional statement.
* **True State**
    * State to transition to if the **Decision** is true.
* **False State**
    * State to transition to if the **Decision** is false.

Every cycle (in this example, the Update() cycle) the state machine performs all actions and transitions for the current state. This removes the need to check all complex logic every cycle, and instead allows the system to focus only on the current state's logic. For example, if a game only allows a character to jump when they are touching the ground, then there is no need to perform any jump logic when they are already airborn. In a finite state machine, the airborn state would not include the "Jump" action.

# Why use a Finite State Machine?

A state machine acts as a manager of your more granular components and can be a very useful tool in organizing/maintaining them. However, improper usage of a Finite State Machine can directly contradict singular responsibility and slog future development. Occasionally, the finite state machine removes some of the conditional logic responsibility from child scripts, which can impact their re-usability. The decision between putting extra logic in the finite state machine versus in the scripts is is controlling can be a tricky one, and ultimately there is no right answer. A rule of thumb I like to follow is: _If a script can't function without the state machine, then the logic needs to be re-balanced._ This doesn't mean that the script has to work immediately without the state machine, but it does mean that the script should be easily adapted/triggered without one. A state machine should only act as a macro-manager of the game state.

# Designing an abstracted FSM

Unity has a great tutorial on designing a [Finite State Machine with Pluggable AI](https://unity3d.com/learn/tutorials/topics/navigation/intro-and-session-goals), but Unity's approach is often-times too specific for the needs of some projects. 

#### For example: 
I have NPCs, Enemies, and the Player. My NPC cannot enter Enemy States, nor can my Player enter NPC states. I need to design state machines that don't allow the designer to accidentally assign states to entities that aren't permitted. Unity's tutorial shows a singular StateController that caches and maintains all Monobehaviours needed throughout the Actions, States, and Decisions made by the state machine, which works if you only have one type of entity. 

## Abstract implementation:
In order to make an extendable finite state machine, dependencies need to be established and organized to make abstraction possible. In Unity's implementation there is a direct dependency on the StateController in the Action, State, and Decision classes:

```
public abstract class Action : ScriptableObject 
{
    public abstract void Act (StateController controller);
}
```

```
public class State : ScriptableObject 
{
    public void UpdateState(StateController controller)
    {
        DoActions (controller);
        CheckTransitions (controller);
    }
...
```

```
public abstract class Decision : ScriptableObject
{
    public abstract bool Decide (StateController controller);
}
```

However, the StateController also has a dependency on State:
```
public class StateController : MonoBehaviour {

    public State currentState;
    public EnemyStats enemyStats;
    public Transform eyes;
    public State remainState;
...
```

In order to mitigate the complexity of this two-way dependency, the minimum required data needs to be passed to or managed in each script. Unity's implementation tracks Monobehaviours directly on the StateController, but I prefer to abstract out the State-specific data into a separate Monobehaviour, and use it to limit the scope of a generic for the StateController instead:

```
public abstract class CharStateData : MonoBehaviour { }
```

```
public abstract class StateController<T> : MonoBehaviour where T : CharStateData
{
    public T stateData;
...
```

Then we additionally implement Action/Decision similarly, again making sure to limit the scope of the generic:
```
public abstract class Action<T> : ScriptableObject where T : CharStateData
{
    public abstract void Act(T data);
}
```

```
public abstract class Decision<T> : ScriptableObject where T : CharStateData
{
    public abstract bool Decide(T data);
}
```

We then implement Transition/State as generic abstract classes. Unity's inspector does not expose generic fields, so we will set the fields implementations to be accessed via abstract method, rather than directly. We will leave direct field implementation for later derived classes, further wrapping them for Inspector serialization.

```
public abstract class Transition<T> where T : CharStateData
{
    public abstract Decision<T> GetDecision();
    public abstract State<T> GetTrueState();
    public abstract State<T> GetFalseState();
}
```

```
public abstract class State<T> : ScriptableObject where T : CharStateData
{
    public abstract Action<T>[] GetActions();
    public abstract Transition<T>[] GetTransitions();

    public void UpdateState(StateController<T> controller)
    {
...

    public void DoActions(T data)
    {
...
```

Now that we have an abstracted Finite State Machine, we need to define a custom CharStateData implementation, and create custom wrappers for our abstract classes so that they can be exposed via Unity Inspector.
