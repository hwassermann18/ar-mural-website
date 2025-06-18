This sections describes all the scripts in the "Tools" folder. Each tool enables a different type of interaction in AR. Tools are activated by pinch gestures and share a common interface. If you have any questions that arise and aren't answered in that section, feel free to search the terms of your question in the search tab on the top right to see where they come up, or check out the footnotes at the bottom.
## **General Script Layout**
We'll cover the basic methods and functions present in all these tools first. For our sake we will use `BrushTool.cs` as our code snippets, which will be slightly different in other tools.
### 1. **Core Variables**
```csharp title="csharp"
public static BrushTool Instance { get; private set; }
[SerializeField] private float granularity;
private bool activated;
private bool isLeftHand;
private float distanceLeft;
private Vector3? oldPos;
private List<Vector3> meshPoints;
public Transform brushParent;
```

| Variable      | Type                | Description |
|---------------|---------------------|-------------|
| `Instance`    | `static BrushTool`  | Singleton reference to the current `BrushTool` instance for global access. |
| `granularity` | `float` *(Serialized)*[^1] | Min distance (in meters) the hand must move before adding a new point; controls stroke smoothness. |
| `activated`   | `bool`              | Whether the tool is subscribed to hand input events (prevents re-activation). |
| `isLeftHand`  | `bool`              | Indicates which hand (left/right) is being used for drawing. |
| `distanceLeft`| `float`             | How far the hand must move to place the next point, based on `granularity`. |
| `oldPos`      | `Vector3?`[^2]          | Last position where a point was placed; used to measure distance traveled. |
| `meshPoints`  | `List<Vector3>`     | List of all recorded 3D points for the current stroke. |
| `brushParent` | `Transform` [^3]        | Parent GameObject for all stroke objects; keeps the scene hierarchy organized. |

[^1]: A serialized private variable shows it's value on the Unity Inspector. You can easily change the value like public variable. But nothign can access this value from another script or places.
[^2]: Vector3 is a data structure used to represent a point in 3D space using 3 float values: x, y, and z. The question mark after indicates that the variable can have a value of null.
[^3]: One of the most fundamental components in Unity game design. Its a component that defines an objects **position, rotation, and scale**.  It also manages parent-child hierarchy, where children move, scale, and rotate relative to their parent.

---

### 2. **Singleton Pattern**
A singleton pattern ensures only one tool instance exists at runtime. This is common in Unity for tools or managers that need global access. 

```csharp title="csharp"
public static BrushTool Instance { get; private set; }
private void Awake()
{
    // Enforce a singleton state pattern
    if (Instance == null)
    {
        Instance = this;
    }
    else
    {
        Destroy(this);
        return;
    }
}
```
These lines of code are at the beginning of all the tool scripts and confirm that there is only one instance of the class.

---

### 3. **Tool Initialization**
```csharp title="csharp" linenums="41"
if (enabled == true)
{
    enabled = false;
}

brushParent = new GameObject("Brush Tool").transform;
brushParent.position = Vector3.zero;
```
The code first disables the tool by default. Line 47 creates a new empty GameObject and names it `"Brush Tool"`. It then stores a reference to its Transform in the variable brushParent. They then set the position for `brushParent` to (0, 0, 0).

Why do we create a parent object for this? Since the Transform class supports parent-child hierarchy, every time the BrushTool is activated, strokes are created and added as children of a shared parent object named `"Brush Tool"`. We will show this happening later in the code in the line: `obj.transform.SetParent(brushParent);`.

---

### 4. **Activate and Deactivate**
Each tool script includes `Activate()` and `Deactivate()` methods to manage input subscription:
```csharp title="csharp"
public void Activate()
{
    if (activated) return;

    activated = true;
    PinchManager.Instance.OnHandPinched += StartTool;
    PinchManager.Instance.OnHandReleased += StopTool;
}
```
```csharp title="csharp"
public void Deactivate()
{
    if (!activated) return;

    activated = false;
    PinchManager.Instance.OnHandPinched -= StartTool;
    PinchManager.Instance.OnHandReleased -= StopTool;
    
    // Clean up current stroke if mid-draw
    if (enabled)
    {
        enabled = false;
        oldPos = null;
        meshPoints = null;
        PreviewManager.Instance.StopPreview();
    }
}
```
`Activate()` connects the tool to gesture events (`OnHandPinched`, `OnHandReleased`) via `PinchManager`, allowing it to respond to hand input. `Deactivate()` unsubscribes from those events and resets the tool state.

