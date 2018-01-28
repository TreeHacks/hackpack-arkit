# hackpack-arkit

[Medium]

### Overview
In this hackpack,  we're going to work with ARKit, the new framework from Apple that allows us to easily create augmented reality experiences in our iOS apps. ARKit is most commonly used in one of three ways: inside of Xcode writing Swift or Objective-C Code, inside of the Unity game engine, and inside of the Unreal game engine. In this tutorial, we’ll be working with with Swift 4 and Xcode 9. If you are familiar with Unity or Unreal, I recommend following Unity or Epic’s ARKit documentation rather than this hackpack.

This hackpack will teach you how to use a custom 3D model in agumented reality, how to use the ARKit hitscan, and how to anchor features to place the custom model on real-world objects. We will create an AR experience where users can “plant” trees in AR and walk around them.

### Getting Started
First, create a new Xcode project and select the Augmented Reality template.

Press next, fill in the following fields, ensuring you select SceneKit for the final field.

### Finding a Model to Work With

[TurboSquid](https://www.turbosquid.com/Search/3D-Models/free) is a website where you can find 3D models to play around with; you can find free (or cheap) 3D models. One thing to be aware of though is that some models here are meant for non-realtime rendering, so the poly counts may be high. Anything over 10k is probably too high for a mobile app. SceneKit supports DAE models. As a shortcut, you can use this [direct link](https://www.turbosquid.com/Search/3D-Models?keyword=&search_type=free&media_typeid=2&file_type=194&min_poly=0k&max_poly=10k) to find DAE models that are free and less than 10k polys. Every model on this list should be appropriate for use in your app.

For this hackpack, we are going to use this free [tree model](https://www.turbosquid.com/3d-models/sample-trees-c4d-free/1008420).

### Import the DAE Model into the Xcode Project

Download the DAE version of the model and drag the file in to the Xcode navigator in to your art.scnassets folder that was created from the AR app template. Once you have added the asset to your Xcode project, you can preview it within Xcode by holding click and drag around to rotate around your mesh. If you don’t see anything in this step, there is an issue with your model.

### Load the Model into Your Scene

Now that the model is in your project you can open up the ViewController.swift file created by the template and swap out the default spaceship for your model. Find this line of code:

```Swift
let scene = SCNScene(named: "art.scnassets/ship.scn")!
```
Change it to point to your downloaded model, which is Lowpoly_tree_sample.dae.

```Swift
let scene = SCNScene(named: "art.scnassets/Lowpoly_tree_sample.dae")!
```

When running the app with this change you will not be able to see the model because it is too large and not positioned in front of the camera. To fix this, select the tree model in the SceneKit editor view by clicking it, and then on the right-hand side pane select the Node inspector (the cube icon). Set the x, y, and z scale to 0.01, reducing the size of the tree by 99%.

You may notice that in the default position, the tree is not visible. This is because it’s origin is at position 0, 0, 0, and it’s invisible because the camera is actually inside of the tree at this location. If you start up the AR app on a device and walk around a bit, when you turn around you’ll see the tree. We need to change the default position to be 1 unit in front of the camera. In order to do this, we’ll need to find the tree as a node within the scene.

Going back to the SceneKit editor (select the DAE file within Xcode), we can click on the tree model itself, and then again in the Node inspector there should be a name. This is the name of the selected mesh, in this case, it's Tree_lp_11. If you’re using a different model, the name may be different, or empty.

Now that we know the name of the node, we can access it within our code.

```Swift
let scene = SCNScene(named: "art.scnassets/Lowpoly_tree_sample.dae")!
let treeNode = scene.rootNode.childNode(withName: "Tree_lp_11", recursively: true)
```

The second line above does a search of the child nodes of the scene object created from the DAE file we imported, and returns a node with the name specified. Since 3D models can have deeply nested nodes, it’s often useful to recursively search through the nested heirarchy to find the mesh object, so we opt to search recursively, even though it is not neccessary for this particular model.

From here, we can simply reposition the node by moving it forward a bit. The means going in a negative direction in the z axis, as the default camera faces down the negative Z. Or in other words it’s looking at the point (0, 0, -Inf).

So to make the tree visible, let’s move it back 1 unit in the z direction. We're going to create a new position object from scratch and assign that as the new position.

```Swift
let scene = SCNScene(named: "art.scnassets/Lowpoly_tree_sample.dae")!
let treeNode = scene.rootNode.childNode(withName: "Tree_lp_11", recursively: true)
treeNode?.position = SCNVector3Make(0, 0, -1)
```

In order to modify the treeNode later, let’s keep an instance reference to it. Outside of any functions, but inside the ViewController class, add an optional reference to treeNode:

```Swift
class ViewController: UIViewController, ARSCNViewDelegate {
var treeNode: SCNNode?
...
}
```

In the next step we’re going to want a reference to the treeNode again, so rather than doing the lookup every time, it is useful to cache the reference to the treeNode. To do this, we'll modify the childNode call we just added inside of viewDidLoad so that it sets this to an instance variable treeNode as opposed to just the local treeNode variable:

```Swift
let scene = SCNScene(named: "art.scnassets/Lowpoly_tree_sample.dae")!
self.treeNode = scene.rootNode.childNode(withName: "Tree_lp_11", recursively: true)
self.treeNode?.position = SCNVector3Make(0, 0, -1)
```

### Using HitTest

Next, let’s modify the app so that when the user taps, it will place a new copy of the tree model to wherever they tapped. This is a very complex operation because tapping on the 2D screen needs to go through a projection matrix based on the estimated shape of the 3D scene it’s seeing through the camera. ARKit makes this complicated process fairly simple by providing the HitTest function.

First, we implement an override for touchesBegan, which is called any time the user taps on the screen. From this we can retrieve the first touch and perform a hitTest in the ARScene. We’ll look for results that are anchors. Anchors are points in space ARKit has identified and is tracking. In other words, it’s a surface of a real-world object.

Once we get a result from the hitTest, we can position the treeNode to the same location as the anchor. The hit test comes back with a 4×4 matrix containing the scale, rotation, and position data. The 4th row of this matrix is the position. We can reconstruct the position using this row by accessing m41, m42, and m43 as x, y, and z respectively. Setting the position to a new SCNVector3 object with these coordinates should move the tree to that location.

```Swift
override func touchesBegan(_ touches: Set<UITouch>, with event: UIEvent?) {
guard let touch = touches.first else { return }
let results = sceneView.hitTest(touch.location(in: sceneView), types: [ARHitTestResult.ResultType.featurePoint])
guard let hitFeature = results.last else { return }
let hitTransform = SCNMatrix4(hitFeature.worldTransform)
let hitPosition = SCNVector3Make(hitTransform.m41,
hitTransform.m42,
hitTransform.m43)
treeNode?.position = hitPosition;
}
```

Give this a try and you’ll find you can teleport the tree to locations in the real-world when you tap!

Now, we can make clones of the tree as well so we can plant our forest!

```Swift
override func touchesBegan(_ touches: Set<UITouch>, with event: UIEvent?) {
guard let touch = touches.first else { return }
let results = sceneView.hitTest(touch.location(in: sceneView), types: [ARHitTestResult.ResultType.featurePoint])
guard let hitFeature = results.last else { return }
let hitTransform = SCNMatrix4(hitFeature.worldTransform)
let hitPosition = SCNVector3Make(hitTransform.m41,
hitTransform.m42,
hitTransform.m43)
let treeClone = treeNode!.clone()
treeClone.position = hitPosition
sceneView.scene.rootNode.addChildNode(treeClone)
}









