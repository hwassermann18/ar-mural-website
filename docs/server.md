# **Server**
The AR Mural Server is a .NET-based backend that enables multiplayer AR drawing experiences on HoloLens. It listens for drawing commands (like add, delete, and modify) sent by clients via MQTT, stores drawing data persistently in a LevelDB database, and manages unique identifiers for all objects drawn in the shared environment.

It will be easier if we split our understanding of a server into 2 parts: the broker and the client. In this first section I will be discussing the files in the [server side repository](https://gitlab.orbit-lab.org/nfallah/ar-mural-server.git). These are the files that create the server and make changes to the database, which runs on a linux virtual machine at Winlab. The second section will discuss the clients of the server; how Unity is going to interact with the broker and enable multiplayer. 

---

## **LevelDB**
[LevelDB](https://github.com/google/leveldb) is a fast, lightweight, open-source **key–value store** developed by Google. It’s optimized for high-speed reads and writes using **log-structured merge trees**, making it ideal for embedded applications like mobile apps, desktop tools, and custom servers — like ours.

In AR Mural, LevelDB is used to **persistently store all drawing commands**[^1]. Each chunk of the AR environment is saved under a unique key (like `"3.0,5.0"` which identifies the chunk coordinate), and the value is a list of drawing commands (in JSON format) relevant to that chunk. The values have all the necessary information about each drawn object inside that chunk

---

### **leveldbwrapper.cpp**
Since LevelDB is originally written in C++ and the server is written in C#, we need a bridge connecting these two files. Thats why we have 2 LevelDB files: `leveldbwrapper.cpp` that exposes basic LevelDB functions in a simple C-style interface, and `leveldbwrapper.cs` that calls the C++ functions using DllImport, allowing you to use LevelDB like a C# class.

Here are some things we should be aware of in the beginning of the file:
```cpp title="C++" linenums="1"
#include <string>
#include <cstring>
#include "leveldb/db.h"

extern "C" {
leveldb::DB* db = nullptr;  
...
```
In order, these "include" statements tell the compiler to include the C++ string library, C-Style string handling functions (which will help with conversions to C++ strings), and most importantly includes the LevelDB header - which tells the compiler to include the db class so we can use its functionality. 

Line 5 is also critical; this line tells the compiler to compile them using C language's rules for naming and linkage, not C++’s. This is crucial when building a C++ library that is going to be used in another language (in this case C#). This is primarily because of *name mangling*, which you can dive into deeper on your own.

Lastly line 6 declares a global pointer to a LevelDB database object. This creates a pointer called db that will eventually point to the database object that acts as a lightweight engine where you can: read values by key (which helps you first pick a chunk), write or overwrite key–value pairs, and fetch values for a given key (a chunk coordinate).

The rest of the code then creates 6 functions.

---

#### **db_open**
```cpp title="C++" linenums="7"
bool db_open(const char *data_directory){
    std::string data_directory_str(data_directory);
    leveldb::Options options;
    options.create_if_missing = true;
    leveldb::Status status = leveldb::DB::Open(options, data_directory_str, &db);
    return status.ok();
}
```
This function deals with opening the folder where we store our data. The input function is a C-style string which is the path to the folder where LevelDB should store its data. It then converts this path to a C++ string and configures how the database should open and then opens the database file (or creates it if it doesn't exist) and stores the pointer in `db` (line 11). It then returns "true" if opening was successful.

---

#### **db_close**
```cpp title="C++" linenums="14"
bool db_close(){
    if (db != nullptr){
        delete db;
        db = nullptr;
        return true;
    }
    return false;
}
```
This function closes the database by deleting the `db` object. This doesn't just free up memory but also invoked shutdown logic to close the file. It then returns true if it successfully shut down the file.

---

#### **db_get**
``` cpp title="cpp" linenums="22"
const char* db_get(const char *key){
    std::string key_str(key);
    std::string value;
    leveldb::Status status = db->Get(leveldb::ReadOptions(), key_str, &value);
    if (!status.ok()){
        return nullptr;
    }
    char *result = new char[value.size() + 1];
    std::strcpy(result, value.c_str());
    return result;
}
```
This function looks up a string value in the LevelDB database using a given key (as a C string), which as you recall are the coordinates of a chunk, and returns the necessary information for all the drawing commands (objects) that were added to that specific chunk, encoded as a JSON array.

 If you aren't familiar with C++, you may think that this function returns a char, not a string. Given the parameters we need to return a raw data type, so in this case a char and not a string. We get around this by using a C-style string (char*), which is a pointer to a sequence of characters. Line 29 allocates the space for this string of characters in a pointer called `result`. Line 30 then copies the string from `value` into the position in memory that result was pointing to, so that now result is pointing to the JSON array containing all the necessary information of the objects in that chunk. Since we allocated space for this pointer, we are also going to have to free this space, which is the next method.

---

#### **db_free**
```cpp title="C++" linenums="33"
void db_free(const char *result) {
    delete[] result;
}

```
This method frees the memory allocated by db_get and is required because C# cannot free memory allocated in C++.

---

#### **db_put**
```cpp title="C++" linenums="36"
bool db_put(const char *key, const char *value) {
    std::string key_str(key);
    std::string value_str(value);
    leveldb::Status status = db->Put(leveldb::WriteOptions(), key_str, value_str);
    return status.ok();
}
```
This function is used to save or update the key-value pair in the database. It sends in the key (chunk coordinate) and the value (JSON array containing all object information in that chunk) and returns true if the database was successfully overwritten. 

---

#### **db_delete**
```cpp title="C++" linenums="42"
bool db_delete(const char *key){
    std::string key_str(key);
    leveldb::Status status = db->Delete(leveldb::WriteOptions(), key_str);
    return status.ok();      
}
```
This function deletes all the object information in a chunk. Given a key (chunk coordinate), it will delete the JSON array associated with it, essentially removing everything from the chunk.

---

#### **leveldbwrapper.cpp Overview**
Here is a quick table summary describing what all the functions in the C++ file do.

| Function    | Purpose                               |
| ----------- | ------------------------------------- |
| `db_open`   | Opens or creates the LevelDB database |
| `db_close`  | Closes and frees the database         |
| `db_get`    | Retrieves a value for a given key     |
| `db_put`    | Stores a key–value pair               |
| `db_delete` | Removes a key from the database       |
| `db_free`   | Frees memory returned by `db_get`     |

---

### **leveldbwrapper.so**
`leveldbwrapper.so` is a shared object file (compiled native library) that exposes a small set of C-compatible functions that wrap around Google’s LevelDB C++ API. It serves as the bridge between the C# server code and the C++ LevelDB backend. You may notice this file isn't in the repository and thats because it was created during the server setup (refer to the README file step 4).

We need this intermidiate file between the C++ and C# files since C# can't directly call C++ functions, but it can call **C-style functions** in a `.so` file. This is why there was so much string formatting in `leveldbwrapper.cpp` to aid in the compilation of `leveldbwrapper.so` file.

| Component            | What It Does                                                    |
| -------------------- | --------------------------------------------------------------- |
| `leveldbwrapper.cpp` | Implements C-style wrapper functions for LevelDB                |
| `leveldbwrapper.so`  | Compiled output of the above — the actual native library        |
| `LevelDBWrapper.cs`  | C# class that calls the `.so` file functions            |
| End Result           | C# can use LevelDB like a native C# object (`Put`, `Get`, etc.) |


---

### **leveldbwrapper.cs**

This is the file that is going to allow us to use the leveldb methods in C#. 
```csharp title="C#" linenums="1"
using System;
using System.Runtime.InteropServices;

public class LevelDBWrapper : IDisposable
...
```
Line 2 brings in a system library that has an attribute called `DLLImport`. This special C# attribute is used to call functions written in C or C++. Since our server runs on .NET, it can't directly use C++ classes, but it can called exported C functions via `DLLImport` which it does using `leveldbwrapper.so`.

Line 4 defines the C# class and implements `IDisposable` which will be used to close the database properly.

```csharp title="C#" linenums="6"
[DllImport("./leveldbwrapper.so", CallingConvention = CallingConvention.Cdecl)]
private static extern bool db_open(string data_directory);

[DllImport("./leveldbwrapper.so", CallingConvention = CallingConvention.Cdecl)]
private static extern bool db_close();

[DllImport("./leveldbwrapper.so", CallingConvention = CallingConvention.Cdecl)]
private static extern IntPtr db_get(string key);
...
```
The functions from the C++ file are then mapped to a private static method in this C# class, also ensuring that argument types are passed over properly. These directly bind the C++ functions with low level declarations; we will wrap these private methods with public C# methods. You may notice also `db_get` is returning an `IntPtr` type, whereas the C++ function returned a `char*` type; this is simply because IntPtr is the closest lowest level equivalent to char*. 

---

#### **Constructor** 
```csharp title="C#" linenums="24"
public LevelDBWrapper(string dataDirectory)
{
    if (!db_open(dataDirectory))
    {
        throw new Exception("FAIL");
    }
}
```
The constructor is called when you create a `LevelDBWrapper` object and called the `db_open()` method from the `.so` file, throwing an exception if it fails.

---

#### **Dispose**
The next 3 methods implement the **.NET** `IDisposable` **pattern**, which is used for cleaning up resources. The `Dispose()` method needs be called explicitely in the code that releases both managed and unmanaged memory by calling the overloaded Dispose method. The finalizer (**~LevelDBWrapper**) is called by the garbage collector. The garbage collector releases managed data but not unmanaged; the finalizer's task is to then free up any unmanaged data. Since we are only calling native code via **DLLImport**, we don't have any C# objects that need to be manually disposed. We will what this implies as we move on. If you would like to learn more about how resources are managed in .NET programming, visit this [site](https://learn.microsoft.com/en-us/dotnet/standard/garbage-collection/implementing-dispose).

1. **~LevelDBWrapper**
```csharp title="C#"
~LevelDBWrapper() 
{
    Dispose(false);
}
```
This method is known as the **finalizer (or destructor)** and is called automatically by the garbage collector to free up unmanaged resources. All this method does is call the overloaded Dispose method with false to indicate it **only** wants to free up **unmanaged resources**.

2. **Dispose()**
```csharp title="C#" 
public void Dispose() 
{
    Dispose(true);
    GC.SuppressFinalize(this);
}
```
This method needs to be explicitely called by the user to start memory cleanup. This method calls `Dispose(true)` which should free up managed and unmanaged resources. It then suppresses the call of the destructor because of redundancy.

3. **Dispose(bool disposing)**
```csharp title="C#" 
protected virtual void Dispose(bool disposing)
{
    if (disposing) // free unmanaged + managed resources, comes from Dispose(true)
    { // If you later add any IDisposable managed resources, dispose them here.
          
    }
    //free ONLY unmanaged resources
    db_close(); // Always close the unmanaged native database
}
```
This is the overloaded Dispose method that is called in the finalizer and parameterless Dispose. As you should recall, `db_close` releases unmanaged memory, so we will always call it in the overloaded Dispose method. There are no managed resources currently in this class to dispose of since it's calling methods from a C++ file that only has unmanaged resources.

---

#### **Get(string key)**
```csharp title="C#"
public string Get(string key)
{
    IntPtr valuePtr = db_get(key);
    if (valuePtr == IntPtr.Zero)
        return null;

    string value = Marshal.PtrToStringAnsi(valuePtr);
    db_free(valuePtr);
    return value;
}
```
This method wraps the C++ methods of `db_get` and `db_free` to get the string of objects within a chunk. It first sets an IntPtr to the position in memory holding the JSON array(type char*) of object data in that chunk. It then checks if the pointer is null, in which case it returns null. It then converts the unmanaged char* string to a C# string, frees the memory of the IntPtr, and returns the string of object data. There are also 2 additional methods: `Put(string key, string value)` and `Delete(string key)` that simply wrap their corresponding C++ method.

---

#### **Example Call Flow**
Here’s what happens when your server calls `database.Get("3,2")`:

1. C# calls `Get(string key)`

2. `Get()` calls `db_get()` via **DllImport**

3. C# jumps into the compiled .so binary

4. .so runs the compiled version of `db_get` from `leveldbwrapper.cpp`

5. The C++ code uses leveldb::DB::Get() to retrieve the value from disk

6. The value is returned to C# as a char* → converted to a C# string

7. Native memory is freed with `db_free`

---

## **Persistance Files**
Both the **client** and **server** repositories include a folder named `Persistence`, which contains a set of shared C# classes used to represent drawing commands and data in the AR Mural system. These files define the core data structures used for communication between client and server.

---

###  Purpose

The files in the `Persistence` folder are used to:

- Serialize and deserialize drawing-related data (`Add`, `Delete`, `Modify` commands)
- Store and retrieve data from the LevelDB database (on the server)
- Reconstruct GameObjects from received data (on the client)
- Ensure the client and server speak the **same language** when exchanging messages over MQTT (via JSON)

---

###  Key Files and Their Roles

| File | Description |
|------|-------------|
| `Command.cs` | Wraps a command type (`ADD`, `DELETE`, `MODIFY`) and its target data. Contains exactly one container per command. |
| `AddContainer.cs` | Full object data used to create a new object in the scene. Includes position, rotation, scale, and tool-specific container (e.g., Brush, Line). |
| `DeleteContainer.cs` | Minimal data (object ID and chunk) needed to remove an object. |
| `ModifyContainer.cs` | Contains updates to position, scale, rotation, and object movement across chunks. |
| `BrushContainer.cs`, `LineContainer.cs`, etc. | Tool-specific data for rendering  |
| `SerializeUtilities.cs` | Contains serializable versions of `Vector2`, `Vector3`, and `Color` to make Unity types compatible with JSON. |
| `IDContainer.cs` (client only) | Used to track object IDs and previous transform states during modification. |

---

###  Why These Files Exist in Both Client and Server

| Functionality | Client | Server |
|---------------|--------|--------|
| Constructs drawing commands | ✅ | ❌ |
| Converts Unity types into serializable data | ✅ | ❌ |
| Deserializes JSON from MQTT messages | ✅ | ✅ |
| Stores and retrieves objects in/from LevelDB | ❌ | ✅ |
| Reconstructs Unity GameObjects | ✅ | ❌ |

Having a copy of these files on both ends ensures:

- **Consistent serialization and deserialization**
- **Synchronized understanding of drawing commands**
- **Interoperability between systems using MQTT and JSON**

These files must stay structurally identical (e.g., same field names and types) across both repos to avoid serialization errors.

---

###  Differences Between Client and Server Versions

- **Client versions** include extra **constructors and helper methods** to build containers from Unity `Transform`s or tool settings (e.g., position, color, scale).
- **Server versions** are minimal and used mainly for deserialization and database storage.
- Some fields in the client may also include extra metadata for real-time manipulation that the server doesn't need.

---

###  Summary

The `Persistence` files are the **backbone of data communication** in the AR Mural system. They enable seamless serialization of drawing actions so that:

- Clients can draw, move, and delete
- The server can store and broadcast
- Other clients can recreate and view the shared experience

Keeping these classes aligned across both repos is essential for multiplayer AR functionality.




[^1]:"Persistance" is a term commonly used with servers. It means that even if the server shuts down unexpectedly or the client disconnects, the data remains available the next time the server starts.