# Fog of War - Vision Calculations

To get our Fog of War working we need to keep track of what areas of the map the player is able to see. Once we know what grid cells are visible, we can write that data to a texture and use a shader to render the fog of war.

## Resources:

[implementing fog of war for rts game in unity](https://blog.gemserk.com/2018/08/27/implementing-fog-of-war-for-rts-games-in-unity-1-2/)

[riot games fog of war story](https://technology.riotgames.com/news/story-fog-and-war)

[visibility](https://www.redblobgames.com/articles/visibility/)

# Basic Solution

To do this we can keep count of how many units are able to see each cell.

The most simple solution is to iterate over every unit, remove them from each cell they can see, update their position and re-add them with the new position.

To find which cells a unit can see we can loop over a grid from -radius to +radius, centred around the unit. Then test if each cell overlaps with the vision circle.

'unitsWithVisionInCell' is a flattened 2d array where [x,y] = [y*width + x]


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
private void UpdateUnitVisibility(GridCell pos, int radius, bool increment)  
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

## Further Improvement to the Calculation

If we know a unit has moved 1 grid space to the right, we know that most of the cells they occupy will remain the same. If we can calculate only the cells that need to change, we can reduce the number of cells that need to be considered.

![[Pasted image 20240901162525.png]]

I have left this optimisation for now, as it makes the code quite a bit more complicated, and after moving things to the Job system, more optimisation wasn't required.
# Unity Job System

[Unity Job System](https://docs.unity3d.com/Manual/JobSystem.html)

Using the Job System will allow us to move our calculations off the main thread, essentially making it free so long as it finishes before we need to use the result.

The job system works best when used with the burst compiler, which does not allow managed types.

## The Job class

To work with the burst compiler our data needs to be converted to a structs and NativeArrays. 

The job will take in an array of Units that need to recalculate their vision, with the current and previous grid cell positions. A count is given to determine how far into the list to iterate.

The job outputs a flattened 2d array with the number of units in a given cell.

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
    [ReadOnly] public NativeArray<UnitVision> units;  
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

	private void UpdateUnitVisibility(GridCell pos, int radius, int radiusSquared bool increment)  
	{
		//...
	}
}
```

## The Schedular

To run the job, we need to create an instance of it and run '.Schedule()'. This returns a job handle, and at some point we need to call .'Complete()' on the handle to ensure the job is finished.

Our 'GridVisibilityManager' starts a job early in the frame, and then uses LateUpdate() to complete the job.

The data being used by the job, in this case the 'unitsToProcess' and 'unitsWithVisionInCell' cannot be used while the job is running. Attempting to do so will cause the job to be finished on the main thread. This will need to be considered when using jobs. 

One strategy if the data needs to be freely available would be to create a copy that can be accessed at any time, and when the job finishes the data can be synced.

In our case, we intend to use the data in another job, if this is the case we can use the previous job handle when scheduling the next job. This tells Unity that the next job depends on the previous one and the previous job must be completed first.


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

	private JobHandle _updateLitCellsJobHandle;  
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
	    if (!_updateLitCellsJobHandle.IsCompleted)  
	    {        
		    _updateLitCellsJobHandle.Complete();  
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
	        
	        _updateLitCellsJobHandle = job.Schedule();  
	        _jobRunning = true;  
	    }
	}  
	  
	private void LateUpdate()  
	{  
	    if (!_jobRunning) return;
	      
	    _updateLitCellsJobHandle.Complete();  
	    _jobRunning = false;  
	}

}
```

# The Burst Compiler

To use the burst compiler, we just need to add the attribute [BurstCompile].

To test the speed of using the Burst compiler I turned off the optimisation to skip units that hadn't moved, and completed the job immediately to more easily profile how long it was taking.

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

To make use of the unit visibility info, we want to write the data into a texture, which can then be used by a shader. Writing to the texture can also be done in a job by creating a NativeArray using '.GetRawTextureData()'.

On this same texture, we are using the red channel to determine which areas are buildable.

```c#
[BurstCompile]  
public struct UpdateGridTextureJob : IJobParallelFor  
{  
    [ReadOnly] public NativeArray<bool> blockedCells;  
    [ReadOnly] public NativeArray<uint> unitsWithVisionInCell;  
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


Because this job depends on the previous, we pass in the job handle '\_updateLitCellsJobHandle' when scheduling.

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