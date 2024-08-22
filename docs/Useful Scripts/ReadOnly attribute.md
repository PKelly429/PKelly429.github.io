
## Attribute
```c#
public class ReadOnlyAttribute : PropertyAttribute  
{  
}
```

## Property Drawer

(Put this inside an Editor folder)

```c#
using UnityEditor;  
using UnityEditor.UIElements;  
using UnityEngine;  
using UnityEngine.UIElements;  
  
[CustomPropertyDrawer(typeof(ReadOnlyAttribute))]  
public class ReadOnlyDrawer : PropertyDrawer  
{  
    public override float GetPropertyHeight(SerializedProperty property,  
        GUIContent label)  
    {        
	    return EditorGUI.GetPropertyHeight(property, label, true);  
    }  
    public override VisualElement CreatePropertyGUI(SerializedProperty property)  
    {        
	    VisualElement container = new VisualElement();  
        PropertyField field = new PropertyField(property);  
        field.SetEnabled(false);  
        container.Add(field);  
        return container;  
    }  
    public override void OnGUI(Rect position,  
        SerializedProperty property,  
        GUIContent label)  
    {        
	    GUI.enabled = false;  
        EditorGUI.PropertyField(position, property, label, true);  
        GUI.enabled = true;  
    }
}
```