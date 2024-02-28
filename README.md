# Modular ship attatchment system in unity

In our game, you can build spaceships like you're playing with building blocks in real-time. This idea was inspired by 'Captain Forever,' but we took it a step further by making it in 3D. Each part of the ship, which we call modules, was designed to easily connect with others, forming different shapes and functions. These modules can attach or detach from each other during gameplay. To build a ship you simply click to select a part, drag it over one of the orbs that show up, and then release.

![image](https://github.com/Jacob19999/unity_spacecship_attachment_system/assets/26366586/1f8ff830-37e5-4d70-8e24-1032b0050f4c)

Some problems that taking this idea to 3D created were figuring out where to 'float' a module when you're holding it and designing a control system that lets you manage flying your ship and building it at the same time.

![image](https://github.com/Jacob19999/unity_spacecship_attachment_system/assets/26366586/3ef9026b-7772-4960-9eaa-13e7a28b0a48)

# Defining Modules

To create the basic building blocks of the ship - the modules - I created an interface structure that made use of inheritance. The root interface for this is called IModule, which serves as the foundation for all module types in 'Endangered Orbits'. This interface ensures that every module within the game adheres to a uniform set of properties and behaviors, creating a consistent and extensible framework for module development. Within this interface, several key elements are outlined.

In the subsequent description of the interfaces, I outline the intended functions as dictated by the interfaces. However, since an interface does not inherently express these implementation intentions, we adopted a traditional class hierarchy to actualize the interfaces. Our game modules then inherited from these classes. This was done primarily to prevent the need to duplicate code across several classes.You could argue that this makes the interfaces redundant.

public interface IModule
{
    public GameObject GetGameObject { get; }
    public string Name { get; set; }
    public int Health { get; set; } 
    public int MaxHealth { get; } 
    public int Mass { get; get; }
    public DamageResistancePercent DamageResistance { get; set; }
    public List<IAttachable> AttachedModules { get; set; }
}

Most of these properties, like “health” are straightforward in their purpose and so I will skip describing them. The two properties that require a more in depth description are AttachedModules and DamageResistance . 

AttachedModules is simply a list of modules that are attached to this module. As will be shown later - each module may have many attachment points. This list contains a reference to any modules that may occupy and such points. This implicitly creates a tree structure where each ‘node’ may have a number of child ‘nodes’ equal to the number of attachment points the module has. This list is primarily a convenience. All child modules are set as children to the parent module via the game object. You can use transform.GetChild(int index) to access the children of any module - but you would have to loop through all children and filter out unrelated game objects. In all cases the parent is accessed via transform.parent. 
DamageResistance is an instance of the class DamageResistancePercent. This class simply exists to store a float but limit it to between 0 and 1 - thus representing a percent.

public interface IAttachable : IModule
{
    public bool IsAttached { get; } 
    public void Attach(AttachmentPoint attachmentPoint);
    public void Eject();
    public void Disconnect(bool removeFromList = true);
    public void PickUp();
    public void Release(Vector3 releaseVelocity);
    public AttachmentPoint FindThisConnectionPoint();
}

IAttachable is the other interface that needs to exist to structure the module system. As you can see it inherits IModule. This interface simply enforces the inclusion of functionality that allows a module to attach to another module. The only state required here is a flag to denote whether the module is attached to something or not. 

Here is a quick description of the purpose of the functions:

1.	Attach
   
This function attempted to join two modules together in this way:
1.1.	Align the current module's game object with the game object of the target attachment point.
1.2.	Set the current module's game object as a child of the parent module's game object. The parent module's game object is retrieved from the target attachment point.
1.3.	Change the current module's game object rigidbody to kinematic.
1.4.	Set the attachment flag to true.

3.	Eject
   
This function launches the module away from what it is connected to. Here is how:
2.1.	Unset the current module's game object as a child of the parent module's game object.
2.2.	Change the current module's game object rigidbody to dynamic.
2.3.	Set the attachment flag to false.
2.4.	Apply a force to the current module's game object to push it away from the parent module.
2.5.	Get the player ship's current vector and slowly apply it to the current module's game object to halt its movement relative to the player ship.
The reason we slow it down in 2.5 is so that modules don't fly off into space and thus prevent the player from acquiring the module for their own ship. This function is never invoked directly by the player’s actions but rather is triggered when the module can not validly be connected to its parent anymore. This can happen, for one example, if the parent module is destroyed in combat.

4.	Disconnect
   
This function simply severs the connection between two modules. Here is how:
3.1.	Unset the current module's game object as a child of the parent module's game object.
3.2.	Disable the rigidbody interactions of the current module's game object.
3.3.	Set the attachment flag to false.
The rigidbody interactions are disabled to ensure the module, post-disconnection, does not collide with other objects while being constrained to the player's mouse cursor. This ensures that the player cannot use the module to interact physically with the environment or other objects.

5.	PickUp
   
This function simply makes the necessary changes to the module, like disabling the rigidbody, to prepare a module to be constrained to the player’s mouse

7.	Release
   
Reverts the module to its original state after being picked up by the player and applies a velocity to the module in proportion to the player’s mouse movement, but with a velocity maximum.

9.	FindThisConnectionPoint
    
Returns the first found attachment point on the current module with ConnectionPoint set to true. See the section on AttachmentPoint to understand this.

When two modules are joined together they are kept together because the child game object is a child of the parent game object. Setting the rigidbody to kinematic keeps the child in the position it is supposed to be at.

# Attachment Points

Attachment Points are basically game objects that are positioned at the point on a module where you want it to be able to attach to other modules. These are controlled via the AttachmentPoint class. The control this class manages is two things:

1.	Based on the position and rotation of the game object - how to attach other modules to the given point.
   
3.	Enabling and disabling the mesh render at appropriate times. This also includes systems that allow the material(s) used to be changed in runtime.

In order for modules to be attached to each other, the AttachmentPoint class enables 
precise alignment based on the position and rotation of the game object that the AttachmentPoint instance is attached to. When the Attach() function is called, it uses an AttachmentPoint instance to determine how another module should be aligned and connected. The process involves aligning the forward 	 
An attachment point in the editor. Note the forward vector.
vectors of the connecting points using quaternion rotations, ensuring that the module is oriented correctly with respect to the target attachment point.The function also adjusts the Z-axis rotation to match that of the attachment point, maintaining consistency in the orientation. Additionally, it calculates the necessary translation to position the module exactly at the attachment point. Once positioned and rotated correctly, the module's GameObject is set as a child of the attachment point's parent GameObject, ensuring it moves and behaves as a unified object.
Finally, the attachment points on both the attaching module and the target module are marked as occupied, and their visual meshes are hidden, indicating that a successful connection has been made.
In order to determine the AttachmentPoint that should be used to make the connection on the module that the player is holding, each one has a bool named connectionPoint. When the attachment process is invoked on the module the player is holding a linear search is performed on the module’s children. When the first AttachmentPoint with this value set to true is found - it is that point which is joined as described above.

# Control

The first problem that needed to be solved was where to float a module while the player was holding it. The most user-friendly solution is to hold the object the same distance away from the camera that the core of the ship is. To achieve this a script is attached to the main camera. In this a plane called the InteractionPlane is created. We use the camera's forward vector to align the plane in the correct position and then place the center of the plane at the core of the ship by getting the position property of its transform. When the loop that updates the position of a held module runs it finds the point on the plane it should reposition the module to by using ScreenPointToRay on the main camera. This finds the distance of the plane from the camera and allows the accurate positioning of the module.

When the player is holding a module it enables an if clause in the Update function on the interaction controller (where all the input control is located) This where the function that updates the position of the held module is called. This also calls a function that checks for the player hovering over an attachment point. By placing the attachment points in their own layer we can use a layer mask to get only the attachment points from a ray cast. When the player is hovering over an attachment point the module is positioned over the prospective attachment area - but with an offset so it looks like it is hovering over the point. The occlusion problem is sufficiently dealt with by simply selecting the 1st attachment point that the ray cast hits.

Finally the problem of dealing with flying the ship while building it in a full newtonian simulation is really only solvable through player skill. It also would likely require additional controllers, such as a joystick - because otherwise the mouse becomes overloaded with responsibilities. We settled on having three modes. First there is a build mode which allows the addition and removal of modules from the ship. There is a camera mode - which is the same as the build mode except you can't modify your ship. Finally there is a combat mode. In this mode the ship attempts to orient in the direction you are looking and the controls for lateral movement control are activated.

