# UITest Unity Server

This is the embedded server that runs in a Unity Project to facilitate automated testing with Xamarin.UITest.  
The companion test project is located here: https://bitbucket.org/agentsofdiscovery/uitestclient/overview

## Setup
- Target .NET 4.X, .NET Standard 2.0
- Grab the `UITestUnity-X.X.unitypackage` and import it in your project
- Place `StartTestServer.cs` on a scene that will never be destroyed. This will start a UITest Server when creating a debug build of your application, listening on your devices `IP:8081`
- Add JSON.NET to your project. There's a Unity specific version that supports IL2CPP, https://github.com/SaladLab/Json.Net.Unity3D/releases  
*or* you can easily use a different serialization library by modifying `RouteT.cs`

## Routes
You can interact with the UITest Server by just running in-editor, by making requests to the supported endpoints with tools such as cURL or Postman.

### Endpoints
`GET CurrentScreen`  
Gets the current visible scene

`GET DeviceInfo`  
Gets the device Height, Width, and DPI

`GET GameObjectFind { type: "GameObject", name: "objName" }`  
Returns object information. Default type is `GameObject`. Supported `UnityEngine.UI` types include:  
`Text`, `Button`, `InputField`, `Image`  

`POST InvokeButton { name: "btnName" }`  
Clicks the specified button, `name` is the name of the element in the scene

`POST InvokeInput { name: "inputName", text: "inputText" }`  
Enters the specified text into the specified `InputField/TMP_InputField`

Their definitions reside in `UITestUnity/Routes`. `DeviceInfoRoute.cs` is a good example of defining a route.
`UnityRoutes.cs` enables the routes themselves, if you add a new route, ensure to enable it there.

### More about routes
There are 2 objects each route definition inherits from, `Route` and `Route<T>`. The basic `Route` is for routes that just return an unserialized `string`, an example implementation is `CurrentScreenRoute.cs`. It's instead recommended to use `Route<T>`, as you can return a basic POCO object as your response, which will be serialized before being sent to the client.

Currently supported objects are returned as `GameText`, `GameButton`, `GameInputField`, `GameImage`. All these types derive from `GameElement`, which contains some basic element information such as:  
```csharp
string Id { get; set; }
string Name { get; set; }
string Type { get; set; }
string Parent { get; set; }
string[] Children { get; set; }
PointF Location { get; set; }
RectangleF Rectangle { get; set; }
bool? IsOnScreen { get; set; }
```

Routes definitions contain the following:  
`string RoutePath`, the name of the endpoint  
`void Enable()`, instantiates the route and registers it in the router.  
`bool SupportsMethod(HttpMethod method)`, defines what HttpMethods the route supports.  
Inheriting from `Route`:  
`string GetResponseString(HttpListenerRequest request)`  
Inheriting from `Route<T>`:  
`T GetResponse(HttpMethod method, NameValueCollection queryString, string data)`  
Defines the response from a request

### Adding new Routes
- Create a new object in `Routes`, put it in `namespace Xamarin.GameTestServer.Unity`
- Define your response object, IE:
```csharp
class MyObject
{
    int ScreenHeight { get; set; }
    // ...
}
```
- Inherit from `Route<T>`, ie:
```csharp
namespace Xamarin.GameTestServer.Unity
{
    class MyRoute : Route<MyObject>
    {
        // ...
    }
}
```
- Write an `Enable()` method that registers an instance of this route in the router:  
```csharp
static void Enable()
{
    Router.AddRoute("MyRouteEndpoint", new MyRoute());
}
```
- Add override for `SupportsMethod` and define what HTTP methods it supports (GET/POST):
```csharp
override bool SupportsMethod(HttpMethod method)
{
    // GET & POST: method == HttpMethod.Get || method == HttpMethod.Post;
    return method == HttpMethod.Get;
}
```
- Override `T GetResponse`. It will return the same type we specified. 
Since the router is running in a separate thread, you will need to perform Unity Engine calls on the Main thread.
```csharp
override MyObject GetResponse(HttpMethod method, NameValueCollection queryString, string data)
{
    var device = new MyObject();
    // Call UI methods on UI Thread
    RunInUIThread(() =>
    {
        device.ScreenHeight = Screen.height;
        // ...
    });
    
    return device;
}
```
- Enable the route in `UnityRoutes.cs`
```csharp
public static void Enable ()
{
    DeviceInfoRoute.Enable();
    CurrentScreenRoute.Enable();
    GameObjectFind.Enable();
    InvokeButtonRoute.Enable();
    InvokeInputRoute.Enable();
    MyRoute.Enable(); // Our new route
}
```

### Adding more supported GameObjects
`GameObjectFind` is the most commonly used endpoint by `UITest`.

To add more supported game elements, add their serializable definitions in `UITestUnity/GameObjects`, look at `GameImage.cs` as an example. Then, open `GameElementExtensions.cs` and add it to the body of extension method: `ToGameElement(this UnityEngine.Object obj, Type type)`. This converts a standard Unity GameObject into one of our serializable GameObjects.
```csharp
// Example definition of a 
namespace Xamarin.GameTestServer
{
    public class GameButton : GameElement
    {
        public string Text { get; set; }
    }
}

/// obj refers to the UnityObject we're on, `Type type` is what we're converting to
public static GameElement ToGameElement(this UnityEngine.Object obj, Type type)
{
    ...
    // We're mapping UnityEngine.UI.Button to GameButton
    else if (type == typeof(Button) || type == typeof(GameButton))
    {
        // Cast obj to our Unity object, this should always succeed since obj
        /// should be a Button to begin with
        var btn = obj as Button;
        if (btn != null)
        {
            // Get our Button's text
            var txtComponent = btn.GetComponentInChildren<Text>();
            string txt = (txtComponent == null) ? "" : txtComponent.text;

            // InitFrom<T> Maps a bunch of meta info such as Id and Name automatically
            var gb = GameButton.InitFrom<GameButton>(go);
            gb.Text = txt; // Set our text from earlier
            gb.Type = btn.GetType().ToString(); // Set our type (UnityEngine.UI.Button)

            return gb;
        }
    }
    ...
}
```

### Running the sample project
First, add Json.NET: https://github.com/SaladLab/Json.Net.Unity3D/releases/download/v9.0.1/JsonNet.9.0.1.unitypackage

Run the project, it should start the GameServer, and you should be able to call:
```bash
curl -X GET 'http://localhost:8081/GameObjectFind?type=Button'
```
And receive:
```json
[
    {
        "Text": "Button",
        "Name": "Button",
        "Id": "-20146",
        "Type": "UnityEngine.GameObject",
        "Parent": "Canvas",
        "Children": [
            "Text"
        ],
        "Location": {
            "IsEmpty": false,
            "X": 178,
            "Y": 316.5
        },
        "Rectangle": {
            "Bottom": 346.5,
            "Height": 30,
            "IsEmpty": false,
            "Left": 178,
            "Location": {
                "IsEmpty": false,
                "X": 178,
                "Y": 316.5
            },
            "Right": 338,
            "Size": "160, 30",
            "Top": 316.5,
            "Width": 160,
            "X": 178,
            "Y": 316.5
        },
        "IsOnScreen": true
    }
]
```
This response will be parsed by our companion wrapper for Xamarin.UITest Unit Tests.