These methods are **not called automatically**. They are triggered by a **manager script** called `StateManager`, which ensures that only one tool is active at a time.

---

### 5. **StartTool**
```csharp title="csharp"
public void StartTool(bool isLeftHand)
{
    if (enabled == true)
    {
        return;
    }

    enabled = true;
    this.isLeftHand = isLeftHand; 
    distanceLeft = granularity;
    Vector3 startPos = isLeftHand ? PinchManager.Instance.LeftPosition :
                                    PinchManager.Instance.RightPosition;
    oldPos = startPos;
    meshPoints = new List<Vector3> { startPos };
    PreviewManager.Instance.StartPreview(startPos);
}
```
**Called when the user pinches** to begin an interaction.

- Makes the tool active and ready to track hand movement(left or right)
- Captures the current hand position as the stroke’s starting point
- Initializes distance tracking for smooth point placement
- Prepares the meshPoints list that will define the stroke path
- Starts the live preview to show what the user is drawing in real time

---

### 6. **Update**
```csharp title="csharp"
private void Update()
{
    Vector3 newPos = isLeftHand ? PinchManager.Instance.LeftPosition : PinchManager.Instance.RightPosition;

    if (!MathUtils.Vector3Equals(newPos, oldPos.Value))
    {
        PreviewManager.Instance.UpdatePreview(newPos);
    }

    float distanceTraveled = (newPos - oldPos.Value).magnitude;

    if (distanceTraveled >= distanceLeft)
    {
        ...
        Vector3 meshPoint = MathUtils.TravelAcross(oldPos.Value, newPos, totalDistanceTracked);
        meshPoints.Add(meshPoint);
        PreviewManager.Instance.AddPreview(meshPoint);
    }
    else
    {
        distanceLeft -= distanceTraveled;
    }

    oldPos = newPos;
}
```
**Runs every frame** while the tool is active.

- Continuously tracks hand movement.
- Updates the preview visuals based on new hand position.
- Some tools (like BrushTool) dynamically add points to a list; others (like ShapeTool or TextTool) update size or orientation in real time.

You may notice that no other method calls update(). This gets to the core of how Unity works behind the scenes. Unity automatically calls `Update()` every frame on any `MonoBehaviour`[^4] script that is both active and [^5]. In `BrushTool`, `Update()` begins running when `StartTool()` sets `enabled = true`, and stops when `StopTool()` sets it to `false`.

[^4]: A `MonoBehaviour` script is the **base class** that all scripts inherit from if you want them to: be attached to GameObjects and respond to Unity's **lifecycle events** like `Start()`, `Update()`, `Awake()`, etc. In the specific case of `BrushTool`, it becomes a unity component you can attach to objects in the scene. 
[^5]:`enabled` is a built-in property from `MonoBehaviour` that determines whether Unity will call runtime methods like `Update()` for this script. In tool scripts like `BrushTool`, `enabled` is used to: Start tracking input when `StartTool()` is called (`enabled = true`) and Stop tracking and finalizing the stroke when `StopTool()` is called (`enabled = false`). You don’t need to declare this variable — it comes from Unity’s MonoBehaviour system.

---

### 7. **StopTool**
```csharp title="csharp"
public async void StopTool(bool isLeftHand)
{
    if (!enabled || isLeftHand != this.isLeftHand) return;

    Vector3 endPos = oldPos.Value;
    meshPoints.Add(endPos);

    GameObject obj = BrushRenderer.Instance.GenerateMesh(true, meshPoints);
    ...
    if (obj != null)
    {
        obj.name = "Brush";
        obj.transform.SetParent(brushParent);
        obj.layer = 7;
        MeshOutline outline = obj.AddComponent<MeshOutline>();
        outline.enabled = false;
        ...
    }

    PreviewManager.Instance.StopPreview();
    enabled = false;
    oldPos = null;
    meshPoints = null;

    if (c != null && ClientManager.Instance.connected)
    {
        await ClientManager.Instance.SendCommand(c);
    }

    if (ClientManager.Instance.connected)
    {
        Destroy(obj);
    }
}
```
**Called when the user releases the pinch** to finalize the interaction.

- Ends the preview and commits the final object to the scene.
- Generates the actual mesh or 3D object (e.g., stroke, shape, text).
- Applies metadata like color, material, and layer.
- Sends the result to the server via `ClientManager` if networking is enabled.
- Cleans up internal state and deactivates the tool.

