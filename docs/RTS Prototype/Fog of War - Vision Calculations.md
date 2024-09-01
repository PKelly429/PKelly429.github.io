To get our Fog of War working we need to keep track of what areas of the map the player is able to see. Once we know what grid cells are visible, we can write that data to a texture and use a shader to render the fog of war.

## Resources:

[implementing fog of war for rts game in unity](https://blog.gemserk.com/2018/08/27/implementing-fog-of-war-for-rts-games-in-unity-1-2/)

[riot games fog of war story](https://technology.riotgames.com/news/story-fog-and-war)

[visibility](https://www.redblobgames.com/articles/visibility/)

# Basic Solution

To do this we can keep count of how many units are able to see each cell.

The most simple solution is to iterate over every unit, remove them from each cell they can see, update their position and re-add them with the new position.

To find which cells a unit can see we can loop over a grid from -radius to +radius, centred around the unit. Then test if each cell overlaps with the vision circle.


``` c#
public void Update()
{
	foreach(unit in allUnits)
	{
		UpdateUnitVisibility(unit.gridPos, unit.radius, false);
		unit.gridPos = WorldToGridPos(unit.position);
		UpdateUnitVisibility(unit.gridPos, unit.radius, true);
	}

	UpdateTexture();
}

private void UpdateUnitVisibility(GridCell pos, int radius, bool add)
{
	for(int x = pos.x-radius, x < pos.x+radius; x++)
	{
		for(int y = pos.y-radius, y < pos.y+radius; y++)
		{
			GridCell cell = new GridCell(x, y);
			if(!cell.Overlaps(pos, radius)) continue;
			
			if(add)
			{
				unitsWithVisionInCell[y * GridBounds + x]++;
			}
			else
			{
				unitsWithVisionInCell[y * GridBounds + x]--;
			}
		}
	}
}
```


# Optimisations

## Only Update Units that Move

Easy enough, if a unit doesn't move skip recalculating it's visibility 

```c#
public void Update()
{
	foreach(unit in allUnits)
	{
		GridPos newPos = WorldToGridPos(unit.position);
		if(unit.gridPos.Equals(newPos)) continue;
		
		UpdateUnitVisibility(unit.gridPos, unit.radius, false);
		unit.gridPos = newPos;
		UpdateUnitVisibility(unit.gridPos, unit.radius, true);
	}

	UpdateTexture();
}
```

## Make the Grid Smaller

For a unit with a range of 20 units, if the Grid Size is 1x1 then to calculate it's vision requires 20x20 = 400 comparisons. If we change the Grid Size to 2x2, then we only need 10x10 = 100 comparisons.

So we perform 4x better if we reduce the Grid Size by 2x.

This is obviously a balance between gameplay and performance as having too big a grid size will make the fog of war blocky, and potentially affect other systems e.g. if the Building System uses the same grid.

## Optimising the Calculation

Rather than iterate over a radius x radius grid and calculate if each cell overlaps, we can do better. Instead we calculate the height of each column, and only iterate over cells that are inside the circle.

[Bresenham's Algorithm](http://members.chello.at/~easyfilter/bresenham.html)

[Wikipedia](https://en.wikipedia.org/wiki/Midpoint_circle_algorithm)

```c#
private void UpdateUnitVisibility(GridCell pos, int radius, bool add)  
{  
	int radiusSquared = radius * radius;
    for (int x = pos.x - radius; x < pos.x + radius; x++)  
    {  
        int xOffset = x - pos.x;  
        int height = (int)Math.Sqrt(radiusSquared - (xOffset * xOffset));  
  
        for (int y = pos.y - height; y < pos.y + height; y++)  
        {
  
            if (increment)  
            {
	            unitsWithVisionInCell[y * GridBounds + x]++;  
            }            
            else  
            {  
                unitsWithVisionInCell[y * GridBounds + x]--;     
            }  
        }
    }
}
```

# Further Improvement to the Calculation

If we know a unit has moved 1 grid space to the right, we know that most of the cells they occupy will remain the same. If we can calculate only the cells that need to change, we can reduce the number of cells that need to be considered.

![[Pasted image 20240901162525.png]]

# Unity Job System

[Unity Job System](https://docs.unity3d.com/Manual/JobSystem.html)

Using the Job System will allow us to move our calculations to a separate thread, essentially making it free so long as it finishes before we complete the job, in this case when we reach the end of the frame.

There is also the option of allowing it to run over multiple frames, and only restarting the job once it finishes, or having it run every x seconds.

## The Job class

```c#
public struct UnitVision  
{  
    public GridCell lastGridCell;  
    public GridCell newGridCell;  
    public int radius;  
}
```

``` c#
public struct UpdateUnitVisibilityJob : IJob  
{  
    [@ReadOnly] public NativeArray<UnitVision> units;  
    public NativeArray<uint> unitsWithVisionInCell;  
  
    public int UnitCount;  
    public int GridBounds;

	public void Execute()  
	{
		for (int i = 0; i < UnitCount; i++)  
		{  
		    UnitVision unit = units[i];
			int radiusSquared = radius * radius;
			UpdateUnitVisibility(unit.lastGridCell radius, radiusSquared, false);  
			UpdateUnitVisibility(unit.newGridCell, radius, radiusSquared, true);
		}
	}

	private void UpdateUnitVisibility(GridCell pos, int radius, int radiusSquared bool add)  
	{
		//...
	}
}
```

## The Schedular

```c#
public class GridVisibilityManager : MonoBehaviour
{
	public const int WorldSize = 1024;  
	public const int GridSize = 4;
	
	public const int GridBounds = WorldSize / GridSize;

	public const int MaxUnits = 100;

	private NativeArray<UnitVision> unitsToProcess;  
	private NativeArray<uint> unitsWithVisionInCell;

	private readonly HashSet<Unit> allUnits = new HashSet<Unit>();

	private JobHandle _jobHandle;  
	private bool _jobRunning;

	public void RegisterUnit(Unit unit)  
	{  
		// when units are added, the job should skip the decrement step
		// left out for brevity
	    allUnits.Add(unit);  
	}  
	  
	public void DeregisterUnit(Unit unit)  
	{  
		// when units are removed, the job should skip the increment step
		// left out for brevity
	    allUnits.Remove(unit);  
	}

	private void Start()  
	{  
	    unitsToProcess = new NativeArray<UnitVision>(MaxUnits, Allocator.Persistent);  
	    unitsWithVisionInCell = new NativeArray<uint>(GridBounds * GridBounds, Allocator.Persistent);  
	}
	  
	private void OnDestroy()  
	{  
	    if (!_jobHandle.IsCompleted)  
	    {        
		    _jobHandle.Complete();  
	    }    
	    
	    unitsToProcess.Dispose();  
	    unitsWithVisionInCell.Dispose();  
	}


	private void Update()  
	{  
	    int processQueueIndex = 0;  
	    foreach (var unit in allUnits)  
	    {
	        // skip if unit hasn't moved  
	        GridCell newPos = GridCell.FromWorldPos(unit.Position);  
	        if(unit.GridPosition.Equals(newPos)) continue; 
	         
	        unitsToProcess[processQueueIndex] = new UnitVision()  
	        {  
	            newGridCell = newPos,  
	            lastGridCell = obj.GridPosition,  
	            radius = obj.Radius
	        };  
	  
	        obj.GridPosition = newPos;  
	        processQueueIndex++;
	        
			if(processQueueIndex >= MaxUnits) break;  
	    }  
	    
	    if (processQueueIndex > 0)  
	    {        
		    var job = new UpdateUnitVisibilityJob()  
	        {  
	            units = unitsToProcess,  
	            unitsWithVisionInCell = unitsWithVisionInCell,  
	            UnitCount = processQueueIndex,  
	            GridBounds = GridBounds  
	        };  
	        
	        _jobHandle = job.Schedule();  
	        _jobRunning = true;  
	    }
	}  
	  
	private void LateUpdate()  
	{  
	    if (!_jobRunning) return;
	      
	    _jobHandle.Complete();  
	    _jobRunning = false;  
	}

}
```

# The Burst Compiler

To test the speed of using the Burst compiler I turned off the optimisation to skip units that hadn't moved, and completed the job immediately to profile how long it was taking.

I added around 20-30 units to the scene, and the above was taking around 0.5ms.

After adding the Burst compiler that went to 0.03ms, a 17x improvement.

```c#
[BurstCompile]  
public struct UpdateUnitVisibilityJob : IJob  
{
	 //...
}

```

Even without the further improvement of ignoring cells that a unit will remain in, the calculation is now having negligible impact on performance.


# Updating the Texture

```c#
[BurstCompile]  
public struct UpdateGridTextureJob : IJobParallelFor  
{  
    [@ReadOnly] public NativeArray<bool> blockedCells;  
    [@ReadOnly] public NativeArray<uint> unitsWithVisionInCell;  
    [WriteOnly] public NativeArray<Color32> texture;  
    
    public void Execute(int index)  
    {        
	    byte r = blockedCells[index] ? byte.MaxValue : byte.MinValue;  
        byte g = byte.MinValue;  
        byte b = unitsWithVisionInCell[index] > 0 ? byte.MaxValue : byte.MinValue; 
        byte a = byte.MaxValue;  
        
        texture[index] = new Color32(r, g, b, a);  
    }
}
```


```c#

[SerializeField] private Texture2D gridTexture;
private NativeArray<Color32> gridTextureData;

private void Start()
{
	// define the texture data
	gridTextureData = gridTexture.GetRawTextureData<Color32>();
}

//...

private void Update()
{
	// ...
	
	// Schedule job after the calculations are finished
	var updateTextureJob = new UpdateGridTextureJob()  
	{  
	    blockedCells = blockedCells,  
	    unitsWithVisionInCell = unitsWithVisionInCell,  
	    texture = gridTextureData  
	};  
	_updateTextureJobHandle = updateTextureJob.Schedule(GridBounds*GridBounds, 8, _updateLitCellsJobHandle);
}
//...

private void LateUpdate()
{
	// When job finished, apply the texture
	_updateTextureJobHandle.Complete();
	gridTexture.Apply();
}
```