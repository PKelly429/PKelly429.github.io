
Script for an attribute that automatically assigns a GUID to a string field on a Unity scriptable object. Works with duplicating objects using ctrl + d in the editor.

## Usage

```c#
public class SomeObject : ScriptableObject  
{
	[ScriptableObjectId] [ReadOnly] public string objectId;
}
```

## Code

```c#
using System;  
using UnityEngine;  
using Object = UnityEngine.Object;  
#if UNITY_EDITOR  
using UnityEditor;  
#endif  
  
[AttributeUsage(AttributeTargets.Field)]  
public class ScriptableObjectIdAttribute : PropertyAttribute  
{  
}  
  
#if UNITY_EDITOR  
[CustomPropertyDrawer(typeof(ScriptableObjectIdAttribute))]  
public class ScriptableObjectIdDrawer : PropertyDrawer  
{  
    public override void OnGUI(Rect position, SerializedProperty property, GUIContent label)  
    {        
	    GUI.enabled = false;  
  
        Object owner = property.serializedObject.targetObject;  
        // This is the unity managed GUID of the scriptable object, which is always unique  
        string unityManagedGuid = AssetDatabase.AssetPathToGUID(AssetDatabase.GetAssetPath(owner));  
  
        if (property.stringValue != unityManagedGuid)  
        {            
	        property.stringValue = unityManagedGuid;  
        }  
        EditorGUI.PropertyField(position, property, label, true);  
  
        GUI.enabled = true;  
    }
}  
#endif
```