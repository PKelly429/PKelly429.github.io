# Handling Commands

In most RTS games, clicking on a unit will display commands unique to that unit.

I wanted to create a system that would allow me to define commands in a way that would be easily configurable.

![[20240819212005.png]]

# The Idea

A Command scriptable object is used to manage the visuals for each Command, as well as hold an event that units can subscribe to if they are ready to respond to that event.

This command object can then be bound to the UI.

Another scriptable object, the Command Template is used to define what Commands a unit has and their order in the UI.

A unit has a reference to a Command Template, along with scripts that subscribe to each Command and implement some code to execute when that Command is pressed.

![[20240820003741.png]]

# How Adding a New Command Works
## Create a new command object

Use the Create menu to create a new scriptable object and assign images for the button.

![[20240819213747.png]]

## Add to a Template

![[20240819213945.png]]

## Write Code for Units to Implement the Command

```c#
public class DebugCommandListener : MonoBehaviour, ICommandRegister 
{  
    [SerializeField] Command command;  
    
    public void Register()  
    {        
	    command.Register(RespondToDebugCommand);  
    }  
    public void Deregister()  
    {        
	    command.Deregister(RespondToDebugCommand);  
    }
      
    public void RespondToDebugCommand()  
    {        
	    Debug.Log("Unit has received debug command");  
    }
}
```

Attach script to our unit and configure the command scriptable object. This could also be configured globally (e.g. In a static singleton with reference to all commands).

![[20240819214418.png]]

## Finished Adding a New Command

This command could now be assigned to multiple units, and easily moved to a new position.

![[20240819235121.png]]

# Implementation

## The Command Scriptable Object

First there is the `BaseCommand` script that defines the images and an Execute Method. The Execute Method will be called by the UI when the button is clicked (or a hotkey pressed).

```c#
[Serializable]  
public abstract class BaseCommand : ScriptableObject  
{  
    public Sprite image;  
    public Sprite imageHover;  
    public abstract void Execute();  
}
```

Next there are two implementations, a regular command for simple actions (such as the Debug Command created here) and a generic command that can be used to create more complex commands requiring data. The Move command for example is a `Command<Vector3>`.

These scripts contain an event that can be subscribed to. When a unit is selected it subscribes to all commands it has been configured for. This allows all selected units to respond to a command.

The simple command immediately calls the event when pressed.

```c#
[CreateAssetMenu(menuName="Command/Command")]  
public class Command : BaseCommand  
{  
    private event ExecuteCommand OnExecute = () => { };  
  
    public delegate void ExecuteCommand();  
  
    public override void Execute()  
    {        
	    OnExecute.Invoke();  
    }        
    public void Register(ExecuteCommand action) => OnExecute += action;  
    public void Deregister(ExecuteCommand action) => OnExecute -= action;  
}  
  
public abstract class Command<T> : BaseCommand  
{  
    private event ExecuteCommand OnExecute = (value) => { };  
  
    public delegate void ExecuteCommand(T value);  

	// Does not implment Execute()
	// Any implementation still needs to implement this method
  
    public void ExecuteWithValue(T value)  
    {        
	    OnExecute.Invoke(value);  
    }        
    public void Register(ExecuteCommand action) => OnExecute += action;  
    public void Deregister(ExecuteCommand action) => OnExecute -= action;  
}
```


## Position Commands

The Move command is an instance of this Position Command, which requires a Vector3.

```c#
[CreateAssetMenu(menuName = "Command/Position Command")]  
public class Vector3Command : Command<Vector3>  
{  
    public override void Execute()  
    {        
	    SceneReferences.Instance.inputHandler.SetCommand(this);  
    }
}
```

Rather than immediately invoke the `OnExecute` event, it first makes a call to the Input Handler.
When the command is executed, the next click will order the unit to move to that position.

