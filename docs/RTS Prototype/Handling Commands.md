I wanted to create a system that would allow units and buildings to define commands that would appear in the UI and would be easily configurable.

I want to be able to define common commands such as Move, Stop, Attack etc. Then allow units to decide which commands they have access to.

![[Pasted image 20240819212005.png]]

##The Command Scriptable Object

'''
[Serializable]  
public abstract class BaseCommand : ScriptableObject  
{  
    public Sprite image;  
    public Sprite imageHover;  
    public abstract void Execute();  
}
'''