---

### 8. **Regenerate** 
```csharp title="csharp"
public GameObject Regenerate(BrushContainer c)
{
    GameObject newObj = BrushRenderer.Instance.GenerateMesh(...);

    if (newObj != null)
    {
        newObj.transform.SetParent(brushParent);
        newObj.name = "Brush";
        ...
    }

    return newObj;
}
```
**Used to recreate a saved object** from serialized data.
Each tool includes a `Regenerate()` method that rebuilds a previously created object from its saved data (like a `BrushContainer`, `LineContainer`, etc.).

This method is used to:

- Reconstruct objects when loading a scene
- Display objects sent over the network from other users
- Recreate objects during undo/redo
- Maintain visual and structural consistency between sessions

It creates a new GameObject, configures it based on saved metadata (e.g. mesh shape, color, size), and adds necessary components (like outlines or colliders). This method will become more relevant when we begin to look at how we are saving data and running a server. 

---
## **Tool Script Specifics**
This section lists relevant implementation notes specific to each tool file.

---

### **Brush Tool**
BrushTool enables freeform 3D drawing using continuous hand movement. It generates a smoothed spline mesh based on tracked hand positions.

**Key behaviors and implementation details:**

- `granularity` controls how often new points are added. Lower values result in smoother strokes but impact performance and network bandwidth.
- Points are collected and stored in `meshPoints` in `Update()`, and only added when the user moves beyond `distanceLeft`.
- When a stroke is finalized in `StopTool()`, it is:
    - Passed to `BrushRenderer` to generate a mesh.
    - Serialized into a `BrushContainer`.
    - Sent to the server (if connected).
    - Destroyed locally (the server version will be redrawn).
- The object is named `"Brush"` and added to the `"Brush Tool"` parent object.
- Uses layer 7 for brush strokes (`Settings.LAYER_BRUSH`).
- `Regenerate()` is used to rebuild the stroke from a `BrushContainer`, typically on scene load or from server data.

We will elaborate more on `BrushRenderer` and `BrushContainer` later.

---

### **Line Tool**
LineTool is nearly identical in structure to BrushTool, but is used to create **a single straight line** between pinch start and release points.

**Key differences from BrushTool:**

- The `meshPoints` list always contains just two positions: start and end.
- No distance tracking or granularity needed — it simply connects the two points.
- The mesh is generated without smoothing (`loop: false`, `smoothnessThreshold: -1`).
- The object is named `"Line"` and added to the `"Line Tool"` parent object.
- Uses a different layer (typically `Settings.LAYER_LINE`).
- Regeneration follows the same container-based pattern (`LineContainer`), and restores the line’s mesh and transform.

---

### **Text Tool**
TextTool allows users to place a block of 3D text in space by pinching and dragging to set size and position.

**Key implementation details:**

- Text content is pulled from `ButtonManager.Instance.textToolField.text`, or falls back to `defaultText`.
- Text is rendered using a `TextMeshPro` component, which supports dynamic font sizing.
- A transparent outline is created using a child GameObject from `OutlineGenerator.GenerateOutline()`.
- The final text is wrapped in a box collider and registered as interactable.
- Objects are given layer 10 (`Settings.LAYER_TEXT`) and parented under `"Text Tool"`.

---

### **Image Tool**
ImageTool fetches 2D images from Unsplash using their API and lets users place them in the scene.

**Unique logic:**

- Uses `UnityWebRequest` to retrieve image data asynchronously.
- Displays a scrollable UI list of thumbnails, each of which can be clicked to place an image.
- Placed images are interactive, with `ObjectManipulator` and drag-handles enabled.
- Textures are applied directly to a `RawImage` or Unity `Renderer`.

**Note:** This tool does not use pinch gestures or `StartTool/StopTool` like the others. It’s UI-driven.

---

### **Object Tool**
ObjectTool lets users choose from a predefined list of 3D prefabs and place them into the AR scene.

**Implementation notes:**

- Prefabs are listed in the `localObjects` list and displayed using a scroll-down UI menu.
- Selecting an object instantiates it in front of the user and enables manipulation (`ObjectManipulator`, `BoundsControl`).
- Objects are named according to their prefab and not automatically serialized or networked (unless handled elsewhere).
- The UI must be manually hooked up in the Unity Inspector (i.e. assign the scroll menu, object prefab, etc.).

---

