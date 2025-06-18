This section covers **manager scripts**. In this project, manager scripts act as the control center for different high-level systems. Rather than handling logic in individual tools or UI components, managers oversee global coordination, such as which tool is active, how the UI behaves, or what object is selected. These scripts ensure the app behaves consistently and modularly, making it easier to scale and maintain. 

Having this hierarchy allows other scripts to do their job *without* having to worry about the big picture. For example, the `BrushTool` just draws, but how do we know when do enable it? Thats where manager scripts come into play. 

---

## **State Manager**
The `StateManager` is a centralized script that controls which drawing tool mode is active at any given time and manages the currently selected object in the AR space. It also handles user interactions like selecting, deleting, or duplicating objects. This script is key to keeping tool behavior modular and transitions clean between tools.

### **Singleton Pattern**
```csharp title="csharp"
public static StateManager Instance;

private void Awake()
{
    Instance = this;
}
```
As mentioned in the **Tool Scripts** section, `StateManager` follows a singleton pattern so other scripts can easily access it and ensures there is only one StateManager.

### **Mode Management**
The manager keeps track of which tool mode is active (brush, line, shape, erase, etc.) and enables/disables them accordingly.
```csharp title="csharp"
public void SwitchMode(ToolType newMode)
{
    DisableMode(mode);
    mode = newMode;
    EnableMode(mode);
}
```
This structure allows only one tool to be active at a time. It turns off the current tool, switches the mode, and activates the new one.