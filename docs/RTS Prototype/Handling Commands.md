I wanted to create a system that would allow units and buildings to define commands that would appear in the UI and would be easily configurable.

I want to be able to define common commands such as Move, Stop, Attack etc. Then allow units to decide which commands they have access to.

![[20240819212005.png]]

## How to Set Up a New Command
### Create a new command object

![[20240819213747.png]]

## Add to template

![[20240819213945.png]]

### Write code for units to implement event

```c#
public class DebugCommandListener : MonoBehaviour, ICommandRegister 
{  
    [SerializeField] Command command;  
    
    public void Register()  
    {        command.Register(RespondToDebugCommand);  
    }  
    public void Deregister()  
    {        command.Deregister(RespondToDebugCommand);  
    }
      
    public void RespondToDebugCommand()  
    {        Debug.Log("Unit has received debug command");  
    }
}
```

In this case the command scriptable object needs to be attached to the script. This could also be configured globally (e.g. In a static singleton with reference to all commands).

![[20240819214418.png]]

### Test
![[20240819-2050-57.0214862.mp4]]

## Implementation

###The Command Scriptable Object

```c#
[Serializable]  
public abstract class BaseCommand : ScriptableObject  
{  
    public Sprite image;  
    public Sprite imageHover;  
    public abstract void Execute();  
}
```

```c#
[CreateAssetMenu(menuName="Command/Command")]  
public class Command : BaseCommand  
{  
    private event ExecuteCommand OnExecute = () => { };  
  
    public delegate void ExecuteCommand();  
  
    public override void Execute()  
    {        OnExecute.Invoke();  
    }        public void Register(ExecuteCommand action) => OnExecute += action;  
    public void Deregister(ExecuteCommand action) => OnExecute -= action;  
}  
  
public abstract class Command<T> : BaseCommand  
{  
    private event ExecuteCommand OnExecute = (value) => { };  
  
    public delegate void ExecuteCommand(T value);  
  
    public void Execute(T value)  
    {        OnExecute.Invoke(value);  
    }        public void Register(ExecuteCommand action) => OnExecute += action;  
    public void Deregister(ExecuteCommand action) => OnExecute -= action;  
}
```