```c#
public void SetCommand(Vector3Command command)  
{  
    HasTargetCommand = true;  
    currentTargetCommand = command;  
}

...

// Called in Update if HasTargetCommand and mouse pressed this frame
private void HandlePositionCommand(Ray ray) 
{      
    int hits = Physics.RaycastNonAlloc(ray.origin, ray.direction, _hits, MaxDistance, groundLayers);  
  
    if (hits > 0)  
    {
	    currentTargetCommand.ExecuteWithValue(_hits[0].point);  
    }    
    
    HasTargetCommand = false;  
    currentTargetCommand = null;  
}

```


## Command Templates

This is attached to a script on each unit, and determines which commands should be shown when they are selected

```c#
[CreateAssetMenu(menuName = "Command/Command Template")]  
public class CommandTemplate : ScriptableObject 
{   
    public BaseCommand[] commandRow1 = new BaseCommand[5];  
    public BaseCommand[] commandRow2 = new BaseCommand[5];  
    public BaseCommand[] commandRow3 = new BaseCommand[5];  
}
```

## The Units

![[20240819235859.png]]

The units are configured with a number of scripts with the `ICommandRegister` interface. `MoveableEntity` implements the Move and Stop commands.

```c#
public interface ICommandRegister  
{  
    public void Register();  
    public void Deregister();  
}
```

The Selectable Object script gathers a reference to all scripts with this interface and when selected, they register to all commands.

```c#
[SerializeField] private CommandTemplate commands;

private ICommandRegister[] commandListeners;  
  
private void Start()  
{  
    commandListeners = GetComponents<ICommandRegister>();  
}  
  
private void RegisterCommands()  
{  
    SceneReferences.Instance.commandButtonGrid.Bind(commands);  
    foreach (var commandListener in commandListeners)  
    {        
		commandListener.Register();  
    }
}  
  
private void DeregisterCommands()  
{  
    foreach (var commandListener in commandListeners)  
    {        
	    commandListener.Deregister();  
    }
}

public void OnSelect()  
{  
    ...

	// This is a temporary measure to tell the UI to use this units commands
	// When multiple units can be selected at once this will need a system to
	// handle combining templates from all selected units
	SceneReferences.Instance.commandButtonGrid.Bind(commands);
	
    RegisterCommands();  
}  
  
public void OnDeselect()  
{  
    ...
    DeregisterCommands();  
}
```

## UI

Finally the UI has a list of button objects and iterates through each of them, binding them to the corresponding Command.

```c#
public void Bind(CommandTemplate unitCommands)  
{  
    if (unitCommands == null)  
    {        
	    Unbind();  
        return;  
    }    
    int next = 0;  
    for (int i = 0; i < unitCommands.commandRow1.Length; i++)  
    {        
	    commandBinders[next].Bind(unitCommands.commandRow1[i]);  
        next++;    
    }    
    for (int i = 0; i < unitCommands.commandRow2.Length; i++)  
    {        
	    commandBinders[next].Bind(unitCommands.commandRow2[i]);  
        next++;    
    }    
    for (int i = 0; i < unitCommands.commandRow3.Length; i++)  
    {        
	    commandBinders[next].Bind(unitCommands.commandRow3[i]);  
        next++;    
    }
}
```

```c#
public class CommandBinder : MonoBehaviour  
{  
    [SerializeField] private Button button;  
    [SerializeField] private Image icon;  
    [SerializeField] private HighlightOnHover hover; //switches icon when hovered 
    
    public void Bind(BaseCommand command)  
    {        
	    Unbind();  
        if (command == null) return;
           
        icon.enabled = true;  
        icon.sprite = command.image;
        
        button.enabled = true;  
        button.onClick.AddListener(command.Execute); 
        
        hover.SetSprites(command.image, command.imageHover);  
    }  
    
    protected void Unbind()  
    {        
	    icon.enabled = false;  
	    button.enabled = false;  
        button.onClick.RemoveAllListeners();  
    }
}
```

